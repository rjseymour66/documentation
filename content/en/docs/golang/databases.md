---
title: "Databases"
weight: 120
description: >
  Working with Databases in Go.
---

## MySQL and Go

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

The database connection pool should be long-lived and passed among handlers, so implement it in the `main` method. Include a flag for the `dsn` so you can easily use a different data source:

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
    if err = db.Ping(); err != nil {
        return nil, err
    }
    return db, nil
}
```

### Data access layer

You want to encapsulate the code that works with MySQL in its own package. Create a new file in `root/internal/models/project.go` for your SQL data model.

The data access layer consists of the following:
- A struct that holds data for each individual object that you want to commit to the database. Generally, the struct fields should correspond to the table columns:
  ```go
  type Object struct {
    ID      int
    Title   string
    Content string
    Created time.Time
    Expires time.Time
  }
  ```
- A `*Type` struct that wraps an `sql.DB` connection pool:
  ```go
  type ExampleModel struct {
     DB *sql.DB
  }
  ```
- On the `ExampleModel`, a method for each database statement that you plan to execute. For example, `Insert`, `Get`, etc.

After you create the data access layer, you have to import it into the `main` function and inject it as a dependency into your main application struct:

```go
type application struct {
	...
    dataAccessObj *models.ExampleModel
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


## Repository pattern

The repository pattern creates a data access layer--an interface between your Go code and the database. It consists of the following structs and functions:
- Custom type that wraps a private database handle. It might include a `sync.RWMutex` to secure read and write operations.
- A constructor function that initiates and returns a database connection.
- Public methods that access the database (CRUD).

### Create the repository

Implement the [respository pattern](#repository-pattern):

1. Create the custom type:
   ```go
   type mysqlRepo struct {
	   db *sql.DB
	   sync.RWMutex
   }
   ```
2. To simplify configuration, create a configuration object with the `mysql` `Config` type. You can add properties to the `Config` type, and then use its `.FormatDSN()` function to return its data source name (DSN):
   ```go
   cfg := mysql.Config{
	   User:   os.Getenv("DBUSER"),
	   Passwd: os.Getenv("DBPASS"),
	   Net:    "tcp",
	   Addr:   "127.0.0.1:3306",
	   DBName: "database-name",
   }
   ```
3. Create a constructor that accepts the `mysql.Config` as an argument:
   ```go
   // NewMySQLRepo returns a MySQL database handle.
   func NewMySQLRepo(cfg mysql.Config) (*mysqlRepo, error) {

	   // Get the db handle
	   var err error
	   db, err := sql.Open("mysql", cfg.FormatDSN())
	   if err != nil {
		   log.Fatal(err)
		   return nil, err
	   }

	   db.SetConnMaxLifetime(time.Minute * 3)
	   db.SetMaxOpenConns(10)
	   db.SetMaxIdleConns(10)

	   // call Ping to confirm the connection
	   pingErr := db.Ping()
	   if pingErr != nil {
		   log.Fatal(pingErr)
		   return nil, pingErr
	   }

	   fmt.Println("Connected!")

	   return &mysqlRepo{
		   db: db,
	   }, nil
   }
   ```
### Create data

Create (add) data to a database with the `Exec()` function. It returns a `Result` type whose interface defines the following functions:
- `LastInsertId() (int64, error)`: verify that you added a row.
- `RowsAffected() (int64, error)`: verify which rows were updated. 

Generally, a `Create*` function should return the id of the newly created row, and an `Update*` function returns an error if no rows were affected.

The following function creates a new `Album` type and adds it to a database called "albums":

```go
func (r *mysqlRepo) CreateAlbum(alb Album) (int64, error) {
	r.Lock()
	defer r.Unlock()

	result, err := r.db.Exec("INSERT INTO album (title, artist, price) VALUES (?, ?, ?)", alb.Title, alb.Artist, alb.Price)
	if err != nil {
		return 0, fmt.Errorf("addAlbum: %v", err)
	}
	id, err := result.LastInsertId()
	if err != nil {
		return 0, fmt.Errorf("addAlbum: %v", err)
	}
	return id, nil
}
```
In the previous example, the `?` characters are _placeholders_. Placeholders prevent SQL injections.


### Retrieve data

Query the database with the database handle in the repository struct. In its most basic form, a query that retrieves data must perform the following:
- Lock the repository with a mutex.
- Execute a `Query("statement")` that returns one or more `Rows` (defer `rows.Close()`. In the statement, use a `?` character to represent values, and pass the value after the statement parameter. This protects agains SQL injection attacks. 
- Loop through the returned rows with the `.Scan(columns...)` function. You must pass as parameters pointers to each column returned in rows. 
- Check if `Query(statement)` returned an error.

#### Query multiple rows

The following function queries a database called "albums" and returns all albums by the specified artist:

```go
type Album struct {
	ID     int64
	Title  string
	Artist string
	Price  float32
}

func (r *mysqlRepo) RetrieveAlbumsByArtist(name string) ([]Album, error) {
	r.Lock()
	defer r.Unlock()

	// albums slice to hold data from returned rows
	var albums []Album

	rows, err := r.db.Query("SELECT * FROM album WHERE artist = ?", name)
	if err != nil {
		return nil, fmt.Errorf("RetrieveAlbumsByArtist %q: %v", name, err)
	}
	defer rows.Close()

	// Loop through rows, using Scan to assign column data to struct fields.
	for rows.Next() {
		var alb Album
		// pass a pointer to each colum in Album
		if err := rows.Scan(&alb.ID, &alb.Title, &alb.Artist, &alb.Price); err != nil {
			return nil, fmt.Errorf("RetrieveAlbumsByArtist %q: %v", name, err)
		}
		albums = append(albums, alb)
	}

	// Check if Query returned any errors
	if err := rows.Err(); err != nil {
		return nil, fmt.Errorf("albumsByArtist %q: %v", name, err)
	}
	return albums, nil
}
```
#### Query a single row 

A query for a single row checks errors to determine whether or not the row returned a value or another type of error:

```go
func (r *mysqlRepo) RetrieveAlbumByID(id int64) (Album, error) {
	r.Lock()
	defer r.Unlock()

	// An album to hold data from the returned row.
	var alb Album


	row := r.db.QueryRow("SELECT * FROM album WHERE id = ?", id)
	if err := row.Scan(&alb.ID, &alb.Title, &alb.Artist, &alb.Price); err != nil {
		// return if there were no matching rows
		if err == sql.ErrNoRows {
			return alb, fmt.Errorf("albumsById %d: no such album", id)
		}
		// return error if there was another error
		return alb, fmt.Errorf("albumsById %d: %v", id, err)
	}
	return alb, nil
}
```
## SQLite

SQLite databases are a single file with the .db extension. Saving the entire database in a file makes it more portable.

To start SQLite, enter `sqlite3` followed by the name of the file you want to use as a database:

```shell
$ sqlite3 dbname.db
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> 
```

Prepend commands with a period (`.`):
```sql
sqlite> .tables
sqlite> .help
sqlite> .quit

-- Create a table
sqlite> CREATE TABLE "interval" (
   ...> "id" INTEGER,
   ...> "start_time" DATETIME NOT NULL,
   ...> "planned_duration" INTEGER DEFAULT 0,
   ...> "actual_duration" INTEGER DEFAULT 0,
   ...> "category" TEXT NOT NULL,
   ...> "state" INTEGER DEFAULT 1,
   ...> PRIMARY KEY("id")
   ...> );

-- Insert data into a table
sqlite> INSERT INTO interval VALUES(NULL, date('now'),25,25,"Pomodoro",3);
sqlite> INSERT INTO interval VALUES(NULL, date('now'),25,25,"ShortBreak",3);
sqlite> INSERT INTO interval VALUES(NULL, date('now'),25,25,"LongBreak",3);

-- Select date from a table
sqlite> SELECT * FROM interval;
sqlite> SELECT * FROM interval WHERE category='Pomodoro';

-- Delete a table, and then verify
sqlite> DELETE FROM interval;
sqlite> SELECT COUNT(*) FROM interval;
0
```



### Connecting with Go

SQLite requires C bindings, so makes sure CG0 is enabled:
```shell
$ go env CGO_ENABLED
1 ## 1 means it is enabled
```

Sometimes, the driver issues warnings that do not affect functionality but can be pesky. Disable the warnings with the following command:
```shell
$ go env -w CGO_CFLAGS="-g -O2 -Wno-return-local-addr"
```
Download and install `go-sqlite3`. Install the driver to compile and cache the library. This lets you use it to build your application without requiring GCC to compile each time:
```shell
$ go get github.com/mattn/go-sqlite3
$ go install github.com/mattn/go-sqlite3
```
The following snippet is a simple example of how to connect to the database. It uses the repository pattern that requires a custom type to implement the repository interface:

```go
// custom type that implements repository interface
type dbRepo struct {
	db *sql.DB
	sync.RWMutex
}

// constructor for repo custom type
func NewSQLite3Repo(dbfile string) (*dbRepo, error) {
	db, err := sql.Open("sqlite3", dbfile)
	if err != nil {
		return nil, err
	}

	db.SetConnMaxLifetime(30 * time.Minute)
	db.SetMaxOpenConns(1)

	if err := db.Ping(); err != nil {
		return nil, err
	}

	if _, err := db.Exec(createTableInterval); err != nil {
		return nil, err
	}

	return &dbRepo{
		db: db,
	}, nil
}

```
### Constructing queries

You have to create queries as methods on the respoitory type to satisfy the interface. Generally, queries require the following steps:
- Create a lock on the SQL storage
- Prepare the statement with `.Prepare()`. Pass a statement with placeholders for the arguments (in the example, it is the `?` character). The database compiles and caches the statement so you can execute the same query multiple times with different parameters more efficiently.
  After you create the statement, make sure that you `defer x.Close()` it.
- Execute the statement with `.Exec()`, providing the arguments. This function returns a `Result` type and an error
- Use the [`Result`](https://pkg.go.dev/database/sql#Result) type to retrieve either the autoincremented id for the row (`LastInsertID`), or the number of rows affected by the query (`RowsAffected`).

```go
func (r *dbRepo) Create(i pomodoro.Interval) (int64, error) {
	// Create entry in the repository
	r.Lock()
	defer r.Unlock()

	// Prepare INSERT statement
	insStmt, err := r.db.Prepare("INSERT INTO interval VALUES(NULL,?,?,?,?,?,")
	if err != nil {
		return 0, err
	}
	defer insStmt.Close()

	// Execute INSERT statement
	res, err := insStmt.Exec(i.StartTime, i.PlannedDuration,
		i.ActualDuration, i.Category, i.State)
	if err != nil {
		return 0, err
	}

	// INSERT results
	var id int64
	if id, err = res.LastInsertId(); err != nil {
		return 0, err
	}

	return id, nil
}
```

When the query returns a row, use the `.QueryRow` function to store it in a variable. Then, you can use the `.Scan` method to convert the columns into Go values (most Go types, see the docs):

```go
// Query DB row based on ID
row := r.db.QueryRow("SELECT * FROM table WHERE id=?", id)

// Parse row into Interval struct
i := structType{}
err := row.Scan(&i.ID, &i.StartTime, &i.PlannedDuration,
    &i.ActualDuration, &i.Category, &i.State)
```

If there are multiple rows returned, you can iterate through them with the [`.Next()`](https://pkg.go.dev/database/sql#Rows.Next) function. `.Next()` returns a `bool`:

```go
// Define SELECT query for breaks
stmt := `SELECT * FROM interval WHERE category LIKE '%BREAK'
ORDER BY id DESC LIMIT ?`

// Query DB for breaks
rows, err := r.db.Query(stmt, n)
if err != nil {
    return nil, err
}
defer rows.Close()

// Parse data into slice of Interval
data := []pomodoro.Interval{}
for rows.Next() {
    i := pomodoro.Interval{}
    err = rows.Scan(&i.ID, &i.StartTime, &i.PlannedDuration,
        &i.ActualDuration, &i.Category, &i.State)
    if err != nil {
        return nil, err
    }
    data = append(data, i)
}
```

## Add a model


### Database table

Create the database table that stores the model information:

```shell
$ mysql -D <database-name> -u root -p$MYSQL_ROOT_PASS
```
```sql
> CREATE TABLE users (
      id INTEGER NOT NULL PRIMARY KEY AUTO_INCREMENT,
      name VARCHAR(255) NOT NULL,
      email VARCHAR(255) NOT NULL,
      hashed_password CHAR(60) NOT NULL,
      created DATETIME NOT NULL
  );

ALTER TABLE users ADD CONSTRAINT users_uc_email UNIQUE (email);
```

### Model skeleton

Create the model in `project/internal/models/`_`model-name.go`_. The model consists of the following:
- A type whose fields map to the database table
- A model type that wraps a database connection pool.
- Methods that perform CRUD operations

For example, the following code creates a users model that represents users that log in to a web application:
```go
type User struct {
    ID             int
    Name           string
    Email          string
    HashedPassword []byte
    Created        time.Time
}

// Define a new UserModel type which wraps a database connection pool.
type UserModel struct {
    DB *sql.DB
}

// We'll use the Insert method to add a new record to the "users" table.
func (m *UserModel) Insert(name, email, password string) error {
    return nil
}

// We'll use the Authenticate method to verify whether a user exists with
// the provided email address and password. This will return the relevant
// user ID if they do.
func (m *UserModel) Authenticate(email, password string) (int, error) {
    return 0, nil
}

// We'll use the Exists method to check if a user exists with a specific ID.
func (m *UserModel) Exists(id int) (bool, error) {
    return false, nil
}
```

### Errors

In `project/internal/models/errors.go`_, add any errors that the new model might return:

```go
var (
    ...
    // Add a new ErrInvalidCredentials error. We'll use this later if a user
    // tries to login with an incorrect email address or password.
    ErrInvalidCredentials = errors.New("models: invalid credentials")

    // Add a new ErrDuplicateEmail error. We'll use this later if a user
    // tries to signup with an email address that's already in use.
    ErrDuplicateEmail = errors.New("models: duplicate email")
)
```

### Add the model in main

Finally, wire the model into the application in `main.go`. This includes adding it to the `application` struct and giving it a database connection:

```go
// Add a new users field to the application struct.
type application struct {
    ...
    users *models.UserModel
    ...
}

func main() {
    ...
    // Initialize a models.UserModel instance and add it to the application
    // dependencies.
    app := &application{
        ...
        users: &models.UserModel{DB: db},
        ...
    }
    ...
}
```

## Validating unique fields

Some values must be unique among other values stored for an application. For example, an email address in a web application:

```sql
> CREATE TABLE users (
      email VARCHAR(255) NOT NULL,
  );

ALTER TABLE users ADD CONSTRAINT users_uc_email UNIQUE (email);
```

The preceding SQL code creates an email field and places a constraint on it that requires it be unique among all other email fields within the database.

To check this in the application, you have to check that the database returns the correct error, then return a custom error that describes what occurred:

```go
func (app *Application) myHandler() {

	_, err = m.DB.Exec(stmt, name, email, string(hashedPassword))
	if err != nil {
		var mySQLError *mysql.MySQLError
		if errors.As(err, &mySQLError) {
			if mySQLError.Number == 1062 && strings.Contains(mySQLError.Message, "users_uc_email") {
				return ErrDuplicateEmail
			}
		}
		return err
	}
}
```