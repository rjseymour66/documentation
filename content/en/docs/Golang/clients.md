---
title: "Clients"
weight: 110
description: >
  Working with Clients in Go.
---


## Clients

In Go, a single client can create multiple connections.

### Creating clients

Go provides a default client, but you cannot customize it with functionality such as a connection timeout:

```go
func newClient() *http.Client {
	c := &http.Client{
		Timeout: 10 * time.Second,
	}
	return c
}
```

### Model responses

You have to model responses with structs. Create a struct to model an individual resource and a struct to model the server response.

### Sending requests

Create a generic method that can send any type of request and handle any response code. The `.Do()` method can send any type of request (GET, POST, PUT, DELETE, etc.).

A reqeust should perform the following:
- Create a request object with [.NewRequest(method, url string, body io.Reader)](https://pkg.go.dev/net/http#NewRequest)
- Set any content headers with [Header.Set(header-name, value)](https://pkg.go.dev/net/http#Header.Set)
- Execute the request with the .Do() method. Save the response in a var
- Close the response body (sooner than later)
- Check that the response code is what you expected. If not, use custom error messages with the `%w` formatting verb.
- If the request was successful, return `nil`

For example:

```go
func sendRequest(url, method, contentType string, expStatus int, body io.Reader) error {
    // create a new request
	req, err := http.NewRequest(method, url, body)
	if err != nil {
		return err
	}

    // Set any headers
	if contentType != "" {
		req.Header.Set("Content-Type", contentType)
	}

    // execute the request with .Do() and save the response in a var
	r, err := newClient().Do(req)
	if err != nil {
		return err
	}

    // make sure the response body is closed 
	defer r.Body.Close()

    // check status codes
	if r.StatusCode != expStatus {
		msg, err := io.ReadAll(r.Body)
		if err != nil {
			return fmt.Errorf("Cannot read body: %w", err)
		}
		err = ErrInvalidResponse
		if r.StatusCode == http.StatusNotFound {
			err = ErrNotFound
		}
		return fmt.Errorf("%w: %s", err, msg)
	}

    // return nil for a successful request
	return nil
}
```

### CRUD functions

Use the following functions with the generic `sendRequest()` function for CRUD operations:

```go


// PATCH
// Use Sprintf to format a url with query parameters
func completeItem(apiRoot string, id int) error {
	u := fmt.Sprintf("%s/todo/%d?complete", apiRoot, id)

	return sendRequest(u, http.MethodPatch, "", http.StatusNoContent, nil)
}
```

### Integration tests

When you run unit tests, you are using local resources that mock the live API. You can run these as much as you'd like. However, to run integration tests, you need to run your client against the actual API. To make sure that you do not make too many requests to the actual API, use build constraints.

Main challenge is that the test needs to be reproducible. 

Define build constraints at the top of the file:

```go
// +build integration

package cmd

// file contents...
```

To exclude a file from integration tests, use the `!` operator before `integration`:

```go
//go:build !integration

package cmd

// file contents
```
When you run the tests, use `-tags <tag-name>` in the command. For example:
```shell
$ go test -v ./cmd -tags integration
```

After you run the integration tests one time, add the `-count=1` tag to ensure that the test does not used cached results:

```shell
$ go test -v ./cmd -tags integration -count=1
```