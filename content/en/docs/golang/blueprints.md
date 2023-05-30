---
title: "Blueprints"
linkTitle: "Blueprints"
weight: 200
description: >
  Implementation details for application strategies.
---

## Web applications

```shell
root
├── cmd
│   └── web
│       ├── context.go
│       ├── handlers.go
│       ├── helpers.go
│       ├── main.go
│       ├── middleware.go
│       ├── routes.go
│       └── templates.go
├── internal
│   ├── models
│   │   ├── errors.go
│   │   ├── snippets.go
│   │   └── users.go
│   └── validator
│       └── validator.go
├── tls
│   ├── cert.pem
│   └── key.pem
└── ui
    ├── html
    │   ├── base.tmpl.html
    │   ├── pages
    │   │   ├── create.tmpl.html
    │   │   ├── home.tmpl.html
    │   │   ├── login.tmpl.html
    │   │   ├── signup.tmpl.html
    │   │   └── view.tmpl.html
    │   └── partials
    │       └── nav.tmpl.html
    └── static
        ├── css
        │   ├── index.html
        │   └── main.css
        ├── img
        │   ├── favicon.ico
        │   ├── index.html
        │   └── logo.png
        ├── index.html
        └── js
            ├── index.html
            └── main.js

```
### `/cmd`

Holds executables

#### `/web`

Holds executables for the web application:
  - `context.go`
  - `handlers.go`
  - `helpers.go`
  - `main.go`
  - `middleware.go`
  - `routes.go`
  - `templates.go`

### `/internal`

#### `/models`

#### `/validator`

### `/tls`

### `/ui`

#### `/html`

#### `/static`


## Template data

## Hashing and storing passwords