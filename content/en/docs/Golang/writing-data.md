---
title: "Writing data"
weight: 6
description: >
  Working with files in golang.
---


#### Writing to a file

Write data to a file with the `os` package. `WriteFile` writes to an existing file or creates one, if necessary:
```go
os.WriteFile(filename, dataToWrite, perms)
```
> **Linux permissions**: Set Linux file permissions for the file owner, group, and all other users (`owner-group-others`). The permission options are read, write, and execute. Use octal values to set permssions:  
  read: 4  
  write: 2  
  execute: 1  

When you assign permissions in an programming language, you have to tell the program that you are using the octal base. You do this by beginning the number with a `0`. So, `0644` permissions means that the file owner has read and write permissions, and the group and all other users have read permissions.

#### Buffers and bytes

Many functions return `[]byte`, so you might have to fill a buffer with data to return.

The `bytes` package contains two types: `Buffer` and `Reader`. The `Buffer` returns a variable size buffer to read and write data. It provides `Write*` methods for `strings`, `runes`, `bytes`, etc.:

```go
func byteStuff() []bytes {
    // compose the page using a buffer of bytes to write to a file
    var buffer bytes.Buffer

    // write html to bytes buffer
    buffer.WriteString("The first string")
    buffer.Write([]byte{'T', 'h', 'e', 's', 'e', 'c', 'o', 'n', 'd', 's', 't', 'r', 'i', 'n', 'g'})
    buffer.WriteString("The last string")

    // return []bytes
    return buffer.Bytes()
}
```

#### tabWriter

`tabwriter.Writer` writes tabulated data with formatted columns with consistent widths using tabs (`\t`).

https://pkg.go.dev/text/tabwriter#pkg-overview