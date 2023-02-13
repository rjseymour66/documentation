---
title: "Repo dump"
weight: 20
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

### Project structure

```
.
├── cmd 
│   └── todo
│       ├── main.go         // config, parse, switch {} flags
│       └── main_test.go    // integration tests (user interaction)
├── go.mod
├── todo.go                 // API logic for flags
└── todo_test.go            // unit tests

```


`/internal` directory is special because other projects cannot import anything  in this directory. This is a good place for domain code.
If an entity needs to be accessible to all domain code, place its file in the `/internal` directory. Its package name is the project package.
Each subdirectory in `/internal` is a domain.

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


### Concurrency

Go provides two strategies that maintain data integrity when you are writing concurrent programs:
- Locks, such as Mutex
- goroutines and channels

### Mutexes

`RLock()` blocks and waits if the associated object is locked for writing. This provides safe concurrent read and write operations while allowing multiple reads to improve performance.

### Channels

Channels allow goroutines to communicate with each other.

```go
// create a channel with make
ch := make(chan type)
// Use an empty struct to create a channel for done. done channels
// only signal that processing is complete, and an empty struct does not allocate
// any memory
done := make(chan struct{})
```

### WaitGroups

A WaitGroup is a mechanism that coordinates the goroutine execution. When you create a goroutine, add 1 to the WaitGroup. When a goroutine finishes, subtract 1 from the WaitGroup. Use `Wait()` to wait until all goroutines are finished so you can complete execution.

```go
wg := sync.WaitGroup{}
```

### Scheduling contention and worker queues

This is when you create too many goroutines and they compete for work. The answer is to use worker queues.

When using worker queues, you create one goroutine per available CPU, and have another goroutine send jobs to be executed by the workers. So, the CPUs are the workers.

Use `runtime.NumCPU()` to determine the number of available CPUs:
```go

```

### Goroutines

Goroutines are usually anonymous functions that follow the keyword `go`, so that they run independently of the `main()` function. Because they run independently of the `main()` function, go uses `WaitGroups`, a mechanism that blocks the `main()` method until all goroutines complete.

The following worker queue example reads numbers from a file, and converts them from type string to float64. The containing function has this signature:

```go
func process(filenames []string, operation string, column int, out io.Writer)
```

When using worker queues, you create one goroutine per available CPU, and have another goroutine send jobs to be executed by the workers. So, the CPUs are the workers.

First, create your channels. The channels allow goroutines to communicate without using locking mechanisms, such as mutexes.

Create the following channels:
- resultCh for the processed float64
- errCh for errors
- doneCh as the signal channel, a Go idiom. The done channel is of type empty struct because its only purpose is to let us know when the work is complete. Use an empty struct because it does not allocate memory
- filesCh is the queue. Add files for processing to this channel. The worker gorouties take files from this channel and process them.

```go
    resCh := make(chan []float64)
    errCh    := make(chan error)
    doneCh   := make(chan struct{})
    filesCh  := make(chan string)
```
Create the WaitGroup:

```go
    wg := sync.WaitGroup{}
```
Create a goroutine that sends files into the filesCh queue. This function runs independently of the `main()` function, but it is not doing any work in the queue. So, you don't have to increase or decrease the wg counter:
```go
    go func() {
        // close the channel at the end because there is no more work to do
        defer close(fileCh)
        for _, fname := range filenames {
            filesCh <- fname
        }
    }()
```
Now, process the work in the queue. Create a loop that creates a goroutine for each available CPU (worker) on the host machine with `runtime.NumCPU()`. Each loop adds 1 to the WaitGroup counter. So there is 1 WaitGroup per goroutine, and 1 goroutine per CPU.

Each goroutine processes files in `filesCh` and either adds the processed data to the `resCh` or adds the error to the `errCh`. When there are no more files in `fileCh`, the goroutine completes and decrements the WaitGroup counter by 1.

```go
for i := 0; i < runtime.NumCPU(); i++ {
    // During each iteration, add a goroutine to the WaitGroup{}
    wg.Add(1)
    go func() {
        // decrement the wg counter
        defer wg.Done()
        // for range on a channel.
        // for every item in this channel, do {...}
        for fname := range filesCh {
            // Open the file for reading
            f, err := os.Open(fname)
            if err != nil {
                // Send errors into the error channel
                errCh <- fmt.Errorf("Cannot open file: %w", err)
                return
            }

            // Parse the CSV into a slice of float64 numbers
            data, err := csv2float(f, column)
            if err != nil {
                errCh <- err
            }

            if err := f.Close(); err != nil {
                errCh <- err
            }
            // if the string was converted to float64, send it to 
            // the results channel
            resCh <- data
        }
    }()
}
```

The work is not complete until the `doneCh` sends a signal. Add the `wg.Wait()` function to block until all goroutines are completed, then close `doneCh`:
```go
    go func() {
        // block until the WaitGroup counter is 0
        wg.Wait()
        // close() indicates that no more values will be sent
        close(doneCh)
    }()
```

Now, all of the goroutines are completed, and you are back in the `main()` function (the main goroutine). Coordinate the channels with the `select` statement:

The select statement is similar to a switch statement. It blocks execution of the program until something happens with one of the channels. This statement:
- returns any error and breaks out of the loop
- adds converted data to the consolidate channel
- writes the data when the work is done

```go
    // create an infinte loop to accept values from the channels
	for {
		select {
		case err := <-errCh:
			return err
		case data := <-resCh:
			consolidate = append(consolidate, data...)
		case <-doneCh:
			_, err := fmt.Fprintln(out, opFunc(consolidate))
			return err
		}
	}
```

### Locks

The [Locker interface](https://pkg.go.dev/sync@go1.19.4#Locker) has methods that lock and unlock an object. Use this when you want to prevent concurrent access to an object during operations.

`&sync.Mutex` implements the Locker interface:
```go
mu := &sync.Mutex{}
```


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

## Containerization

Containers package your application and all the required dependencies using a standard image format, and they run the application in isolation from other processes running on the same system.

A Dockerfile is a recipe for how to create an image. Go is well-suited for containerization because it creates a single binary that does not require additional runtimes or dependencies.

### Build options

CGO_ENABLED=0
: Enables statically linked binaries to make the application more portable. You can use the go binary with images that do not support shared libraries.

GOOS=linux
: Containers run on Linux, so set this to enable repeatable bulds even with building the application on a different platform.

-ldflags="-s-w"
: `-ldflags="-s -w"` lets you specify additional linker options that go build uses at the link stage. `-s -w` strips the binary of debugging symbols to decrease its size.
: To see all linker options, use `go tool link`.

`-tags=containers`
: Build files that have the `// +build containers` directive at the top.

Execute `go build` with these options and compare the binaries:

```shell
## optimized for containers
$ CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -tags=containers
$ ls -lh pomo
-rwxrwxr-x 1 ryanseymour ryanseymour 7.2M Jan 12 23:59 pomo
$ file pomo
pomo: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, stripped

## standard build
$ go build
$ ls -lh pomo
-rwxrwxr-x 1 ryanseymour ryanseymour 13M Jan 13 00:01 pomo
$ file pomo
pomo: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=d921fee20837a42fbe52f957a9fe6643393711cf, with debug_info, not stripped

```

### Dockerfile

After you build a binary that is optimized for containers, create a Dockerfile to build the image. The following Dockerfile creates an image with a regular user `pomo`, and copies the binary into the `/app` directory:

```Dockerfile
FROM alpine:latest 
RUN mkdir /app && adduser -h /app -D pomo 
WORKDIR /app 
COPY --chown=pomo /pomo/pomo .
CMD ["/app/pomo"]
```

Next, build the image:
```shell
$ docker build -t pomo/pomo:latest -f containers/Dockerfile .
Sending build context to Docker daemon  62.86MB
Step 1/5 : FROM alpine:latest
latest: Pulling from library/alpine
8921db27df28: Pull complete 
Digest: sha256:f271e74b17ced29b915d351685fd4644785c6d1559dd1f2d4189a5e851ef753a
Status: Downloaded newer image for alpine:latest
 ---> 042a816809aa
Step 2/5 : RUN mkdir /app && adduser -h /app -D pomo
 ---> Running in aa2c45fe24e7
Removing intermediate container aa2c45fe24e7
 ---> 8315b0cd3500
Step 3/5 : WORKDIR /app
 ---> Running in fd9e0b3fcedc
Removing intermediate container fd9e0b3fcedc
 ---> 1c29d6e70641
Step 4/5 : COPY --chown=pomo /pomo/pomo .
 ---> f9cd8113caa2
Step 5/5 : CMD ["/app/pomo"]
 ---> Running in 0c3a50e3b4ed
Removing intermediate container 0c3a50e3b4ed
 ---> f5bbd680ac85
Successfully built f5bbd680ac85
Successfully tagged pomo/pomo:latest


$ docker images
REPOSITORY                 TAG       IMAGE ID       CREATED          SIZE
pomo/pomo                  latest    f5bbd680ac85   22 seconds ago   14.5MB
...


```
### Multistage builds

Multistage builds create smaller image sizing. The following Dockerfile uses Go's official image. With this image, you don't have to compile the binary before creating the image:

```Dockerfile
FROM golang:1.19 AS builder
RUN mkdir /distributing 
WORKDIR /distributing 
COPY notify/ notify/
COPY pomo/ pomo/
WORKDIR /distributing/pomo
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -tags=containers

FROM alpine:latest 
RUN mkdir /app && adduser -h /app -D pomo 
WORKDIR /app 
COPY --chown=pomo /pomo/pomo .
CMD ["/app/pomo"]
```


You can also build statically linked binaries and share them with a multistage build that does not include an operating system image or users:

```shell
#Dockerfile.scratch

FROM golang:1.19 AS builder
RUN mkdir /distributing 
WORKDIR /distributing 
COPY notify/ notify/
COPY pomo/ pomo/
WORKDIR /distributing/pomo
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -tags=containers

FROM scratch
WORKDIR /
COPY --from=builder /distributing/pomo/pomo .
CMD ["/pomo"]
```

## Project setup

### Modules

A Go code repository comprises of exactly one module. The module includes packages, and these packages include source files. To create a module, go to the top-level directory of the project and enter the following command:

```bash
go mod init <project-name>
```
The preceding command creates a `go.mod` file in the top-level of your project that lists your project name at the top.

### Packages

Packages are directories in a go project. The name of the directory is the package name. For example, source files in the `go-src/stocks/` package are in the `stocks` package. At the top of the file, declare package names with `package <package-name>`, and import packages with the `import <package-name>` statement.
> `import` statements use the fully-qualified package name. This begins with the module name containing the package. For example, `import go-src/<package-name>`

Prepend any imported package code with the package name, or an alias for the package: `alias package/name`. For example, `s go-src/stocks` allows you to prepend any code with `s.`, such as `s.Investment`.

*main*: any program that has to run as an application must be in the `main` package.

## Arrays

### Internals

An array in Go is a fixed-length data type that contains a contiguous block of elements of the same type

Having memory in a contiguous form can help to keep the memory you use stay loaded within CPU caches longer
Since each element is of the same type and follows each other sequentially, moving through the array is consistent and fast

An array is declared by specifying the type of data to be stored and the total number of elements required, also known as the array’s length.
The type of an array variable includes both the length and the type of data that can be stored in each element

When you pass variables between functions, they’re always passed by value. When your variable is an array, this means the entire array, regardless of its size, is copied and passed to the function.
You can pass a pointer to the array and only copy eight bytes, instead of eight megabytes of memory on the stack
You just need to be aware that because you’re now using a pointer, changing the value that the pointer points to will change the memory being shared

```go
var array [5]int                        // standard declaration
array := [5]int{10, 20, 30, 40, 50}     // array literal declaration
array := [...]int{10, 20, 30, 40, 50}   // Go finds the length based on num of elements
array := [5]int{1: 10, 2: 20}           // initialize specific elements

// pointers
array := [5]*int{0: new(int), 1: new(int)}  // array of pointers
array2 := [3]*string{new(string), new(string), new(string)}
// dereference to assign values
*array[0] = 10
*array[1] = 20
*array2[0] = "Red"
*array2[1] = "Blue"
```

Once an array is declared, neither the type of data being stored nor its length can be changed
they’re always initialized to their zero value for their respective type

An array is a value in Go. This means you can use it in an assignment operation
```go
var array1 [5]string
array2 := [5]string{"Red", "Blue", "Green", "Yellow", "Pink"}
array1 = array2
```
## Maps

A map provides you with an unordered collection of key/value pairs. maps are unordered collections, and there’s no way to predict the order in which the key/value pairs will be returned because a map is implemented using a hash table
The map’s hash table contains a collection of buckets. When you’re storing, removing, or looking up a key/value pair, everything starts with selecting a bucket. This is performed by passing the key—specified in your map operation—to the map’s hash function. The purpose of the hash function is to generate an index that evenly distributes key/value pairs across all available buckets.

The strength of a map is its ability to retrieve data quickly based on the key. They do not have a capacity or a restriction on growth.
Use len() to get the length of the map
The map key can be a value from any built-in or struct type as long as the value can be used in an expression with the == operator. You CANNOT use:
- slices
- functions
- struct types that contain slices

### Creating and initializing

To create a map, use `make` or a map literal. The map literal is idiomatic:

```go
// create with make
dict := make(map[string]int)

// create and initialize as a literal IDIOTMATIC
dict := map[string]string{"Red": "#da1337", "Orange": "#e95a22"}

// slice as the value
dict := map[int]string{}

// assigning values with a map literal
colors := map[string]string{}
colors["Red"] = "#da137"

// DO NOT create nil maps, they result in a compile error
var colors map[string]string{}

```
### Testing if values exist in a map

Test with the following 2 options:
1. You can retrieve the value and a flag that explicitly lets you know if the key exists:
```go
value, exists := colors["Blue"]

if exists {
    fmt.Println(value)
}
```
2. Return the value and test for the zero value to determine if the key exists:
```go
value, exists := colors["Blue"]

if value != "" {
    fmt.Println(value)
}
```

### Iterating over maps with the for range loop

This works the same as slices, except index/value -> key/value:

```go
// Create a map of colors and color hex codes.
colors := map[string]string{
    "AliceBlue":   "#f0f8ff",
    "Coral":       "#ff7F50",
    "DarkGray":    "#a9a9a9",
    "ForestGreen": "#228b22",
}

// Display all the colors in the map.
for key, value := range colors {
    fmt.Printf("Key: %s  Value: %s\n", key, value)
}
```

Use the built-in function `delete` to remove a value from the map:

```go
delete(colors, "Coral")
```

### Passing maps to functions

Functions do not make copies of the map. Any changes made to the map by the function are reflected by all references to the map:

```go
func main() {
    // Create a map of colors and color hex codes.
    colors := map[string]string{
       "AliceBlue":   "#f0f8ff",
       "Coral":       "#ff7F50",
       "DarkGray":    "#a9a9a9",
       "ForestGreen": "#228b22",
    }

    // Call the function to remove the specified key.
    removeColor(colors, "Coral")

    // Display all the colors in the map.
    for key, value := range colors {
        fmt.Printf("Key: %s  Value: %s\n", key, value)
    }
}

// removeColor removes keys from the specified map.
func removeColor(colors map[string]string, key string) {
    delete(colors, key)
}
```

## Type system

Go is statically typed, which means that the compiler wants to know the type for every value in the program. This creates more efficient and secure code that is safe from memory corruption and bugs.

Types tell the compiler how much memory to allocate (its size) and what the memory represents. Some types get their representation based on the machine architecture (64- vs 32-bit).

### User-defined types

There are 2 ways to declare a user-defined type in Go:
1. Use the keyword `struct` to create a composite type:
```go
type user struct {
    name    string
    age     int64
    email   string
}
```
After you define the struct, you can declare a variable of the type with a value or zero value:
```go
// Idiomatic: declare zero-values with var
var bill user
```

You can also declare a struct literal, which is the declaration and values all in one:

```go
// add a comma after the last value
ryan := user{
    name:   "Ryan",
    age:    38,
    email:  "email@email.com",
}

// without the field names
ryan := user{"Ryan", 38, "email@email.com"}

// embedding user-defined types
type superuser struct {
    person      user
    password    string
}

boss := superuser{
    person: user{
        name:   "Ryan",
        age:    38,
        email:  "email@email.com",
    },
    password:   "pword",
}
```

2. Use an existing type as the specification for a new type:

```go
type Distance int64

// lets you add methods to a slice. This is useful when you want to decalure behavior around a built-in or refernce type
type List []string
```

### Methods

Methods add behavior to user-defined types:

```go
func (r receiver) fName(p params) {}
```

The receiver binds the function to the specified type to create a method for the type. How the receiver is declared determines how Go operates on the type value. There are 2 types of receivers:

1. value. When you declare a method using a value receiver, the method will always be operating against a copy of the value used to make the method call

If adding or removing something from a value of this type need to create a new value, then use value receivers for your methods.

```go
func (u user) sendEmail()
ryan := user{"Ryan", 38, "email@email.com"}
ryan.sendEmail()
```
You can also call methods that are declared with a value receiver using a pointer:
```go
func (u user) sendEmail()
ryan := &user{"Ryan", 38, "email@email.com"}
ryan.sendEmail()
```
Go adjusts (dereferences) the pointer value to comply with the method's receiver. So, `sendEmail()` is operating against a copy of the value that `ryan` points to.

2. pointer. When you call a method declared with a pointer receiver, the value used to make the call is shared with the method. Pointer receivers operate on the actual value, and any changes are reflected in the value after the call.

Generally, use pointers when you are using nonprimitive values.

If adding or removing something from a value of this type need to mutate the existing value, then use value receivers for your methods.

```go
func (u *user) changeEmail(email string) {
    u.email = email
}
ryan := &user{"Ryan", 38, "email@email.com"}
ryan.changeEmail("new@email.com")
```
You can also call methods that are declared with a pointer receiver using a value:
```go
func (u *user) changeEmail(email string) {
    u.email = email
}
ryan := user{"Ryan", 38, "email@email.com"}
ryan.changeEmail("new@email.com")
```
Go adjusts (dereferences) the pointer value to comply with the method's receiver. So, `sendEmail()` is operating against a copy of the value that `ryan` points to.

### Reference types
When you decalure a reference type, the value is a header value that contains a pointer to the underlying data structure. The header contains a pointer, so you can pass a copy of any reference type and share the underlying data structure.

The following are reference types:
- slice
- map
- channel
- function types

### Interfaces

Interfaces are types that declare behavior. They are implemented by user-defined types through methods. If a type implements an interface with a method, then a value of that user defined type can be assigned to values of the interface type.

Go uses implicit interfaces, which means that you do not have to declare that the method implements the interface, you just need to use the contract.

When you call a method that accepts an interface value, Go looks at the method set for the user-defined type and tries to find a method that implements the interface. The user-defined type is called the 'concrete type' because it provides the interface concrete behavior. For example:

```go 
// io.Reader interface
type Reader interface {
    Read(b []byte) (n int, err error)
}

// Stringer interface. `Stringer` is the name of the interface, and its signature is within the curly brackets. 
type Stringer interface {
    String() string
}
```
You can implement the `io.Reader` interface for a user-defined type if the function is:
- named `Read`
- accepts a slice of bytes ([]byte is a nil slice)
- returns an integer and an error

You can implement the `Stringer` interface if the function is:
- named `Stringer`
- returns a string


Interface values are two-word data structures:
1. A pointer to an internal table called iTable. iTable contains information about the user-defined stored value that implements the interface: the value's type and its list of methods.
2. A pointer to the actual stored value.

So, if you have a user that implements the `Stringer` interface:

```go
// user type
type user struct {
    name    string
    age     int64
    email   string
}


// user-defined interface 
type birthdater interface {
    birthday()
}

// user implements the Stringer interface
func (u user) String() string {
    return fmt.Sprintf("My name is %s", u.name)
}

// method with ptr receiver to update the user's email
func (u *user) updateEmail(newEmail string) {
    u.email = newEmail
}

// implement the birthdater interface
func (u *user) birthday() {
    u.age++
}

// user struct literal without field names
ryan := user{"Ryan", 38, "email@email.com"}
```
The Stringer iTable contains information about the user type, which includes its list of methods. It also contains a pointer to the memory address that stores the actual user value. You cannot swap value and pointer receiver semantics with interface implementation.

If you implement an interface using a pointer receiver, then only pointers of that type implement the interface:

```go

// user type
type user struct {
    name        string
    password    string
}

// superuser type
type superuser struct {
    person      user
    password    string
}

// user-defined interface 
type passUpdater interface {
    updatePassword(p string)
}

// user

```