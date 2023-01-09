---
title: "Errors"
weight: 3
description: >
  How to create and use errors.
---

Create custom errors in the `errors.go` file. You can use these errors during error handling instead of using error strings. Essentially, you are wrapping errors with additional messages to provide more information for the user while keeping the original error available for inspection (usually during tests) with `errors.Is(err, expectedErr)`.

Custom errors use the format `Err*`:
```go
var (
	ErrNotNumber        = errors.New("Data is not numeric")
	ErrInvalidColumn    = errors.New("Invalid column number")
	ErrNoFiles          = errors.New("No input files")
	ErrInvalidOperation = errors.New("Invalid operation")
)
```

`fmt.Errorf` creates a custom formatted error:
```go
return fmt.Errorf("Error: %s is not a valid string", s)
```

Test if the action returns a specific error. For example, the following snippet returns `nil` if the file does not exist; otherwise, it returns the error:
```go
file, err := os.ReadFile(filename)
if err != nil {
    if errors.Is(err, os.ErrNotExist) {
        return nil
    }
    return err
}
```

#### Custom error types



#### Compact error checking

If a function or method returns only an error, you can assign any error and check it for nil on the same line:
```go
if err := returnErr(); err != nil {
    // handle error
}
```
#### Returning errors

Return only an error if you want to check that a method performs an operation correctly:

```go
func Add(a *int, b int) error {
    a += b
    return nil
}
```
When you ae returning an error, use STDERR instead of STDOUT to display error messages, and exit with a code other than `1`:
```go
if err := l.Get(todoFileName); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
}
```