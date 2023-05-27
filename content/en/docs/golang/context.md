---
title: "Context"
weight: 140
description: >
  Working with the context package.
---

[Context](https://go.dev/blog/context)

[`context` package](https://pkg.go.dev/context)

When a Go server receives a request, it passes it to a handler, and the handler runs in its own goroutine. Each goroutine needs access to the request-scoped values: tokens, timeouts, cancellations, etc.

## Context interface

The context type implements the Context interface, which has the following signature:

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```
- `Deadline()` returns the time when the context is cancelled. This helps determine whether there is enough time to complete the work, or to set timeouts for I/O operations.
- `Done()` returns a channel that signals to functions running with the context that the context is cancelled.
- `Err()` returns an error why the context is cancelled.
- `Value()` returns the request-scoped value associated with `key`.

## Context type

The context type has the following methods:

```go
func Background() Context
func TODO() Context
func WithValue(parent Context, key, val any) Context
```

## Derived contexts

`Background()` returns an empty context. It is the top-level context--when a context that is created with `Background()` is canceled, then all the contexts that are derived from that context are cancelled.

By default, the Context for an incoming request is canceled when the handler returns. You can control when a context is canceled using the following functions:

func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
: Returns a copy of `parent` whose Done channel closes as soon as parent.Done closes or when cancel is called.

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
: Returns a copy of `parent` whose Done channel closes as soon as parent.Done closes, when cancel is called, or timeout elapses.

func WithValue(parent Context, key, val any) Context
: Returns a copy of `parent` whose `Value` method returns `val` for `key`.










A function that accepts a context should always check if the context is canceled.

`Background` creates a non-cancelable context. Then, you can derive a cancelable timeout context using the `WithTimeout` function.

The signal package has a NotifyContext function to catch an OS signal.


## Contexts

If you are executing commands that must communicate over a network, you should use a timeout. To create a timeout, use `context.WithTimeout()`.

`context.WithTimeout()` creates a context from the parent context and a timeout value. To create a new, empty context, pass the `context.Background()` as the parent:

```go
...
ctx, cancel := context.WithTimeout(context.Background(), s.timeout)
defer cancel()
...
```
`context.WithTimeout()` returns the context and a `cancel` function to cancel the context when it is no longer required. You should defer the `cancel` function.

If the context expires because of the timeout, you can check the context's `.Err()' function for a `DeadlineExceeded` error:

```go
	cmd := exec.CommandContext(ctx, s.exe, s.args...)
	cmd.Dir = s.proj

	if err := cmd.Run(); err != nil {
		if ctx.Err() == context.DeadlineExceeded {
			return ...
			}
		}
        ...
```