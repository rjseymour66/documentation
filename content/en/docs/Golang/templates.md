---
title: "Templates"
weight: 150
description: >
  Creating and working with templates in Go.
---

Templates can write dynamic webpages, config files, emails, etc. First, you have to parse the template, then you execute it.

## Parse a template

You can 


These are the general steps:

1. Parse the contents of a template file:
    ```go
    t, err = template.ParseFiles(templateFile)
    ```
2. Create a struct that contains the content to inject into the template. The following struct injects the title and body of a webpage:
    ```go
    c := content {
        Title: "This is the title",
        Body:  "This is the body text",
    }
    ```
3. Execute the template, and write the executed template into a writer, such as a buffer:
    ```go
    if err := t.Execute(&buffer, c); err != nil {
        return nil, error
    }
    ```
4. Return or use the buffer somehow.


## Template composition

You can compose templates to minimize how often you have to write boilerplate html. A common pattern is to create a base template that invokes other templates within its HTML.

To define a distinct template, use the `{{define `_`template-name`_`}}` and `{{end}}` actions:

```go
{{define "base"}}
<!DOCTYPE html>
<html lang="en">
<head>
    ...
    <title>{{template "title" . }} - Snippetbox</title>
</head>
<body>
    ...
    <main>
        {{template "main" .}}
    </main>
    ...
</body>
</html>
{{end}}
```
The `{{template `_`template-name`_`. }}` actions invoke other templates named _`template-name`_. The `.` represents any dynamic data that you want to pass to the template.

Define the other templates in the relevant pages. For example, if you want to define the content that will be in the homepage, create a file called `home.tmpl.html` and define the templates:

```go
{{define "title"}}Home{{end}}

{{define "main"}}
    <h2>Latest Snippets</h2>
    <p>There's nothing to see here yet!</p>
{{end}}
```

## Parse composed templates

To parse a template composed of more than one template, you must do the following:
1. Create a struct containing the template file names.
2. Parse the files into a template set with the `template.ParseFiles()` function.
3. Execute the template, passing the name of the parent template to the `ExecuteTemplate()` function.

For example, the following handler parses composed templates to serve a website homepage:

```go
func home(w http.ResponseWriter, r *http.Request) {
    if r.URL.Path != "/" {
        http.NotFound(w, r)
        return
    }

    // Initialize a slice containing the paths to the two files. It's important
    // to note that the file containing our base template must be the *first*
    // file in the slice.
    files := []string{
        "./ui/html/base.tmpl",
        "./ui/html/pages/home.tmpl",
    }

    // Use the template.ParseFiles() function to read the files and store the
    // templates in a template set. Notice that we can pass the slice of file
    // paths as a variadic parameter?
    ts, err := template.ParseFiles(files...)
    if err != nil {
        log.Print(err.Error())
        http.Error(w, "Internal Server Error", 500)
        return
    }

    // Use the ExecuteTemplate() method to write the content of the "base" 
    // template as the response body.
    err = ts.ExecuteTemplate(w, "base", nil)
    if err != nil {
        log.Print(err.Error())
        http.Error(w, "Internal Server Error", 500)
    }
}
```

## Partials

A partial is a smaller template component that you might reuse in multiple pages. For example, a navigation bar.



## Dynamic data

As stated earlier, the dot (`.`) represents any dynamic data that you want to pass to the template.

You can pass dynamic data when you execute the template. For example, you can pass a struct to the template set:

```go
err = ts.ExecuteTemplate(w, "base", snippet)
if err != nil {
    app.serverError(w, err)
}
```
When the template set is executed, it will include `snippet` as its dynamic data. If `snippet` is a struct (in this example, it is a struct), you can render the value of an exported field in your template by postfixing the dot with the field name.

For example, if you pass a snippet object to a template set:

```go
type Snippet struct {
	ID      int
	Title   string
	Content string
	Created time.Time
	Expires time.Time
}
```

You can dynamically render these any exported fields in a template, as follows:

```html
{{define "title"}}Snippet #{{.ID}}{{end}}

{{define "main"}}
<div class="snippet">
    <div class="metadata">
        <strong>{{.Title}}</strong>
        <span>#{{.ID}}</span>
    </div>
    <pre><code>{{.Content}}</code></pre>
    <div class="metadata">
        <time>Created: {{.Created}}</time>
        <time>Expires: {{.Expires}}</time>
    </div>
</div>
{{end}}
```

### Method access

If the underlying data type of a rendered field has a method, you can invoke that method and render it. For example, `Snippet.Created` is of type `time.Time`, so you can access the `.Weekday` method:

```html
<span>{{.Snippet.Created.Weekday}}</span>
```

If you access a method that accepts arguments, then you pass them by listing them after the method name, with a space between each argument:

```html
<span>{{.Snippet.Created.AddDate 0 6 0}}</span>
```


### Multiple pieces of dynamic data

By default, you can only pass one piece of dynamic data when executing a template set. A common workaround is to wrap the dynamic data in a struct.

To wrap the dynamic data, you must create a `templates.go` file in your `/cmd/web` directory, and wrap the data:

```go
type templateData struct {
	Snippet  *models.Snippet
	Snippets []*models.Snippet
}
```

Next, instantiate the wrapper struct, and pass it to the template set during execution:

```go
func (app *application) handlerView(w http.ResponseWriter, r *http.Request) {
    ...
    // Create an instance of a templateData struct holding the snippet data.
    data := &templateData{
        Snippet: snippet,
    }

    // Pass in the templateData struct when executing the template.
    err = ts.ExecuteTemplate(w, "base", data)
    if err != nil {
        app.serverError(w, err)
    }
}

```

Use dot notation to access the fields in the template:

```html
{{define "title"}}Snippet #{{.Snippet.ID}}{{end}}

{{define "main"}}
<div class="snippet">
    <div class="metadata">
        <strong>{{.Snippet.Title}}</strong>
        <span>#{{.Snippet.ID}}</span>
    </div>
    <pre><code>{{.Snippet.Content}}</code></pre>
    <div class="metadata">
        <time>Created: {{.Snippet.Created}}</time>
        <time>Expires: {{.Snippet.Expires}}</time>
    </div>
</div>
{{end}}
```

### Pipelining nested templates

When you invoke a template from within another template, you must pipeline the dot to the template you are invoking. To do so, include the dot at the end of each `template` or `block` action:

```html
<body>
    <header>
        <h1><a href="/">Snippebox</a></h1>
    </header>
    {{template "nav" .}}
    <main>
        {{template "main" .}}
    </main>
    <footer>Powered by <a href="https://golang.org/">Go</a></footer>
    <script src="/static/js/main.js"></script>
</body>
```
In the preceding example, the dynamic data is passed to the "nav" and "main" templates.

## Actions

_Actions_ are components that render data using conditional logic and the value of dot.

### Blocks

A block is a template that can include default content if the page invokes a template that is not in the executed template set.

### if
`{{if}}` renders content conditionally, with the `{{else}}` and `{{end}}` actions.

### with

`{{with}}` changes the value of dot. When dot holds a value, any dot within its scope is set to its value.

For example, the dot in the following template represents the templateData struct wrapper:

```go
type templateData struct {
	Snippet  *models.Snippet
	Snippets []*models.Snippet
}
```

```html
{{define "title"}}Snippet #{{.Snippet.ID}}{{end}}

{{define "main"}}
<div class="snippet">
    <div class="metadata">
        <strong>{{.Snippet.Title}}</strong>
        <span>#{{.Snippet.ID}}</span>
    </div>
    <pre><code>{{.Snippet.Content}}</code></pre>
    <div class="metadata">
        <time>Created: {{.Snippet.Created}}</time>
        <time>Expires: {{.Snippet.Expires}}</time>
    </div>
</div>
{{end}}
```

You can use `{{with}}` to pass the `.Snippet` value to any dot within its scope:

```html
{{define "title"}}Snippet #{{.Snippet.ID}}{{end}}

{{define "main"}}
    {{with .Snippet}}
    <div class="snippet">
        <div class="metadata">
            <strong>{{.Title}}</strong>
            <span>#{{.ID}}</span>
        </div>
        <pre><code>{{.Content}}</code></pre>
        <div class="metadata">
            <time>Created: {{.Created}}</time>
            <time>Expires: {{.Expires}}</time>
        </div>
    </div>
    {{end}}
{{end}}
```
If there is a value assigned to `templateData.Snippet`, then any dots between `{{with .Snippet}}` and `{{end}}` become `Snippet`, rather than `templateData`.

### range
`{{range}}` changes the value of dot. It loops over data passed to the template:

```html
{{range .Snippets}}
<tr>
    <td><a href="/snippet/view?id={{.ID}}">{{.Title}}</a></td>
    <td>{{.Created}}</td>
    <td>{{.ID}}</td>
</tr>
{{end}}
```

You can use `{{if}}`, `{{continue}}`, and `{{break}}` during range loops just as you would in any C-derived language.

## Functions

The following table describes the most common templating functions:

| Function | Description |
|:---------|:------------|
| {{eq .Foo .Bar}} | Yields true if .Foo is equal to .Bar |
| {{ne .Foo .Bar}} | Yields true if .Foo is not equal to .Bar |
| {{not .Foo}} | Yields the boolean negation of .Foo |
| {{or .Foo .Bar}} | Yields .Foo if .Foo is not empty; otherwise yields .Bar |
| {{index .Foo i}} | Yields the value of .Foo at index i. The underlying type of .Foo must be a map, slice or array, and i must be an integer value. |
| {{printf "%s-%s" .Foo .Bar}} | Yields a formatted string containing the .Foo and .Bar values. Works in the same way as fmt.Sprintf(). |
| {{len .Foo}} | Yields the length of .Foo as an integer. |
| {{$bar := len .Foo}} | Assign the length of .Foo to the template variable $bar |

> In Go templating, _yeild_ means _render_.

## Caching templates