---
title: "Servers"
weight: 70
description: >
  Working with Servers in Go.
---


The `net/http` package provides the `ListenAndServer()` function that creates a default server. However, this function does not allow you to define timeouts to manage bad connections or server resources, so you should define custom server.

## Server type

Create a custom server with the [`Server` type](https://pkg.go.dev/net/http#Server). The `Server` type is a struct with the following fields:

```go
type Server struct {
	Addr              string
	Handler           Handler
	TLSConfig         *tls.Config
	ReadTimeout       time.Duration
	ReadHeaderTimeout time.Duration
	WriteTimeout      time.Duration
	IdleTimeout       time.Duration
	MaxHeaderBytes    int
	TLSNextProto      map[string]func(*Server, *tls.Conn, Handler)
	ConnState         func(net.Conn, ConnState)
	ErrorLog          *log.Logger
	BaseContext       func(net.Listener) context.Context
	ConnContext       func(ctx context.Context, c net.Conn) context.Context
}
```

## Writing a basic server

There are two important concepts that are central to creating a custom server in Go:
- `Server` type: listens for client connections and routes incoming requests in a goroutine to a type that implements the `Handler` interface.
- `Handler` interface: contains one method, `ServeHTTP(ResponseWriter, *Request)`.

You can create a server with any type as long as it implements `Handler` interface. A server should have the following:
- One or more registered routes with an associated function to handle requests to that route.
- A server multiplexer that can manage multiple registered routes.

### Directory structure

A suggested server directory structure:

```shell
.
├── cmd
│   └── serverdir
│       └── server.go
├── internal
│   └── helpers
│       ├── handler.go
│       └── helpers.go
├── errors
│   └── errors.go
└── server-name
    ├── server.go
    ├── server_test.go
    └── app.go

```

In the preceding example:
- `cmd/serverdir/server.go` contains the `main` method. This is where you create a logger and execute the server's `ListenAndServe()` method.
- `internal/helpers/*` contains private code such as JSON formatters and handlers for the handler chaining pattern.
- `errors/errors.go` contains custom errors.
- `server-name/server.go` contains the custom server implementation, including the constructor, routes, etc.
- `server-name/server_test.go` tests `server.go` with the `httptest.NewRecorder()` method.
- `server-name/app.go` contains the server's business logic.





### Define the custom Server type

In the project root, create a subdirectory and file called `json-server/server.go`:

```bash
mkdir json-server
touch json-server/server.go
```

The `Server` type must implement the `Handler` interface, so embed the `Handler` interface in the Server type:

```go
type mux http.Handler

type Server struct {
	mux
}
```
Interface embedding elevates the interface methods to the containing type, so the `Server` implements the `Handler`'s `ServeHTTP` method.

### Define routes

This server will have the following routes:

- `/json`: accepts a request in key-value pairs, and returns the request in JSON format.
- `/health`: returns a simple message that the server is running.

At the top of the file above (`type mux http.Handler`), create `const` values that define the routes that your server will handle.

```go
const (
	jsonRoute = "/json"
	healthCheckRoute = "/health"
)
```
### Register routes

Create a method on the custom `Server` type called `registerRoutes()`. This function will create a new multiplex server, register handlers for each route, and assign the multiplex server to the `Server.mux`:

```go
func (s *Server) registerRoutes() {
	mux = http.NewServeMux()
	// TODO: register routes
	s.mux = mux
}
```

> All route handlers should end with `*Handler`.

#### /health handler

The `/health` handler is the easiest to implement because it replies with a simple response:

1. Create a method on `Server` called `healthCheckHandler`:
   ```go
   func (s *Server) healthCheckHandler(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "server is running")
   }
   ```
2. Register the handler with `http.HandleFunc`. This method registers a handler with a multiplexer server. It accepts two arguments: the path as a `string`, and the handler function. It implements `ServeHTTP()`:
   ```go
   func (s *Server) registerRoutes() {
	   mux = http.NewServeMux()
	   mux.HandleFunc(healthCheckRoute, s.healthCheckHandler)
	   // TODO: register routes
	   s.mux = mux
   }
   ```

#### /json handler

The initial implementation of the `/json` handler responds with a simple success message to verify that the server is correctly implemented. Serving JSON requires additional packages that will be implemented during a later step:

1. Create a method on `Server` called `jsonHandler`:
   ```go
   func (s *Server) jsonHandler(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusCreated)
		fmt.Fprintln(w, "serving json")
   }
   ```
   For a list of standard methods and statuses, see the [net/http package docs](https://pkg.go.dev/net/http#pkg-constants).
2. Register the handler with `http.HandleFunc`:
   ```go
   func (s *Server) registerRoutes() {
	   mux = http.NewServeMux()
	   mux.HandleFunc(healthCheckRoute, s.healthCheckHandler)
	   mux.HandleFunc(jsonRoute, s.jsonHandler)
	   s.mux = mux
   }

### Create a Server constructor

Create the `NewServer()` constructor method. This method creates a new server type, registers the routes to the server, then returns the server:

```go
func NewServer() *Server {
	var s Server
	s.registerRoutes()
	return &s
}
```

### Create the main method

1. In the project root, create the main `server.go` file:

   ```bash
   mkdir -p cmd/server/server.go
   ```
1. In the `main` method, define the `address` that the server listens on, and a `timeout` duration. The `timeout` makes the server return after `timeout` seconds if the response is not complete:
   ```go
   const (
       addr    = "localhost:8080"
	   timeout = 10 * time.Second
   )
   ```
1. Create a new multiplexer server:
   ```go
   server := server.NewServer()
   ```
1. Create the server. Assign the `address` and `timeout` variables. For `Handler`, use `TimeoutHandler` so you can assign the timeout value:
   ```go
   server := &http.Server{
	   Addr:        addr,
	   Handler:     http.TimeoutHandler(server, timeout, "timeout"), // runs server for timeout seconds
	   ReadTimeout: timeout, // max time for reading the entire request
   }
   ```
1. Start the server. The normal behavior for stopping a server returns the `http.ErrServerClosed` error, so check for that error and return a message:
   ```go
   if err := server.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
	   fmt.Fprintln(os.Stderr, "server closed unexpectedly:", err)
   }   
   ```



## Existing docs


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

