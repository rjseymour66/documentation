---
title: "MySQL"
weight: 20
description: >
  Connecting and working with MySQL and Go.
---

[golang/go SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers)

This section borrows heavily from [Tutorial: Accessing a relational database](https://go.dev/doc/tutorial/database-access). For information about transactions, query cancellation, and connection pools, see [Accessing a relational database](https://go.dev/doc/database/).

### Import the driver

1. Go to [SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers) and locate the [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql/) link.
2. In the go file that runs SQL code, import the driver. Because you are not using the driver directly, import it with the `blank identifier`:
   ```go
   package main

   import "_ github.com/go-sql-driver/mysql"
   ```
   Use the blank identifier because you do not use any methods from the `mysql` driver--you use only the driver's `init()` function so the driver can register itself with the `database/sql` package.

3. If you have not installed the package dependency, run `go get` in the shell:
   ```shell
   $ go get -u github.com/go-sql-driver/mysql
   ```
   This installs the latest version of the driver.

### Get a database connection pool

To get a database connection pool, you need to provide a database source name (DSN) to the `sql` package's `Open()` function. The `Open()` function returns a pool of many conncurrent database connections. Go opens and closes these connections through the database driver, as needed.

The DSN is different for each driver. The `mysql` driver has a `parseTime=true` option, which tells the driver to convert SQL `TIME` and `DATE` to Go `time.Time` objects.

The database connection pool should be long-lived and passed among handlers, so implement it in the `main` method. Include a flag for the `dsn` so you can easily use a different data source. In addition, include a helper function to encapsulate the database connection code:

```go
func main() {

	dsn := flag.String("dsn", "web:pass@/snippetbox?parseTime=true", "MySQL data source name")

	...

    db, err := openDB(*dsn)
    if err != nil {
        errorLog.Fatal(err)
    }
    defer db.Close()

	...
}

func openDB(dsn string) (*sql.DB, error) {
	// sql.Open(driver-name, DSN) {...}
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }
    // Connections are created lazily (as needed), so
    // ping the db to establish a connection and confirm 
    // everything is working.
    if err = db.Ping(); err != nil {
        return nil, err
    }
    return db, nil
}
```

### Data access layer

You want to encapsulate the code that works with MySQL in its own package. Create a new file in `root/internal/models/project.go` for your SQL data model.

The data access layer consists of the following:
- An interface that holds method signatures for the data object. This facilitates testing with mocks:
  ```go
  type SnippetModelInterface interface {
	  Insert(title string, content string, expires int)   (int, error)
	  Get(id int) (*Example, error)
	  Latest() ([]*Example, error)
  }
  ```
- A struct that holds data for each individual object that you want to commit to the database. Generally, the struct fields should correspond to the table columns:
  ```go
  type Example struct {
    ID      int
    Title   string
    Content string
    Created time.Time
    Expires time.Time
  }
  ```
- An `ExampleModel` struct that wraps an `sql.DB` connection pool. This is the type that you use as the receiver on the interface methods:
  ```go
  type ExampleModel struct {
     DB *sql.DB
  }
  ```
- Interface method implementations (`Insert`, `Get`, etc.)

After you create the data access layer, you have to import it into the `main` function and inject it as a dependency into your main application struct:

```go
type application struct {
	...
    dataAccessObj   models.ExampleModelInterface
}

func main() {
	...
    db, err := openDB(*dsn)
    if err != nil {
        errorLog.Fatal(err)
    }
    defer db.Close()


    app := &application{
        errorLog:      errorLog,
        infoLog:       infoLog,
        dataAccessObj: &models.ExampleModel{DB: db},
    }
	...
}
```

### Execute queries

Go provides the following three methods for executing SQL queries with the connection pool:
- `DB.Query()`: `SELECT` queries that return multiple rows.
- `DB.QueryRow()`: `SELECT` queries that return a single row.
- `DB.Exec()`: Queries like `INSERT` and `DELETE` that return no rows.

Create (add) data to a database with the `Exec()` function. It returns a `Result` type whose interface defines the following functions:
- `LastInsertId() (int64, error)`: verify that you added a row.
- `RowsAffected() (int64, error)`: verify which rows were updated. 

> When you add data to the database, the function should return the id of the newly created row. When you update data, the function should return an error if no rows were affected.

#### Exec() example

The `Exec()` statement use the following format. To prevent SQL injection attacks, use _placeholder_ (`?`) characters in place of values:

```go
func (m *ExampleModel) Insert(title string, content string, expires int) (int, error) {
    
	// write the statement
    stmt := `INSERT INTO snippets (title, content, created, expires)
    VALUES(?, ?, UTC_TIMESTAMP(), DATE_ADD(UTC_TIMESTAMP(), INTERVAL ? DAY))`

	// execute the statement. The first parameter is the statement you execute, 
	// followed by the placeholder values, in order.
    result, err := m.DB.Exec(stmt, title, content, expires)
    if err != nil {
        return 0, err
    }

    // Use the LastInsertId() method on the result to get the ID of our
    // newly inserted record in the snippets table.
    id, err := result.LastInsertId()
    if err != nil {
        return 0, err
    }

    // The ID returned has the type int64, so we convert it to an int type
    // before returning.
    return int(id), nil
}
```

#### QueryRow() example

A `QueryRow()` statement should use the following format to return one row from a table: 

```go
func (m *ExampleModel) Get(id int) (*Example, error) {
    // Write the SQL statement we want to execute.
    stmt := `SELECT id, title, content, created, expires FROM snippets
    WHERE expires > UTC_TIMESTAMP() AND id = ?`

    // Use the QueryRow() method on the connection pool to execute our
    // SQL statement, passing in the untrusted id variable as the value for the
    // placeholder parameter. This returns a pointer to a sql.Row object which
    // holds the result from the database.
    row := m.DB.QueryRow(stmt, id)

    // Initialize a pointer to a new zeroed Snippet struct.
    s := &Example{}

    // Use row.Scan() to copy the values from each field in sql.Row to the
    // corresponding field in the Snippet struct. Notice that the arguments
    // to row.Scan are *pointers* to the place you want to copy the data into,
    // and the number of arguments must be exactly the same as the number of
    // columns returned by your statement.
    err := row.Scan(&s.ID, &s.Title, &s.Content, &s.Created, &s.Expires)
    if err != nil {
        // If the query returns no rows, then row.Scan() will return a
        // sql.ErrNoRows error. We use the errors.Is() function check for that
        // error specifically, and return our own ErrNoRecord error
        // instead. Returning our own error type allow
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNoRecord
        } else {
            return nil, err
        }
    }

    // If everything went OK then return the Snippet object.
    return s, nil
}
```
A shorter, alternate version of the same query chains `.Scan()` to the executing query:

```go
func (m *ExampleModel) Get(id int) (*Example, error) {
    s := &Example{}
    
    err := m.DB.QueryRow("SELECT ...", id).Scan(&s.ID, &s.Title, &s.Content, &s.Created, &s.Expires)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrNoRecord
        } else {
             return nil, err
        }
    }

    return s, nil
}
```

#### Query() example

A `Query()` statement should use the following format to return a resultset consisting of one or more rows from the table:

```go
func (m *ExampleModel) Latest() ([]*Example, error) {
    // Write the SQL statement we want to execute.
    stmt := `SELECT id, title, content, created, expires FROM snippets
    WHERE expires > UTC_TIMESTAMP() ORDER BY id DESC LIMIT 10`

    // Use the Query() method on the connection pool to execute our
    // SQL statement. This returns a sql.Rows resultset containing the result of
    // our query.
    rows, err := m.DB.Query(stmt)
    if err != nil {
        return nil, err
    }

    // We defer rows.Close() to ensure the sql.Rows resultset is
    // always properly closed before the Latest() method returns. This defer
    // statement should come *after* you check for an error from the Query()
    // method. Otherwise, if Query() returns an error, you'll get a panic
    // trying to close a nil resultset.
    defer rows.Close()

    // Initialize an empty slice to hold the Snippet structs.
    examples := []*Example{}

    // Use rows.Next to iterate through the rows in the resultset. This
    // prepares the first (and then each subsequent) row to be acted on by the
    // rows.Scan() method. If iteration over all the rows completes then the
    // resultset automatically closes itself and frees-up the underlying
    // database connection.
    for rows.Next() {
        // Create a pointer to a new zeroed Example struct.
        s := &Example{}
        // Use rows.Scan() to copy the values from each field in the row to the
        // new Example that we created. Again, the arguments to row.Scan()
        // must be pointers to the place you want to copy the data into, and the
        // number of arguments must be exactly the same as the number of
        // columns returned by your statement.
        err = rows.Scan(&s.ID, &s.Title, &s.Content, &s.Created, &s.Expires)
        if err != nil {
            return nil, err
        }
        // Append it to the slice of examples.
        examples = append(examples, s)
    }

    // When the rows.Next() loop has finished we call rows.Err() to retrieve any
    // error that was encountered during the iteration. It's important to
    // call this - don't assume that a successful iteration was completed
    // over the whole resultset.
    if err = rows.Err(); err != nil {
        return nil, err
    }

    // If everything went OK then return the examples slice.
    return examples, nil
}
```

### Transactions

A _transaction_ is a wrapper around more than one database query. This means you can execute a group of queries with the same database connection. A transaction should use the following format:

```go
type ExampleModel struct {
    DB *sql.DB
}

func (m *ExampleModel) ExampleTransaction() error {
    // Calling the Begin() method on the connection pool creates a new sql.Tx
    // object, which represents the in-progress database transaction.
    tx, err := m.DB.Begin()
    if err != nil {
        return err
    }

    // Defer a call to tx.Rollback() to ensure it is always called before the 
    // function returns. If the transaction succeeds it will be already be 
    // committed by the time tx.Rollback() is called, making tx.Rollback() a 
    // no-op. Otherwise, in the event of an error, tx.Rollback() will rollback 
    // the changes before the function returns.
    defer tx.Rollback()

    // Call Exec() on the transaction, passing in your statement and any
    // parameters. It's important to notice that tx.Exec() is called on the
    // transaction object just created, NOT the connection pool. Although we're
    // using tx.Exec() here you can also use tx.Query() and tx.QueryRow() in
    // exactly the same way.
    _, err = tx.Exec("INSERT INTO ...")
    if err != nil {
        return err
    }

    // Carry out another transaction in exactly the same way.
    _, err = tx.Exec("UPDATE ...")
    if err != nil {
        return err
    }

    // If there are no errors, the statements in the transaction can be committed
    // to the database with the tx.Commit() method. 
    err = tx.Commit()
    return err
}
```

### Prepared statements

Prepared statements are beneficial for complex SQL statements that are repeated often (i.e. bulk inserts). However, they are not always efficient or necessary.

A prepared statement, is created on a particular database connection, and the `sql.Stmt` object remembers which connection. The next time you need to run the statement, that package tries to use the same connection. If it is no longer available, then `sql.Stmt` prepares another connection. This might result in too many prepared statements on too many connections.

A prepared statement shuld use the following format:

```go
// We need somewhere to store the prepared statement for the lifetime of our
// web application. A neat way is to embed in the model alongside the connection
// pool.
type ExampleModel struct {
    DB         *sql.DB
    InsertStmt *sql.Stmt
}

// Create a constructor for the model, in which we set up the prepared
// statement.
func NewExampleModel(db *sql.DB) (*ExampleModel, error) {
    // Use the Prepare method to create a new prepared statement for the
    // current connection pool. This returns a sql.Stmt object which represents
    // the prepared statement.
    insertStmt, err := db.Prepare("INSERT INTO ...")
    if err != nil {
        return nil, err
    }

    // Store it in our ExampleModel object, alongside the connection pool.
    return &ExampleModel{db, insertStmt}, nil
}

// Any methods implemented against the ExampleModel object will have access to
// the prepared statement.
func (m *ExampleModel) Insert(args...) error {
    // Notice how we call Exec directly against the prepared statement, rather
    // than against the connection pool? Prepared statements also support the
    // Query and QueryRow methods.
    _, err := m.InsertStmt.Exec(args...)

    return err
}

// In the web application's main function we will need to initialize a new
// ExampleModel struct using the constructor function.
func main() {
    db, err := sql.Open(...)
    if err != nil {
        errorLog.Fatal(err)
    }
    defer db.Close()

    // Create a new ExampleModel object, which includes the prepared statement.
    exampleModel, err := NewExampleModel(db)
    if err != nil {
       errorLog.Fatal(err)
    }

    // Defer a call to Close() on the prepared statement to ensure that it is
    // properly closed before our main function terminates.
    defer exampleModel.InsertStmt.Close()
}
```

### Type mapping

The following table describes how Go types map to SQL types:

|   Go        | SQL |
|:-----------:|:----|
| `string`    | CHAR, VARCHAR, TEXT |
| `boolean`   | BOOL |
| `int`       | INT |
| `int46`     | BIGINT |
| `float`     | DECIMAL, NUMERIC |
| `time.Time` | TIME, DATE, TIMESTAMP (`parseTime=true` DSN parameter) |