---
title: "Errors"
weight: 30
description: >
  How to handle, create, and use errors.
---

Errors are values in Go. The following example is the most common format for error handling:
```go
value, err := funcThatReturnsValandErr() {
    if err != nil {
        // handle err
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

The preceding example sends the error to STDERR and exits the program.


### Compact error checking

If a function or method returns only an error, you can assign any error and check it for `nil` on the same line:
```go
if err := returnErr(); err != nil {
    // handle error
}
```
### Basic error handling

`fmt.Errorf` creates a custom formatted error:
```go
return fmt.Errorf("Error: %s is not a valid string", s)


### Compact error checking

If a function or method returns only an error, you can assign any error and check it for `nil` on the same line:
```go
if err := returnErr(); err != nil {
    // handle error
}
```

### Check for specific errors with .Is()

Check if an error is a specific type with the [`.Is()` function](https://pkg.go.dev/errors#Is). This function accepts the error value and an error type for comparison. This function is helpful during testings.

For example, the following snippet returns `nil` if the the error is `os.ErrNotExist` (the file does not exist); otherwise, it returns the error:
```go
file, err := os.ReadFile(filename)
if err != nil {
    if errors.Is(err, os.ErrNotExist) {
        return nil
    }
    return err
}
```

### Wrap errors

For errors, use `%w` to decorate the original error with additional information for the users. Essentially, you can customize the error message while also returning the default Go error:
```go
if err != nil {
    return nil, fmt.Errorf("Cannot read data from file: %w", err)
}
```
For more info, read [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-add-extra-information-to-errors-in-go).

## Custom error types

Create custom errors in the `errors.go` file. You can use these errors instead of error strings. Essentially, you are wrapping errors with additional messages to provide more information for the user while keeping the original error available for inspection (usually during tests) with `errors.Is(err, expectedErr)`.

Custom errors use the format `Err*`:
```go
var (
    ErrNotNumber        = errors.New("Data is not numeric")
    ErrInvalidColumn    = errors.New("Invalid column number")
    ErrNoFiles          = errors.New("No input files")
    ErrInvalidOperation = errors.New("Invalid operation")
)
```

## Returning errors from functions

Return only an error if you want to check that a method performs an operation correctly:

```go
func Add(a *int, b int) error {
    a += b
    return nil
}
```
When you are returning an error, use STDERR instead of STDOUT to display error messages, and exit with `1`:
```go
if err := l.Get(todoFileName); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
}
```