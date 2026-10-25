# Introduction

Styles are the conventions that govern our code. The term style is a bit of a
misnomer, since these conventions cover far more than just source file
formatting—gofmt handles that for us.

The goal of this guide is to manage this complexity by describing in detail the
Dos and Don'ts of writing Go code at Uber. These rules exist to keep the code
base manageable while still allowing engineers to use Go language features
productively.

This guide was originally created by [Prashant Varanasi] and [Simon Newton] as
a way to bring some colleagues up to speed with using Go. Over the years it has
been amended based on feedback from others.

  [Prashant Varanasi]: https://github.com/prashantv
  [Simon Newton]: https://github.com/nomis52

This documents idiomatic conventions in Go code that we follow at Uber. A lot
of these are general guidelines for Go, while others extend upon external
resources:

1. [Effective Go](https://go.dev/doc/effective_go)
2. [Go Common Mistakes](https://go.dev/wiki/CommonMistakes)
3. [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments)

We aim for the code samples to be accurate for the two most recent minor versions
of Go [releases](https://go.dev/doc/devel/release).

All code should be error-free when run through `revive` and `go vet`.
We recommend setting up your editor to:

- Run `goimports` on save
- Run `revive` and `go vet` to check for errors

`golint` is deprecated and frozen.
The [golang/lint README] says so,
and [golang/go#38968] is the freeze.
There is no Go release note for that freeze.
Upstream replaced `golint` with `revive` in
[commit 860b988a94b53bb1b1f4c6ed9957e5a645f4eaff] ([#253]).
That pull request has no review comments.
The linter set is [Linting](lint.md).

  [golang/lint README]: https://github.com/golang/lint/blob/master/README.md
  [golang/go#38968]: https://github.com/golang/go/issues/38968
  [commit 860b988a94b53bb1b1f4c6ed9957e5a645f4eaff]: https://github.com/uber-go/guide/commit/860b988a94b53bb1b1f4c6ed9957e5a645f4eaff
  [#253]: https://github.com/uber-go/guide/pull/253

You can find information in editor support for Go tools here:
<https://go.dev/wiki/IDEsAndTextEditorPlugins>
