# Use sync/atomic types

Use the typed values in [`sync/atomic`]
(`Bool`, `Int32`, `Int64`, `Uint32`, `Uint64`, `Uintptr`, and `Pointer`)
instead of a raw integer and a function such as `atomic.SwapInt32`.

Go 1.19 added those types so the value stays hidden
and every access goes through the atomic API
([Go 1.19 release notes](https://go.dev/doc/go1.19#atomic_types),
[issue #50860](https://go.dev/issue/50860)).

`go fix` runs the `atomictypes` modernizer
([Go 1.27 release notes](https://go.dev/doc/go1.27#go-fix);
that list of new modernizers does not cite an issue number).
`atomictypes` rewrites a variable when every use is a primitive
`sync/atomic` call.
An `int32` becomes `atomic.Int32`.
It leaves the variable alone when any use is a plain read or write,
so a race is not rewritten into a typed load
([`atomictypes`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/modernize#atomictypes)).

A boolean flag is `atomic.Bool`, not an `int32`.
`go.uber.org/atomic` is not required for these types.
Its [README] still describes `sync/atomic` as raw integers
and does not mention the Go 1.19 types.
No commit on this file chooses that package over the standard library.
The last edit is
[commit 37c45b52bf9adf36ccff5db54187cdf4d45e0cc9]
([#265]), a pointer typo.
That pull request has no review comments.

Follow the `atomic.Bool` example for a flag
even when `atomictypes` is silent on the racy read.

  [`sync/atomic`]: https://pkg.go.dev/sync/atomic
  [README]: https://github.com/uber-go/atomic/blob/master/README.md
  [commit 37c45b52bf9adf36ccff5db54187cdf4d45e0cc9]: https://github.com/uber-go/guide/commit/37c45b52bf9adf36ccff5db54187cdf4d45e0cc9
  [#265]: https://github.com/uber-go/guide/pull/265

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import "sync/atomic"

type foo struct {
  running int32 // atomic
}

func (f *foo) start() {
  if atomic.SwapInt32(&f.running, 1) == 1 {
    // already running…
    return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running == 1 // race!
}
```

</td><td>

```go
import "sync/atomic"

type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
    // already running…
    return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running.Load()
}
```

</td></tr>
</tbody></table>

When every access is already an atomic function call,
`atomictypes` does rewrite the primitive form.
`go fix -json` on the bad column reports
`var n int32 may be simplified using atomic.Int32`.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import "sync/atomic"

type foo struct {
  n int32
}

func (f *foo) inc() {
  atomic.AddInt32(&f.n, 1)
}
```

</td><td>

```go
import "sync/atomic"

type foo struct {
  n atomic.Int32
}

func (f *foo) inc() {
  f.n.Add(1)
}
```

</td></tr>
</tbody></table>
