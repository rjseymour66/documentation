---
title: "Servers"
weight: 70
description: >
  Working with Servers in Go.
---


The `net/http` package provides the `ListenAndServer()` function that creates a default server. However, this function does not allow you to define timeouts to manage bad connections or server resources, so you should define custom server.

## Writing a server

There are two important concepts that are central to creating a custom server in Go:
- `Server` type: listens for client connections and routes incoming requests in a goroutine to a type that implements the `Handler` interface.
- `Handler` interface: contains one method, `ServeHTTP(ResponseWriter, *Request)`.

You can create a server with any type as long as it implements `Handler` interface. A server should have the following:
- One or more registered routes with an associated function to handle requests to that route.
- A server multiplexer that can manage multiple registered routes.

### Define the custom Server type

The `Server` type must implement the `Handler` interface, so embed the `Handler` interface in the Server type:

```go
type mux http.Handler

type Server struct {
	mux
}
```
Interface embedding elevates the interface methods to the containing type, so the `Server` implements the `Handler`'s `ServeHTTP` method.

### Define routes

Create `const` values that define the routes that your server will handle. This server will handle `/json` and `/health` routes:

```go
const (
	jsonRoute = "/json"
	healthCheckRoute = "/health"
)
```

### Register routes



## Existing docs

### Custom servers

Create a custom server with the [`Server` type](https://pkg.go.dev/net/http#Server). The `Server` type is a struct with the following fields:

```go 
type Server struct {
	Addr string
	Handler Handler
	TLSConfig *tls.Config
	ReadTimeout time.Duration
	ReadHeaderTimeout time.Duration
	WriteTimeout time.Duration
	IdleTimeout time.Duration
	MaxHeaderBytes int
	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)
	ConnState func(net.Conn, ConnState)
	ErrorLog *log.Logger
	BaseContext func(net.Listener) context.Context
	ConnContext func(ctx context.Context, c net.Conn) context.Context
}
```

Implement the fields that you need when you define a custom server:

```go 
customServer := &http.Server{
    Addr:         fmt.Sprintf("%s:%d", *host, *port),
    Handler:      MuxFunc(*todoFile),
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
}
```

Start the server with `ListenAndServe`:

```go
if err := s.ListenAndServe(); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
}
```

### Multiplexers

A multiplexer maps incoming requests to the proper handler functions using the request URL. `net/http` provides the DefaultServeMux function that returns the default multiplexer, but you should define a custom multiplexer for the following reasons:
- The default registers routes globally.
- You can add dependencies to the routes.
- Custom multiplexers allow integration testing

### HTTP handlers

Handlers handle a request and responds to it. An object of type `Handler` satisfies the `Handler` field in a custom HTTP server.

You create the server, then use `HandleFunc` to register routes to handler functions. Then you use `HandlerFunc` to define the handler function for the route.

- `http.Handler` is an interface that has the `ServeHTTP(w ResponseWriter, r *Request)` signature.
- `http.HandlerFunc` is an adapter type that lets you define ordinary functions as HTTP handlers. It implements `ServeHTTP()`.
- `http.HandleFunc` registers a handler with a multiplexer server. It accepts two arguments: the path as a `string`, and the handler function. It implements `ServeHTTP()`.
- `http.NewServeMux()` returns a custom server. It implements `ServeHTTP()`.

### Router functions

```go
func todoRouter(todoFile string, l sync.Locker) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		list := &todo.List{}

		l.Lock()
		defer l.Unlock()
		if err := list.Get(todoFile); err != nil {
			replyError(w, r, http.StatusInternalServerError, err.Error())
			return
		}

		if r.URL.Path == "" {
			switch r.Method {
			case http.MethodGet:
				getAllHandler(w, r, list)
			case http.MethodPost:
				addHandler(w, r, list, todoFile)
			default:
				message := "Method not supported"
				replyError(w, r, http.StatusMethodNotAllowed, message)
			}
			return
		}

		id, err := validateID(r.URL.Path, list)
		if err != nil {
			if errors.Is(err, ErrNotFound) {
				replyError(w, r, http.StatusNotFound, err.Error())
				return
			}
			replyError(w, r, http.StatusBadRequest, err.Error())
			return
		}

		switch r.Method {
		case http.MethodGet:
			getOneHandler(w, r, list, id)
		case http.MethodDelete:
			deleteHandler(w, r, list, id, todoFile)
		case http.MethodPatch:
			patchHandler(w, r, list, id, todoFile)
		default:
			message := "Method not supported"
			replyError(w, r, http.StatusMethodNotAllowed, message)
		}
	}
}
```

Multiplexer definition:

```go
func newMux(todoFile string) http.Handler {
	m := http.NewServeMux()
	mu := &sync.Mutex{}

	m.HandleFunc("/", rootHandler)

	t := todoRouter(todoFile, mu)

	m.Handle("/todo", http.StripPrefix("/todo", t))
	m.Handle("/todo/", http.StripPrefix("/todo/", t))

	return m
}
```

## HTTP general

### Status codes

Use `http.StatusText()` to return the text for an HTTP status code. For example:
```go
http.StatusText(200)
```
Go provides status code constants to make code more human readable. For example:
`http.NotFound` when a client requests an unknown route
`http.StatusOK` 200
`http.StatusInternalServerError` return for standard server error
`http.StatusMethodNotAllowed` when a client uses an unsupported request method
`http.StatusBadRequest` 
`http.StatusCreated` 201. request succeeded and led to the creation of a resource

### Request methods

Go provides constant values to identify request methods:

```go
http.MethodGet
http.MethodPost
```

