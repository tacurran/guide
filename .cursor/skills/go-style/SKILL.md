---
name: go-style
description: >-
  Apply the Uber Go style guide when writing, reviewing, or refactoring Go.
  Use this skill for Go files, for reviews of Go changes, and whenever the
  change should follow this guide (style.md, generated from src/*.md).
---

# Uber Go style

Follow this procedure while writing, reviewing, or refactoring Go. The guide is `style.md`, generated from `src/`. Read the `src/*.md` file named below when a section is long; do not paste it into the change. Do not enforce a style rule this guide does not state.

`gofmt` owns formatting. Do not hand-format.

When a rule names a checker, run that checker and keep its rewrite. Do not restate the checker in a comment or a second cleanup.

Where this guide calls a rule house judgment, follow the example even when `go vet`, `go fix`, and staticcheck are silent. Silence is not a waiver. Do not turn an unenforced rule into a broader ban.

`golint` is deprecated and frozen. Do not run it and do not treat its findings as style. `revive` replaced it. Code is error-free under `revive` and `go vet`.

## Guidelines

### Pointers to Interfaces

Pass interfaces as values. Almost never use a pointer to an interface. The concrete value stored in the interface may still be a pointer. If the methods must modify the underlying data, that concrete value is a pointer.

### Verify Interface Compliance

Add a compile-time check where a mismatch would break users:

- an exported type required to implement an interface as part of its API
- an exported or unexported type in a collection of types that implement the same interface
- any other case where violating the interface would break users

Assign the zero value of the asserted type: `nil` for a pointer, slice, or map; an empty struct for a struct. `var _ http.Handler = (*Handler)(nil)` fails to compile if `*Handler` stops implementing `http.Handler`.

### Receivers and Interfaces

A value-receiver method may be called on a value or a pointer. A pointer-receiver method may be called only on a pointer or an addressable value. A value stored in a map is not addressable, so a pointer method cannot be called on it. The method set of a value does not include pointer methods, so a value does not satisfy an interface that requires one. A pointer satisfies the interface even when the method has a value receiver. Read `src/interface-receiver.md`.

### Zero-value Mutexes are Valid

The zero value of `sync.Mutex` and `sync.RWMutex` is valid. Almost never use a pointer to a mutex: prefer `var mu sync.Mutex` over `new(sync.Mutex)`. On a struct that is used by pointer, store the mutex as a non-pointer field. Do not embed the mutex, even when the struct is unexported. On an exported struct, embedding makes `Lock` and `Unlock` part of the API.

### Copy Slices and Maps at Boundaries

A slice or map shares its backing storage. Copy a slice or map before storing one you received. Return a copy when a slice or map would expose internal state. Read `src/container-copy.md`.

### Defer to Clean Up

Use `defer` to release resources such as files and locks. Skip `defer` only when you can prove the function runs in nanoseconds. The readability of `defer` outweighs its cost on anything larger.

### Channel Size is One or None

Usually make a channel unbuffered (size 0) or buffered with size 1. This is house judgment: `go vet`, `go fix`, and staticcheck do not enforce it. Follow it anyway. Any other size needs scrutiny: how the size was chosen, what stops the channel from filling and blocking writers, and what happens when it does.

### Start Enums at One

Usually start an `iota` enum at a non-zero value (`iota + 1`), because the zero value is the default. This is house judgment. Exception: start at zero when the zero value is the intended default.

### Use `"time"` to handle time

Use `time` for instants and durations. Use `time.Time` for an instant and its methods to compare, add, or subtract. Use `time.Duration` for a period; do not pass a bare `int` and multiply it back into a duration at the call. Use `Time.AddDate` to move by a calendar day. Use `Time.Add` when the instant must be exactly 24 hours later.

At an external boundary, prefer `time.Time` and `time.Duration` (`flag`, JSON `time.Time` as RFC 3339, SQL `DATETIME`/`TIMESTAMP` when the driver supports it, `gopkg.in/yaml.v2`). When a duration cannot be a `time.Duration`, use `int` or `float64` and put the unit in the field name. When an instant cannot be a `time.Time`, use a `string` formatted as RFC 3339 unless a different format was agreed. `time` does not parse leap seconds and does not include them in differences. Read `src/time.md`.

### Errors

#### Error Types

Pick the declaration from the caller's needs. Read the table in `src/error-type.md`.

- No matching, static message: `errors.New`.
- No matching, dynamic message: `fmt.Errorf`.
- Matching, static message: a top-level variable from `errors.New`, matched with `errors.Is`.
- Matching, dynamic message: a custom `error` type.

On Go 1.26 and later, match a custom type at the call site with `errors.AsType`. Do not declare a local variable and pass it to `errors.As`. `go fix` runs `errorsastype`, which rewrites that local-variable form. Run `go fix`. `errors.As` still matches the same type; the modernizer only rewrites the local `var` plus `errors.As` pattern. `errors.Is` and `errors.As` stay the matching API.

Exporting an error variable or type makes it part of the package API.

#### Error Wrapping

Propagate a failure in one of three ways: return the error unchanged, wrap it with `fmt.Errorf` and `%w`, or wrap it with `fmt.Errorf` and `%v`.

Return the original error when you have no context to add. Otherwise add short context (`"new store: %w"`). Do not add a phrase that only states the obvious, such as "failed to". Use `%w` when the caller should match the cause, and document and test a known `var` or type that is part of the contract. Use `%v` when the caller must not match the cause. Once the error is sent to another system, make it obvious that it is an error (an `err` tag, or a "Failed" prefix in logs). Read `src/error-wrap.md`.

#### Error Naming

Name an exported error variable `Err…` and an unexported one `err…`. This prefix replaces the `_` prefix on unexported globals. Name a custom error type with the suffix `Error`. staticcheck `ST1012` wants an API error variable named `errFoo` or `ErrFoo`. Run staticcheck. The `Error` suffix on types is this guide's rule where that check is silent.

#### Handle Errors Once

Handle each error once. Do not log an error and also return it. Match with `errors.Is` or `errors.As` and branch, log and degrade, return a defined domain error, or return the error wrapped or verbatim. Read `src/error-once.md`.

### Handle Type Assertion Failures

Use the comma-ok form of a type assertion. The single-return form panics on the wrong type.

### Don't Panic

Production code returns an error and lets the caller decide. Do not use panic as the error strategy. Panic only for something irrecoverable, such as a nil dereference. Exception: a failure during program initialization that must abort may panic (`template.Must`). In tests, use `t.Fatal` or `t.FailNow`, not panic.

### Use sync/atomic types

Use the typed values in `sync/atomic` (`Bool`, `Int32`, `Int64`, `Uint32`, `Uint64`, `Uintptr`, `Pointer`). Do not pair a raw integer with `atomic.SwapInt32` or a similar function. A flag is `atomic.Bool`, not an `int32`. `go.uber.org/atomic` is not required for these types.

`go fix` runs `atomictypes`. It rewrites a variable when every use is a primitive `sync/atomic` call (`int32` becomes `atomic.Int32`). Run `go fix`. It leaves the variable alone when any use is a plain read or write, so a race is not rewritten into a typed load. Follow the `atomic.Bool` example, including `Load`, even when `atomictypes` is silent on that racy read. Read `src/atomic.md`.

### Avoid Mutable Globals

Do not mutate a package-level variable, including a function pointer. Inject the dependency (a field on a struct, set in the test). Read `src/global-mut.md`.

### Avoid Embedding Types in Public Structs

Do not embed an implementation struct or interface in a public struct. Write the delegate methods. An embedded type leaks the implementation, freezes the API (added interface methods, removed struct methods, removing or replacing the embedded type are breaking changes), and hides the real method set. Read `src/embed-public.md`.

### Avoid Using Built-In Names

Do not reuse a predeclared identifier (`error`, `string`, and the rest of the spec list) as a name, including as a field name. Shadowing hides the builtin; a colliding field name is hard to grep and easy to misread. The guide says `go vet` should point out shadowing. The compiler does not reject these names. Avoid the field-name collision even when no checker reports it. Read `src/builtin-name.md`.

### Avoid `init()`

Avoid `init()`. When it is unavoidable, keep it deterministic, independent of other `init` functions, free of global and environment state (machine info, environment variables, working directory, arguments), and free of I/O. Anything else belongs in a helper called from `main`, or in `main`. A library must not perform init magic.

`init` is acceptable for a complex expression that cannot be a single assignment, a pluggable hook such as a `database/sql` dialect or an encoding registry, or deterministic precomputation. Read `src/init.md`.

### Exit in Main

Call `os.Exit` or `log.Fatal*` only in `main`. Every other function returns an error. Do not exit by panicking. A function that exits is hard to follow, aborts `go test`, and skips `defer` cleanup.

#### Exit Once

Prefer a single `os.Exit` or `log.Fatal*` in `main`. Put the work in a function that returns an error (or an exit code, if the program must distinguish codes). The helper's name and signature are not fixed: it may take arguments, parse them in `main`, carry the code in a custom error, or sit outside `package main`. What matters is that one place in `main` exits. The same rule applies to any library call that exits. Read `src/exit-main.md` and `src/exit-once.md`.

### Use field tags in marshaled structs

Put a field tag on every struct field that is marshaled to JSON, YAML, or another format that names fields with tags. The tag is the contract. Renaming the Go field must not change the serialized name.

### Don't fire-and-forget goroutines

Do not leak a goroutine in production code. Each goroutine has a predictable stop time, or a signal that tells it to stop, and the caller can block until it exits. In a package that starts goroutines, test for leaks with `go.uber.org/goleak`. Read `src/goroutine-forget.md`.

#### Wait for goroutines to exit

Wait for several goroutines with `sync.WaitGroup` (`WaitGroup.Go`, then `Wait`), as in `src/goroutine-exit.md`. Wait for one goroutine with a `chan struct{}` that it closes when it returns.

#### No goroutines in `init()`

Do not start a goroutine from `init`. Expose a value the caller constructs, with a method (`Close`, `Stop`, `Shutdown`) that signals the goroutine and waits for it. Use a `WaitGroup` when that value owns more than one goroutine. Read `src/goroutine-init.md`.

## Performance

Apply these rules only on the hot path.

### Prefer strconv over fmt

Convert between primitives and strings with `strconv`, not `fmt`.

### Avoid repeated string-to-byte conversions

Convert a fixed string to `[]byte` once and reuse that slice. Do not convert it again on each use.

### Prefer Specifying Container Capacity

Pass a capacity hint to `make` when the size is known. A map hint is approximate and does not reserve every slot. A slice capacity is exact: `append` does not allocate until the length reaches that capacity. Read `src/container-capacity.md`.

## Style

### Avoid overly long lines

Aim to wrap before 99 characters. The limit is soft, and code may exceed it.

### Be Consistent

Be consistent. Apply a guideline at package scope or larger so one package does not contain two styles.

### Group Similar Declarations

Group related `import`, `const`, `var`, and `type` declarations. Do not put unrelated constants in one `const` block. Inside a function, group variables that are declared next to each other, even when they are unrelated.

### Import Group Ordering

Keep two import groups: the standard library, then everything else. `goimports` does this. Run `goimports`.

### Package Names

Use a short, lowercase, singular name, with no underscores. The name should not need a rename at most call sites. Do not name a package `common`, `util`, `shared`, or `lib`.

### Function Names

Use MixedCaps for function names. Exception: a test function may contain underscores to group cases, as in `TestMyFunction_WhatIsBeingTested`.

### Import Aliasing

Alias an import when the package name is not the last element of the path. Otherwise do not alias, unless two imports would use the same name. When they conflict, alias the import whose name collides; leave the one that already matches its path.

### Function Grouping and Ordering

In a file, put `struct`, `const`, and `var` first, then exported functions, in rough call order. Group functions by receiver. `newXYZ` or `NewXYZ` comes immediately after the type and before the other methods. Put plain helpers at the end of the file.

### Reduce Nesting

Handle the error or the special case first and `return` or `continue`. Keep the main path at the shallowest level.

### Unnecessary Else

When both branches only assign the same variable, assign the common value and overwrite it in a single `if`. Delete the `else`.

### Top-level Variable Declarations

At package scope, use `var` and omit the type when it matches the expression. Write the type when the expression's type is not the type you want (`var _e error = F()` when `F` returns a concrete error).

### Prefix Unexported Globals with _

Prefix an unexported top-level `var` or `const` with `_`. Exception: an unexported error value uses `err` and no underscore. See Error Naming.

### Embedding in Structs

Put embedded fields first in the struct, then a blank line, then the other fields. Embed only when every promoted exported method and field is one you would add to the outer type directly, and the zero value, copy behavior, and exported surface stay honest. Otherwise use a named field. Do not embed a mutex. Read the rest of the constraints in `src/struct-embed.md`.

### Local Variable Declarations

Use `:=` when the variable is set to an explicit value. Use `var` when the useful value is the zero value, including a slice you will `append` to.

### nil is a valid slice

Return `nil` for an empty slice, not `[]T{}`. Test emptiness with `len(s) == 0`, not a `nil` comparison. A `var` slice is ready to `append` without `make`. A nil slice and a non-nil length-zero slice are not interchangeable (serialization can tell them apart). staticcheck `S1009` flags a nil check that is redundant next to `len` on a slice, map, or channel. Run staticcheck. Returning `nil`, and testing a slice with `len`, still apply where `S1009` is silent.

### Reduce Scope of Variables

Declare a variable in the smallest scope that works. Do not shrink scope when that fights Reduce Nesting. If the value is needed after the `if`, do not declare it in the `if` and handle the `else` as the error path. Keep a constant local unless it is used in more than one function or file, or it is part of the package's external contract.

### Avoid Naked Parameters

When a naked argument's meaning is not obvious, name it with a C-style comment (`true /* isLocal */`). Better, replace a naked `bool` with a named type so the parameter can grow past two states.

### Use Raw String Literals to Avoid Escaping

Use a raw string literal instead of a hand-escaped interpreted string. staticcheck `S1007` suggests a raw string for a regular expression. Run staticcheck. Follow this rule for other escaped strings even when `S1007` is silent. Read `src/string-escape.md`.

### Initializing Structs

#### Use Field Names to Initialize Structs

Name the fields of a struct literal.

`go vet`'s `composites` check reports an unkeyed literal only when the struct type comes from another package. Run `go vet`. `go fix` does not run `composites`.

Keyed fields on a same-package struct are house judgment. Follow the keyed form even when `go vet`, `go fix`, and staticcheck are silent.

Exception: a test-table struct with 3 or fewer fields may omit the names.

Go 1.27 allows a literal key to be any field selector. `go fix` runs `embedlit`, which rewrites `T{U: U{x: 1}}` to `T{x: 1}`. Run `go fix`. Do not write the nested embedded-type form. A promoted field name is still a field name. Read `src/struct-field-key.md`.

#### Omit Zero Value Fields in Structs

Omit a zero-value field unless the name adds meaning. Include the zero in a test table when the column name is the point of the case. Read `src/struct-field-zero.md`.

#### Use `var` for Zero Value Structs

When every field is omitted, declare the value with `var`, not `T{}`.

#### Initializing Struct References

Initialize a struct pointer with `&T{}` (and its fields), not `new(T)` followed by assigning the fields. This is not a request to switch other `new` calls.

### Initializing Maps

Use `make` for an empty map and for a map filled in code, so initialization is not visually a nil declaration, and so a capacity hint can be added. A nil map panics on write. Use a map literal for a fixed set of elements.

### Format Strings outside Printf

Declare a format string that is not a literal at the call as a `const`, so `go vet` can check it. Run `go vet`.

### Naming Printf-style Functions

Give a `Printf`-style function a name `go vet` already knows, or a name that ends in `f` (`Wrapf`, not `Wrap`). For a custom `f` name, run `go vet -printfuncs=` with those names.

## Patterns

### Test Tables

Use a table when the same logic runs against many inputs and outputs. Name the slice `tests` and the case `tt`. Prefix inputs with `give` and outputs with `want`. A table struct with 3 or fewer fields may omit field names.

Do not put complex or conditional logic in the loop: several branching fields (`shouldCallX`, `expectCall`), per-case mock setup functions, or assertions that depend on earlier assertions. Split that into more than one table or into separate `Test` functions. Prefer a narrow behavior, shallow assertions, every field used by every case, and the same logic for every case. A short body may branch once on success versus failure with a field such as `shouldErr`.

Do not write `tt := tt` before `t.Parallel`. A `t.Parallel` subtest uses the range variable directly. `go fix` runs `forvar`, which deletes that copy in a `range` loop. Run `go fix`. `forvar` does not edit a three-clause `for`. Table tests use `range`. Read `src/test-table.md`.

### Functional Options

For optional arguments on a constructor or other public API that is expected to grow, especially once the function already has three or more arguments, take a variadic `Option`. Use an `Option` interface with an unexported method that records onto an unexported options struct. Prefer that over a closure: the option value can be compared in tests and can implement other interfaces, including `fmt.Stringer`. This pattern is house judgment. Read `src/functional-option.md`.

## Linting

Lint the same way across the codebase. At minimum run errcheck, goimports, revive, govet, and staticcheck. `revive` is the successor of `golint`; do not enable `golint`. The recommended runner is golangci-lint. This repo's `.golangci.yml` is the example config. A team may add linters. Read `src/lint.md`.

## Before you finish

On the Go files you changed, run:

1. `gofmt`
2. `go vet`
3. `staticcheck`, when the module is available
