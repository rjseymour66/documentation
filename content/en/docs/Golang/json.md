---
title: "JSON data"
weight: 4
description: >
  How to work with JSON data.
---

#### Structs and struct tags

When you define a struct to model a JSON response or request, use struct tags to map a struct field to a field in the JSON object. 

Struct tags are enclosed in backticks (\`\`). Use the format `json:"json_field"`, using snake case when necessary. For example:

```go
type responseFormat struct {
	Results todo.List `json:"results"`
}
```
Capitalize the field name so that you can export it as JSON with Go's native JSON encoding.

#### Encoding JSON to a stream

Use the [json.Encoder](https://pkg.go.dev/encoding/json@go1.19.4#Encoder) and its `.Endcode()` method to write the program's memory representation of JSON (a struct) to an output stream (an `io.Writer`). `Encoder` accepts the writer, and `.Encode()` takes the values that you want to encode. You can chain the methods. For example:

```go
var buffer bytes.Buffer

objToJSON := struct {
    Value string `json:"value"`
}{
    Value: stringName
}

if err := json.NewEncoder(&buffer).Encode(item); err != nil {
    // handle error
}
```

#### Decoding JSON from stream

The [json.Decoder](https://pkg.go.dev/encoding/json@go1.19.4#Decoder) is just like the `json.Encoder`, except that it reads from an input stream (`io.Reader`) and decodes JSON into its memory representation (a struct):

```go

```

#### Marshal methods

> **IMPORTANT**: Always pass pointers to `json.Marshall` and `json.Unmarshall`.

Marshalling transforms a memory representation of an object into the JSON data format for storage or transmission. In other words, it returns data in JSON format.

Create a MarshallJSON() method when you have to map structs to complex JSON objects. For example:

```go
func (r *todoResponse) MarshallJSON() ([]byte, error) {
	resp := struct {
		Results      todo.List `json:"results"`
		Date         int64     `json:"date"`
		TotalResults int       `json:"total_results"`
	}{
		Results:      r.Results,
		Date:         time.Now().Unix(),
		TotalResults: len(r.Results),
	}

	return json.Marshal(resp)
}
```
The preceding method models a response with an anonymous struct, then returns the response in JSON format.


#### Unmarshalling
Unmarshalling transforms a JSON object into a memory representation that is executable.

To unmarshall a JSON object into memory, pass the data and a pointer to the data structure that you want to store the data in:
```go
type person struct {
    name    string
    age     int
}

var jsonData := `[
{"name": "Steve", "age": "21"},
{"name": "Bob", "age": "68"}
]`

var unmarshalled []person

json.Unmarshall(data, &unmarshalled)
```