---
title: "Project setup"
weight: 1
description: >
  Setting up a Go project.
---

## Makefile

Create a Makefile at the base of your project to manage project tasks, builds, and dependencies. A Makefile consists of targets, which are tasks that you can run by entering `make target-name`.


The following sections describe common Makefile idioms and targets for a basic Go project.

### Global variables

Add global variables at the top of the Makefile in all caps. For example, the following variable defines the Go version:

```Makefile
GO_VERSION := 1.19.4
```

To use this variable in the Makefile, enclose it in parentheses and prepend it with a `$`:

```Makefile
$(GO_VERSION)
```

### Build

Build your application:

```Makefile
build:
    go build -o appname path/to/main.go
```

### Test

Run go tests:

```Makefile
test:
    go test ./... -coverprofile=coverage.out
```

### Code coverage

View how much of the source code is adequetly tested:
> What does this do?

```Makefile
coverage:
	go tool cover -func coverage.out | grep "total:" | \
	awk '{print ((int($$3) > 80) != 1) }'
```

### Generate coverage report

Runs the Go coverage tool to generate an HTML page that describes what code is covered by tests:

```Makefile
report:
	go tool cover -html=coverage.out -o cover.html
```

### Format your source code

This is not an issue in VSCode, but you can add a target that verifies the Go source code is formatted correctly:

```Makefile
check-format:
	test -z $$(go fmt ./...)
```

### Static linters

[golangci-lint](https://golangci-lint.run/) is a Go linters aggregator--it provides multiple linters for Go source code.

#### Install

```Makefile
install-lint: 
	sudo curl -sSfL \
	https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh \
 	| sh -s -- -b $$(go env GOPATH)/bin v1.51.1
```

#### Run the linter

```Makefile
static-check:
	golangci-lint run ./...
```

## Github Actions