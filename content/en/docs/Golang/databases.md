---
title: "Databases"
weight: 120
description: >
  Working with Databases in Go.
---

## Repository pattern

The repository pattern creates an interface between your Go code and the database. It consists of the following structs and functions:
- Custom type that wraps a private database handle. It might include a `sync.RWMutex` to secure read and write operations.
- A constructor function that initiates and returns a database connection.
- Public methods that access the database (CRUD).

## MySQL

[golang/go SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers)

### MySQL service

The MySQL service must be running to access a database. The following commands check the service, restart the service, and then configure the service to start each time the computer boots:

```shell
# check status
$ sudo systemctl status mysql

# restart service
$ sudo systemctl start mysql

# start at boot
$ sudo systemctl enable mysql
```

### Access a MySQL database

Access mysql with the mysql CLI utility:

```shell
$ mysql -u <username> -p
# optional database name
$ mysql -u <username> -p <database>
```

The `-u` flag indicates the MySQL username, and the `-p` flag makes the CLI utility prompt you for your password after you enter the command. You can also provide the password to the command by entering it directly after the `-p` flag, with no spaces in between. 
> Use environment variables if you are passing the password to the console.

For details about all connection options, refer to the [documentation](https://dev.mysql.com/doc/refman/8.0/en/connecting.html).

After you access the mysql server, you can view all databases:

```sql
mysql> create database golang;
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| golang             |
| grafana            |
| information_schema |
| mysql              |
| performance_schema |
| sakila             |
| sys                |
+--------------------+
7 rows in set (0.00 sec)

mysql> use golang;
Database changed
```
### User administration

To view your current database:

```mysql
mysql> SELECT DATABASE();
+------------+
| DATABASE() |
+------------+
| golang     |
+------------+
1 row in set (0.00 sec)

```

To view details about your current session, enter `status`:

```mysql
> status;
--------------
mysql  Ver 8.0.33-0ubuntu0.22.04.2 for Linux on x86_64 ((Ubuntu))

Connection id:          24
Current database:
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         8.0.33-0ubuntu0.22.04.2 (Ubuntu)
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8mb4
Db     characterset:    utf8mb4
Client characterset:    utf8mb4
Conn.  characterset:    utf8mb4
UNIX socket:            /var/run/mysqld/mysqld.sock
Binary data as:         Hexadecimal
Uptime:                 2 days 11 hours 33 min 56 sec

Threads: 2  Questions: 30  Slow queries: 0  Opens: 147  Flush tables: 3  Open tables: 66  Queries per second avg: 0.000
--------------

```

For comprehensive instructions on granting privileges, refer to [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-create-a-new-user-and-grant-permissions-in-mysql).

To create a user, enter the following:

```sql
> CREATE USER '<username>'@'localhost' IDENTIFIED BY '<password>';
```
To grant that user permissions to a specific table:

```sql
> GRANT PRIVILEGE ON database.table TO '<username>'@'host';
```

For example:

```sql
-- grant specific privileges
GRANT CREATE, ALTER, DROP, INSERT, UPDATE, DELETE, SELECT, REFERENCES, RELOAD on *.* TO '<username>'@'localhost' WITH GRANT OPTION;

-- grant all privileges
GRANT ALL PRIVILEGES ON *.* TO '<username>'@'localhost' WITH GRANT OPTION;
```

As a best practice, flush the grant tables to ensure that the new privileges are put into effect:

```sql
> FLUSH PRIVILEGES;
```

To confirm that privileges were granted:

```sql
> SHOW GRANTS FOR '<username>'@'host';
```

### Initialize the database

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


You can run SQL scripts to easily execute DDL statements:

```shell
$ mysql> source path/to/script.sql
```
Example `script.sql`:
```sql
-- This is a comment
CREATE TABLE IF NOT EXISTS "interval" (
		"id" INTEGER,
		"start_time" DATETIME NOT NULL,
		"planned_duration" INTEGER DEFAULT 0,
		"actual_duration" INTEGER DEFAULT 0,
		"category" TEXT NOT NULL,
		"state" INTEGER DEFAULT 1,
		PRIMARY KEY("id")
	);
```
Alternately, you can model SQL statements as a `const`:

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

## MySQL and Go

This section borrows heavily from [Tutorial: Accessing a relational database](https://go.dev/doc/tutorial/database-access). For information about transactions, query cancellation, and connection pools, see [Accessing a relational database](https://go.dev/doc/database/).

### Import the driver

1. Go to [SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers) and locate the [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql/) link.
2. In the go file that runs SQL code, import the driver. Because you are not using the driver directly, import it with the `blank identifier`:
   ```go
   package main

   import "_ github.com/go-sql-driver/mysql"
   ```
   If you have not installed the package dependency, run `go get` in the shell:
   ```shell
   $ go get -u github.com/go-sql-driver/mysql
   ```

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
