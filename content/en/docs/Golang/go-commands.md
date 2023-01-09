---
title: "go <command>"
weight: 1
description: >
  Go tooling and tips.
---

Format code:
```go
// formats the code
$ gofmt -w <file>.go
// lists files in dir that do not conform to go formatting 
$ gofmt -l dirname/*.go
```
Verify that you can build the file. Execute this from the directory with the `.go` files:
```shell
$ go build
```
Build the binary:
```shell
$ go build # uses module name for binary name
$ go build -o <binary-name>
```

Run a program, like a server:
```shell
$ go run .
```

Run tests:
```shell
# verbose output
$ go test -v
# tests for a specific dir
$ go test -v ./<dirname>/
$ go test -v ./cmd/
```
Add external dependencies to the project:
```shell
$ go get github.com/entire/module/path
```
When you import these packages into your project:
```go
import (
    "github.com/entire/module/path/v1"
    "github.com/entire/module/path/dependcy2"
)
```
Clean up go.mod and install missing dependencies:
```shell
$ cd <project-root dir>
$ go mod tidy
```
#### go mod

Update the `go.mod` file with [mod commands](https://go.dev/ref/mod#mod-commands).
List all go packages in the current directory tree:
```shell
# list packages
$ go list
# list modules
$ go list -m 
```