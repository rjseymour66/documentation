---
title: "Go tooling"
weight: 2
description: >
  XXXXXXXXXXXXXxx
---

#### Cross-compilation

Build static go binaries for operating systems that are different than the one that you are building it on. Because you build a static binary, the target machine does not need any additional libraries or tools to run the binary.

For example, use the `GOOS` environment variable with the `build` command to compile for a Windows machine:

```go
$ GOOS=window go build
// For a list of accepted GOOS values, see https://go.dev/src/go/build/syslist.go
```

## go.mod and go.sum

Go modules group related packages into a single unit to be versioned together. Because they track an application's dependencies, they ensure that users build the application with the same dependencies as the original developer.

Go modules allow you to write go programs outside of the $GOPATH directory, as in previous releases. 

Go sum records the checksum for each module in the application to ensure that each build uses the correct version.