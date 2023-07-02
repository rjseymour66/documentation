---
title: "PostgreSQL"
weight: 30
description: >
  Connecting and working with PostgreSQL and Go.
---


## Import the driver

1. Go to https://github.com/lib/pq
2. Go to the root of the project, and use `go get` to download the driver:
   ```shell
   $ go get github.com/lib/pq@v1
   ```
3. In the Go file that runs the SQL code, import the driver. Because you are not using the driver directly, import it with the `blank identifier`:
   ```go
   package main

   import "_ github.com/lib/pq"
   ```

## Get a database connection pool

The Postgres data source name (DSN) uses the following format:

```
postgres://<username>:<password>@<host>/<dbname>
```

If you set the DSN as an environment variable, you can use it to authenticate to the database:

```shell
$ psql $POSTGRES_DSN_ENV_VAR
```