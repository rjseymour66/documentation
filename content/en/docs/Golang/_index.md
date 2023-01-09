---
title: "Golang"
linkTitle: "Golang"
weight: 1
description: >
  How to use golang
---

## Todo

0. How to instantiate each type:
    - string, bool (false is the zero type), int, float
    - pointers
    - structs
    - functions
    - container types:
        - arrays
        - slice
        - maps
    - channels
    - interfaces
  Instantiation methods:
    - standard declaration
    - literal

1. Comma ok idiom
2. How to structure custom errors (see goci)
3. How to test equality
4. How to read from STDIN and a flag
5. Type embedding
6. Returning functions (closures), like returning the cleanup function (p. 240), or any of the Cobra *Action() funcs
7. How to parse query parameters with [URL.Query()](https://pkg.go.dev/net/url#URL.Query)
8. Encoding/decoding vs Marshalling/UnMarshalling
   - I think encoding writes bytes, marshalling writes strings into structs
9. Explain any Flush() methods https://pkg.go.dev/text/tabwriter#Writer.Flush
10. Write a section about designing a client, explaining funcs
11. Describe how to write Cobra CLI tools w Viper
12. How do build tags work? [Digital Ocean](https://www.digitalocean.com/community/tutorials/customizing-go-binaries-with-build-tags)