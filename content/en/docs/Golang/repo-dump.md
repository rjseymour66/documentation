---
title: "Repo dump"
weight: 30
description: >
  Repo dump that needs reorg.
---

### Find a home...

- deferred function calls are not executed when `os.Exit()` is called.



### Basics

### Formatting verbs

[Formatting verbs](https://pkg.go.dev/fmt#hdr-Printing)

### Idioms

Use `if found...` to determine if an element exists in a list:
```go
// returs a bool and int
func (hl *HostsList) search(host string) (bool, int) {
    ...
}

// found is a boolean. You can use a truncated syntax:
func (hl *HostsList) Add(host string) error {
	if found, _ := hl.search(host); found {
		return fmt.Errorf("%w: %s", ErrExists, host)
	}
    ...
}
```

The comma, ok idiom checks whether a value is in a map (??????)


## iota

The `iota` operator creates a set of constants that increase by 1 for each line. This is helpful to track state or lifecycle stages.

To create an iota constant, create a constant variable and assign the first value `= iota`. For example:

```go
const (
	StateNotStarted = iota
	StateRunning
	StatePaused
	StateDone
	StateCancelled
)
```



### run() function in main()

If you use the `main()` function to run all of the code, it is difficult to create integration tests. To fix this, break the `main()` function into smaller functions that you can test independently. Use the `run()` function as a coordinating function for the code that needs to run in `main()`. So, the `main()` function parses command line flags and calls the `run()` function.

When you use the `run()` method strategy, you write unit tests for all the individual functions within `run()`, and you write an integration test for `run()`.

### Print statements

```go
fmt.Errorf("Custom formatted error messages: %s", optionalErr)
fmt.Fprintf(writer, "Writes this formatted string to the writer: %s", text)
fStr := fmt.Sprintf("Returns a formatted string: %s", text)
fmt.Fprintln(io.Writer, c ...content) // writes to writer and appends newline
```
If a function returns a `string`, you can return `fmt.Sprintf("Return this string")`:
```go
func (s *stepErr) Error() string {
	return fmt.Sprintf("Step: %q: %s: Cause: %v", s.step, s.msg, s.cause)
}

```

### Equality

```go
bytes.Equal(bSlice1, bSlice2)
```

### Environment variables

Getting and checking if an environment variable is set:
```go
if os.Getenv("ENV_VAR_NAME") != "" {
    varName = os.Getenv("ENV_VAR_NAME")
}
```


### Interfaces

When possible, use interfaces as function arguments instead of concrete types to increase flexibility.

```go
io.Reader // any go type that you can read data from
io.Writer // any go type that you can write to
fmt.Stringer // returns a string. Similar to .toString() in Java.
```
The `Stringer` interface allows you to use the type directly in print functions. For example:
```go
func (r *Receiver) String() string {
    // return a string
}

fmt.Print(*r)
```
### io.Writer

Commonly named `w` or `out`. Examples of `io.Writer`:
- os.Stdout
- bytes.Buffer (implements `io.Writer` (and `io.Reader`) as a pointer receiver, so use `&`)
- files (type os.File implements `io.Writer`)
- gzip.Writer

> Use a file or `os.Stdout` in the program, and `bytes.Buffer` when testing.

GZIP writer example:
```go
// open or create file at targetPath
// rwxrxrx
out, err := os.OpenFile(targetPath, os.O_RDWR|os.O_CREATE, 0755)
if err != nil {
    return err
}
defer out.Close()

// open file w contents to zip
in, err := os.Open(path)
if err != nil {
    return err
}
defer in.Close()

// create new zip writer
zw := gzip.NewWriter(out)
zw.Name = filepath.Base(path)

// copy contents
if _, err = io.Copy(zw, in); err != nil {
    return err
}

// close zip writer
if err := zw.Close(); err != nil {
    return err
}
// returns an error on fail
return out.Close()
```

### Methods

### Constructors

Go doesn't have constructor methods, but it is a good idea to create them so that users instantiate structs correctly.

Always prepend constructor names with `[Nn]ew*`. For example, the following constructor creates a new step in a processing pipeline:

```go
func newStep(name, exe, message, proj string, args []string) step {
	return step{
		name:    name,
		exe:     exe,
		message: message,
		args:    args,
		proj:    proj,
	}
}
```

When there are too many parameters that you want to pass to a function, create a `config` struct:
```go
type config struct {
    // value type
    // ...
}
```
When you create a `config` object, assign each field the value of a CLI flag.

### Embedded types, extending types

Embedding types makes all the fields and methods of one type available in the to the embedding type.

You can embed an existing type by embedding it in a new type. For example, if you want to implement a new method on an existing type, you can embed it without adding any fields:

```go
// extends the step type
type exceptionStep struct {
	step
}
```

Because you did not add any new fields, you can use the embedded type's constructor:
```go
// original type
func newStep(name, exe, message, proj string, args []string) step {
	return step{
		name:    name,
		exe:     exe,
		message: message,
		proj:    proj,
		args:    args,
	}
}

// extened type
type timeoutStep struct {
	step
	timeout time.Duration
}


// extended type constructor
func newTimeoutStep(name, exe, message, proj string, args []string, timeout time.Duration) timeoutStep {
	s := timeoutStep{}
    // embedded type constructor
	s.step = newStep(name, exe, message, proj, args)

	s.timeout = timeout
	if s.timeout == 0 {
		s.timeout = 30 * time.Second
	}

	return s
}
```

### Value recievers

Use a value receiver when the method:
- mutates the receiver
- is too large to reasonably pass in memory

Inside the method body, dereference the receiver using the `*` operator to access and mutate the value of the receiver. Otherwise, you are operating on the address value:

```go
func (r *Receiver) MethodName(param type) {
    *r = // do something else
}
```
> **Best practice**: The method set of a single type should use the same receiver type. If the method does not mutate the receiver, you can assign the pointer receiver to a value at the start of the method.

### Variadic functions

Represents zero or more values of a type. Precede the type with three periods (`...`). For example:

```go
func concatInput(args ...string) {
    return strings.Join(args, " "), nil
}
```


### Data structures and formats

### Structs

Create a zero-value struct:
```go
type person struct {
    name    string
    age     int
}

john := person{}
```


### Files

If you need to create a temp file:
```go
temp, err := os.CreateTemp("", "pattern*.extension")
```
- First parameter is the directory that you want to create the temporary file in. If it is left blank, it uses the `/tmp` directory.
- The second parameter defines the file name. Use a `*` character to tell the program to generate a random number to make the name unique.

Get the name of the file:
```go
name := fileName.Name()
```

### Examining file metadata

Use [os.FileInfo](https://pkg.go.dev/io/fs#FileInfo) to examine file metadata. To return the `FileInfo` file attributes for a file, use `os.Stat(filename)`:
```go
info, err := os.Stat(fileName)
```

### Opening a file

Open a file with `os.OpenFile()`:
```go
f, err = os.OpenFile(*logFile, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644)
if err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
}
// always defer the close
defer f.Close()
```

### Reading data

### Reading from a file

Read data from a file with the `os` package. `ReadFile` reads the contents of a file and returns a `nil` error:
```go
os.ReadFile(filename)
```

Extract the last element in a filepath--generally, the file--with the filepath.Base() function:
```go
filename := filepath.Base("/path/to/home.html")
// filename == home.html
```

### Scanner for lines and words

The Scanner type accepts an `io.Reader` and reads data that is delimited by spaces or new lines.

By default, it reads lines, but you can configure it to read words:

```go
scanner := bufio.NewScanner(r)
// scan words
scanner.Split(bufio.ScanWords)
```
Use the `.Scan()` function in a for loop to read lines or tokens, depending on the `.Split()` configuration:
```go
for s.Scan() {
    // if non-EOF error
    if err := s.Err(); err != nil {
        return "", err
    }
    file = s.Text()

    if len(s.Text()) == 0 {
        return "", fmt.Errorf("File cannot be blank")
    }
}
```

To find the number of bytes in each scanned token:
```go
// scan words
scanner.Split(bufio.ScanWords)

byteLength := 0
for scanner.Scan() {
    byteLength += len(scanner.Bytes())    
}
```
### Reading CSV data

Create a `.NewReader()` the same way that you create a `.NewScanner()` and read with the following methods:
- `.Read()` returns a `[]string` that represents a row.
- `.ReadAll()` returns a `[][]string`, where each slice is a row in the CSV file. 

Below is an example that reads an entire CSV file and tries to convert the values to `float64`:
```go
func csv2float(r io.Reader, column int) ([]float64, error) {
	// Create the CSV Reader used to read in data from CSV files
	cr := csv.NewReader(r)
	// adjusting column arg for 0-based index
	column--
	// Read in all CSV data
	allData, err := cr.ReadAll()
	if err != nil {
		return nil, fmt.Errorf("Cannot read data from file: %w", err)
	}

	var data []float64
	/*
		convert [][]string to [][]float64
	*/

	// loop through all records
	for i, row := range allData {
		// skip the first row that contains the column headers
		if i == 0 {
			continue
		}
		// Checking number of cols in CSV file to verify flag value
		if len(row) <= column {
			// file does not have that many columns
			return nil,
				fmt.Errorf("%w: File has only %d columns", ErrInvalidColumn, len(row))
		}
		// Try to convert data read into a float number
		v, err := strconv.ParseFloat(row[column], 64)
		if err != nil {
			return nil, fmt.Errorf("%w: %s", ErrNotNumber, err)
		}

		data = append(data, v)
	}
	return data, nil
}
```


### Writing data

### Writing to a file

Write data to a file with the `os` package. `WriteFile` writes to an existing file or creates one, if necessary:
```go
os.WriteFile(filename, dataToWrite, perms)
```
> **Linux permissions**: Set Linux file permissions for the file owner, group, and all other users (`owner-group-others`). The permission options are read, write, and execute. Use octal values to set permssions:  
  read: 4  
  write: 2  
  execute: 1  

When you assign permissions in an programming language, you have to tell the program that you are using the octal base. You do this by beginning the number with a `0`. So, `0644` permissions means that the file owner has read and write permissions, and the group and all other users have read permissions.

### Buffers and bytes

Many functions return `[]byte`, so you might have to fill a buffer with data to return.

The `bytes` package contains two types: `Buffer` and `Reader`. The `Buffer` returns a variable size buffer to read and write data. It provides `Write*` methods for `strings`, `runes`, `bytes`, etc.:

```go
func byteStuff() []bytes {
    // compose the page using a buffer of bytes to write to a file
    var buffer bytes.Buffer

    // write html to bytes buffer
    buffer.WriteString("The first string")
    buffer.Write([]byte{'T', 'h', 'e', 's', 'e', 'c', 'o', 'n', 'd', 's', 't', 'r', 'i', 'n', 'g'})
    buffer.WriteString("The last string")

    // return []bytes
    return buffer.Bytes()
}
```

### tabWriter

`tabwriter.Writer` writes tabulated data with formatted columns with consistent widths using tabs (`\t`).

https://pkg.go.dev/text/tabwriter#pkg-overview

## Contexts

If you are executing commands that must communicate over a network, you should use a timeout. To create a timeout, use `context.WithTimeout()`.

`context.WithTimeout()` creates a context from the parent context and a timeout value. To create a new, empty context, pass the `context.Background()` as the parent:

```go
...
ctx, cancel := context.WithTimeout(context.Background(), s.timeout)
defer cancel()
...
```
`context.WithTimeout()` returns the context and a `cancel` function to cancel the context when it is no longer required. You should defer the `cancel` function.

If the context expires because of the timeout, you can check the context's `.Err()' function for a `DeadlineExceeded` error:

```go
	cmd := exec.CommandContext(ctx, s.exe, s.args...)
	cmd.Dir = s.proj

	if err := cmd.Run(); err != nil {
		if ctx.Err() == context.DeadlineExceeded {
			return ...
			}
		}
        ...
```

### Filesystem

The `filepath` library creates cross-platform filepaths--they compile correctly for each supported OS.

Get the relative directory path between a root and target path:
```go
relDir, err := filepath.Rel(root, filepath.Dir(path))
if err ...
```



### Time

Get the current time:
```go
current = time.Now()
```
Get the zero value for time.Time with an empty struct:
```go
zeroVal = time.Time{}
```
Format the time with a constant. Then, you can pass `timeFormat` to the `.Format()` method of a time.Time() type: For example:

```go
const timeFormat = "Jan/02 @15:04"
task.CreatedAt.Format(timeFormat)
```

Create a ticker when you want to do something at a regular interval:
```go

```

### Building commands with os/exec

### Find the OS

Go can compile a binary for any OS, so you should check the `runtime.GOOS` constant to determine the OS.

Use the `Cmd` type to build external commands to execute in your program. The `exec.Command()` function takes the name of the executable program as the first argument and zero or more arguments that will be passed to the executable during execution:
```go
// define the arguments for the command
args := []string{"build", ".", "errors"}
// create the command with the executable and args
cmd := exec.Command("go", args...)
// set the directory for the external command exection
cmd.Dir = proj
// execute the command with .Run()
if err cmd.Run(); err != nil {
    return fmt.Errorf("'go build' failed: %s", err)
}
```

### Example 1

Create a command that adds a task to a todo application through STDIN. For brevity, this example omits error checking in some places:
```go
/* 1 */ task := "This is the task"
/* 2 */ workingDir := os.Getwd() // check error
/* 3 */ cmdPath := filepath.Join(workingDir, appName)
/* 4 */ cmd := exec.Command(cmdPath, "-add")
/* 5 */ cmdStdIn, err := cmd.StdinPipe()
/* 6 */
io.WriteString(cmdStdIn, task)
cmdStdIn.Close()

/* 7 */
if err := cmd.Run(); err != nil {
    t.Fatal(err)
}
// Alt 7: you could run cmd.CombinedOutput() to get the STDOUT and STDERR
out, err := cmd.CombinedOutput()
// error checking
// https://pkg.go.dev/os/exec@go1.19.3#Cmd.CombinedOutput
```
In the preceding example:
1. Create the task string
2. Get the current working directory from root
3. Create a command consisting absolute path and add the name of the binary
4. `cmd` is a command struct that executes the command with the provided arguments
5. Connect a pipe to the command's STDIN. The command now looks like this:
   `| /path/to/appName -add`
6. Write the task to STDIN
7. Run the command

### Example 2

Create a slice literal to store the parameters:
```go
params := []string{}
params = append(params, arg1)
params = append(params, arg2)
// expand slice values into function
exec.Command(/path/, params...)
```

### Useful exec. methods

```go
exec.LookPath(fileName string) // returns location of fileName in PATH or error
```
### Mocking a command during tests

```go
func mockCmd(exe string, args ...string) *exec.Cmd {
	cs := []string{"-test.run=TestHelperProcess"}
	cs = append(cs, exe)
	cs = append(cs, args...)
	cmd := exec.Command(os.Args[0], cs...)
	cmd.Env = []string{"GO_WANT_HELPER_PROCESS=1"}
	return cmd
}
```
## Logging

Use logs to provide feedback for background processes. To create a logger, you need to create:
- [*log.logger](https://pkg.go.dev/log#Logger) type
- Logging destination ([w io.Writer](#interfaces))

By default, Go's `log` library sends messages to STDERR, but you can configure it to write to a file. It adds the date and time to each log entry, and you can add a prefix to the string to help searchability
```go
l := log.New(io.Writer, "LOGGER PREFIX: ", log.LstdFlags)
```
log.LstdFlags uses the default log flags, such as date and time.


### Templates

Templates can write dynamic webpages, config files, emails, etc. These are the general steps:

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





## Signals

Signals communicate events among running processes, such as SIGINT, the interrupt signal.

When a program receives an interrupt signal, it stops processing immediately. This can lead to data loss and issues with resources, so you have to make sure that the program exits cleanly.

To handle signals, complete the following:
- create a channel to receive a signal
- pass the channel to the `signal.Notify` function, followed by a list of the signals that the application should listen for
- Wrap the main part of the function in a goroutine so it can run concurrently with `signal.Notify`
- Create an infinite for loop with a select statement that handles the various channels. The channel that handles the signal should call `signal.Stop(signalChannel)` to stop relaying any incoming signals to the channel.

```go
sig := make(chan os.Signal, 1)
done := make(chan struct{})

signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)

// goroutine that runs concurrently with signal.Notify
go func() {
    // do work
    close(done)
}()

for {
    select {
    case rec := <-sig:
        signal.Stop(sig)
        return fmt.Errorf("%s: Exiting: %w", rec, ErrSignal)
    case <-done:
        return nil
    }
}
```

## Sorting

The `sort` package provides functions that sort 


## Network connections

## Conditional builds

Add build tags to control what files are included in a build:

```go
/* do not include these files: */

//go:build !inmemory && !containers
// +build !inmemory,!containers

/* include these files: */

//go:build inmemory || containers
// +build inmemory containers

```

To verify which files Go will include in a build according to the tags, use `go list`. Use the `-f` option:

```shell
## list all files
$ go list -f '{{ .GoFiles }}' ./...
[main.go]
[app.go buttons.go grid.go notification.go summaryWidgets.go widgets.go]
[reposqlite.go root.go]
[interval.go summary.go]
[sqlite3.go]

## list files with inmemory build tag
$ go list -tags=inmemory -f '{{ .GoFiles }}' ./...
[main.go]
[app.go buttons.go grid.go notification.go summaryWidgets.go widgets.go]
[repoinmemory.go root.go]
[interval.go summary.go]
[inMemory.go]

## list files with containers build tag
$ go list -tags=containers -f '{{ .GoFiles }}' ./...
[main.go]
[app.go buttons.go grid.go notification_stub.go summaryWidgets.go widgets.go]
[reposqlite.go root.go]
[interval.go summary.go]
[inMemory.go]
```

## Compiling for different architectures

Use `go env GO_VAR` to return the environment variable value. For example, the following commands describe the operating system and architecture of the machine:

```shell
$ go env GOOS
linux

$ go env GOARCH
amd64
```
Set `GOOS` and `GOARCH` with `go build` to compile binaries for different operating systems and architectures:

```shell
$ GOOS=windows GOARCH=amd64 go build
$ file pomo.exe 
pomo.exe: PE32+ executable (console) x86-64 (stripped to external PDB), for MS Windows
```

Create a build script to automate builds for different operating systems and architectures. Add them in the `/scripts` directory:

```bash
## cross_build.sh

#!/bin/bash
OSLIST="linux windows darwin"
ARCHLIST="amd64 arm arm64"

for os in ${OSLIST}; do
    for arch in ${ARCHLIST}; do
        if [[ "$os/$arch" =~ ^(windows/arm64|darwin/arm)$ ]]; then continue; fi

        echo Building binary for $os $arch
        mkdir -p releases/${os}/${arch}
        CGO_ENABLED=0 GOOS=$os GOARCH=$arch go build -tags=inmemory \
            -o releases/${os}/${arch}
    done
done
```
### Dynamically and statically linked binaries

By default, Go binaries are dynamically linked, which means that the binary loads any required shared libraries dynamically at run time. Set CGO_ENABLED to 0 to build a binary for a system that supports only statically shared libraries. Setting this value with the build command does not impact its go env variable value:

```shell
$ go env CGO_ENABLED
1

$ file pomo
pomo: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=05dce90786cee01b2444961ff030b08e2e9f6648, with debug_info, not stripped

$ go env CGO_ENABLED
1

$ CGO_ENABLED=0 go build
$ file pomo
pomo: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, with debug_info, not stripped

$ go env CGO_ENABLED
1

```

