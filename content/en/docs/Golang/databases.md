---
title: "Databases"
weight: 120
description: >
  Working with Databases in Go.
---


## Repository pattern

The repository pattern is an interface to access data storage while decoupling your business login from your storage implementation.

The Repository Pattern requires two components:
1. An interface that specifies all the methods that a given type must implement to qualify as a repository for the application. Define this interface in the same package that you use it.
2. A custom type that implements that interface working as the repository. Make the type and its fields private so callers can access it only through the interface methods.

For example, the following 

### Connecting to a database

Connect to a SQL database using the [`database/sql`](https://pkg.go.dev/database/sql). You also need access to the database driver, which works with the `database/sql` package to implemet the details to interface with the database engine.

When you use a database with an application, you want to initialize the db with the required structure. For large applications, you might need version controlled scripts or migration files to initialize your db. For smaller apps, create a table initialization statement within a `const`:

```go
const (
	createTableName string = `CREATE TABLE IF NOT EXISTS "interval" (
		"id" INTEGER,
		"start_time" DATETIME NOT NULL,
		"planned_duration" INTEGER DEFAULT 0,
		"actual_duration" INTEGER DEFAULT 0,
		"category" TEXT NOT NULL,
		"state" INTEGER DEFAULT 1,
		PRIMARY KEY("id")
	);`
)
```
The SQLite driver automatically handles conversion between Go types and SQL types.

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
