# Error Types

There are few options for declaring errors.
Consider the following before picking the option best suited for your use case.

- Does the caller need to match the error so that they can handle it?
  If yes, we must support the [`errors.Is`] or [`errors.As`] functions
  by declaring a top-level error variable or a custom type.
  At the call site, on Go 1.26 and later, use [`errors.AsType`]
  instead of a local `var` plus `errors.As`.
  `go fix` rewrites that pattern.
- Is the error message a static string,
  or is it a dynamic string that requires contextual information?
  For the former, we can use [`errors.New`], but for the latter we must
  use [`fmt.Errorf`] or a custom error type.
- Are we propagating a new error returned by a downstream function?
  If so, see the [section on error wrapping](error-wrap.md).

[`errors.Is`]: https://pkg.go.dev/errors#Is
[`errors.As`]: https://pkg.go.dev/errors#As
[`errors.AsType`]: https://pkg.go.dev/errors#AsType

| Error matching? | Error Message | Guidance                            |
|-----------------|---------------|-------------------------------------|
| No              | static        | [`errors.New`]                      |
| No              | dynamic       | [`fmt.Errorf`]                      |
| Yes             | static        | top-level `var` with [`errors.New`] |
| Yes             | dynamic       | custom `error` type                 |

[`errors.New`]: https://pkg.go.dev/errors#New
[`fmt.Errorf`]: https://pkg.go.dev/fmt#Errorf

For example,
use [`errors.New`] for an error with a static string.
Export this error as a variable to support matching it with `errors.Is`
if the caller needs to match and handle this error.

<table>
<thead><tr><th>No error matching</th><th>Error matching</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

if err := foo.Open(); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
  return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
  if errors.Is(err, foo.ErrCouldNotOpen) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

For an error with a dynamic string,
use [`fmt.Errorf`] if the caller does not need to match it,
and a custom `error` if the caller does need to match it.

<table>
<thead><tr><th>No error matching</th><th>Error matching</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

// package bar

if err := foo.Open("testfile.txt"); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

</td><td>

```go
// package foo

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

func Open(file string) error {
  return &NotFoundError{File: file}
}


// package bar

if err := foo.Open("testfile.txt"); err != nil {
  if notFound, ok := errors.AsType[*NotFoundError](err); ok {
    // handle notFound
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

Do not match that error with a separate variable and `errors.As`.
`go fix` rewrites this with the `errorsastype` modernizer:

```go
var notFound *NotFoundError
if errors.As(err, &notFound) {
  fmt.Println(notFound.File)
}
```

`go fix -json` reports
`errors.As can be simplified using AsType[*NotFoundError]`.

[`errors.AsType`] was added in Go 1.26
([Go 1.26 release notes](https://go.dev/doc/go1.26#errorspkgerrors)).
That paragraph does not cite an issue number.
`errors.As` still matches the same custom type.
The modernizer only rewrites the local `var` plus `errors.As` pattern
([`errorsastype`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/modernize#errorsastype)).
`errors.Is` and `errors.As` are the Go 1.13 matching API
([Go 1.13 release notes](https://go.dev/doc/go1.13#error_wrapping),
[issue #29934](https://go.dev/issue/29934)).

Note that if you export error variables or types from a package,
they will become part of the public API of the package.
