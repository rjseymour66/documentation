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

## Blocks

A block is a template that can include default content if the page invokes a template that is not in the executed template set.