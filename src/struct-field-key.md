# Use Field Names to Initialize Structs

You should almost always specify field names when initializing structs.

`go vet`'s `composites` check reports an unkeyed composite literal
only when the struct type is imported from another package
([`composites`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/composite)).
`go vet` on the program below reports
`net.DNSConfigError struct literal uses unkeyed fields`:

```go
_ = &net.DNSConfigError{err}
```

The same check does not report an unkeyed literal of a struct
defined in the same package.
`go vet` is silent on `User{"John", "Doe", true}` below.
`go fix` does not run `composites`:
some types should stay unkeyed
(`src/cmd/vendor/golang.org/x/tools/go/analysis/suite/fix/fix.go`
at tag `go1.27.3`).

Keyed fields for a same-package struct are house judgment.
Follow the keyed example even when `go vet`, `go fix`, and staticcheck
are silent,
except for the test-table case below.

Go 1.27 allows a struct-literal key to be any field selector,
not only a top-level field name
([Go 1.27 release notes](https://go.dev/doc/go1.27#language),
[issue #9859](https://go.dev/issue/9859)).
The `embedlit` modernizer deletes a redundant embedded-field type.
`go fix -json` on the bad literal reports
`embedded field type can be removed from struct literal`
([`embedlit`](https://pkg.go.dev/golang.org/x/tools/go/analysis/passes/modernize#embedlit)).
This section does not require that nested form.
A promoted field name is still a field name.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
return T{U: U{x: 1}}
```

</td><td>

```go
return T{x: 1}
```

</td></tr>
</tbody></table>

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
k := User{"John", "Doe", true}
```

</td><td>

```go
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

</td></tr>
</tbody></table>

Exception: Field names *may* be omitted in test tables when there are 3 or
fewer fields.

```go
tests := []struct{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```
