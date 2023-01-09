---
title: "Basic data structures"
weight: 20
description: >
  Descibes the basic data structures in Go.
---

## Find a home...

- deferred function calls are not executed when `os.Exit()` is called.

## Go commands



## Basics

#### Formatting verbs

[Formatting verbs](https://pkg.go.dev/fmt#hdr-Printing)

For errors, use `%w` to decorate the original error with additional information for the users. Essentially, you can customize the error message while also returning the default Go error:
```go
if err != nil {
    return nil, fmt.Errorf("Cannot read data from file: %w", err)
}
```
For more info, read [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-add-extra-information-to-errors-in-go).

#### Idioms

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

#### Arrays

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

#### Slices

Use the `...` operator to expand a slice into a list of values:

```go
// accepts any variable number of string args
func getFile(r io.Reader, args ...string) {}
...
// the ... operator expands a slice into a list of values
t, err := getFile(os.Stdin, flag.Args()...) {}
```

```go
slice := make([]string, 5)          // create a slice of strings with 5 capacity
slice := make([]int, 3, 5)          // length 3, cap 5

// Idiomatic slice literals
slice := []int{10, 20, 30}          // slice literal
slice := []string{99: ""}           // initialize the index that represents the length and capacity you need

// nil slice is created by declaring a slice without any initialization
var slice []int                     // nil slice

// empty slice
slice := make([]int, 3)             // empty slice with make
slice := []int{}                    // slice literal to create empty slice of integers

// Create a slice of strings.
// Contains a length and capacity of 5 elements.
source := []string{"Apple", "Orange", "Plum", "Banana", "Grape"}

// Slice the third element and restrict the capacity.
// Contains a length and capacity of 1 element.
slice := source[2:3:3]

// Append a new string to the slice. This doesn't change 'Banana' in the underlying array, it creates a new one
slice = append(slice, "Kiwi")

// change value of slice
slice[1] = 25

// making a slice of a slice
slice := []int{10, 20, 30, 40, 50}
newSlice := slice[1:3]              // length 2, cap 4. 1 is the index position of the element that the new slice starts with
// Length:   3 - 1 = 2
// Capacity: 5 - 1 = 4

// append
newSlice = append(newSlice, 60)

// Slice the third element and restrict the capacity.
// Contains a length of 1 element and capacity of 2 elements.
slice := source[2:3:4]
```
Variadic slices:
```go
s1 := []int{1, 2}
s2 := []int{3, 4}

// Append the two slices together and display the results.
fmt.Printf("%v\n", append(s1, s2...))

Output:
[1 2 3 4]
```
#### Maps

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

// map with a struct literal value
var testResp = map[string]struct {
	Status int 
	Body string 
} {
	//...
}
```
# iota

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



## Project structure

```
.
├── cmd 
│   └── todo
│       ├── main.go         // config, parse, switch {} flags
│       └── main_test.go    // integration tests (user interaction)
├── go.mod
├── todo.go                 // API logic for flags
└── todo_test.go            // unit tests

```

#### run() function in main()

If you use the `main()` function to run all of the code, it is difficult to create integration tests. To fix this, break the `main()` function into smaller functions that you can test independently. Use the `run()` function as a coordinating function for the code that needs to run in `main()`. So, the `main()` function parses command line flags and calls the `run()` function.

When you use the `run()` method strategy, you write unit tests for all the individual functions within `run()`, and you write an integration test for `run()`.

## Print statements

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

## Equality

```go
bytes.Equal(bSlice1, bSlice2)
```

## Strings

Initialize a buffer with string contents using the bytes.NewBufferString("string") func. This simulates an input (like STDIN):
```go
b := bytes.NewBufferString("string")
```

Use `io.WriteString` to write a string to a writer as a slice of bytes:
```go
output, err := io.WriteString(os.Stdout, "Log to console")
if err != nil {
    log.Fatal(err)
}
```
This command seems to be used a lot with the `exec.Command` `os/exec` package?

`.TrimSpace()` removes whitespace, `\n`, `\t`:
```go
func main() {
	fmt.Println(strings.TrimSpace(" \t\n Hello, Gophers \n\t\r\n"))
}
```
You can build strings using `fmt.Sprintf()`:
```go
u := fmt.Sprintf("%s/todo/%d", apiRoot, id)
```

## Pointers

`*` either declares a pointer variable or dereferences a pointer. Dereferencing is basically following a pointer to the address and retrieving stored value.

`&` accesses the address of a variable. Use this for the same reasons that you use a pointer receiver: mutating the object or in place of passing a large object in memory.

Here are some bad examples:

```go
func main() {
	test := "test string"
	var ptr_addr *string
	ptr_addr = &test
	fmt.Printf("ptr_addr:\t%v\n", ptr_addr)
	fmt.Printf("*ptr_addr:\t%v\n", *ptr_addr)
	fmt.Printf("test:\t\t%v\n", test)
	fmt.Printf("&test:\t\t%v\n", &test)
}

// output
ptr_addr:	0xc00009e210
*ptr_addr:	test string
test:		test string
&test:		0xc00009e210
```

## Environment variables

Getting and checking if an environment variable is set:
```go
if os.Getenv("ENV_VAR_NAME") != "" {
    varName = os.Getenv("ENV_VAR_NAME")
}
```


## Interfaces

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
#### io.Writer

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

## Methods

#### Constructors

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

#### Embedded types, extending types

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

#### Value recievers

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

## Errors



## Data structures and formats

#### Slices

Add to a slice with append:
```go
*sliceName = append(*sliceName, valToAppend)
```

#### Structs

Create a zero-value struct:
```go
type person struct {
    name    string
    age     int
}

john := person{}
```


# Contexts

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

## Filesystem

The `filepath` library creates cross-platform filepaths--they compile correctly for each supported OS.

Get the relative directory path between a root and target path:
```go
relDir, err := filepath.Rel(root, filepath.Dir(path))
if err ...
```


## Flags

`flag.<FunctionName>` lets you define CLI flags. For example, to create a flag that performs an action if the flag is provided, you can use `flag.Bool`.

The following flag function definition returns the value of a `bool` variable that stores the value of the flag. After you create a flag, you have to call the `Parse()` function to parse the arguments provided to the command line:

```go
lines := flag.Bool("l", false, "Count the number of lines")
      // flag.Bool(flagName, default value, usage info)
flag.Parse()
```
> **IMPORTANT**: Each `flag.*` returns a pointer. To use the value in this variable that 'points' to an address, you have to derefence it with the `*` symbol. If you don't dereference, you will use the address of the variable, not the value stored at the address

Now, you have a variable `lines` that stores the address of a `bool` set to `false`. When a user includes the `-l` flag in the CLI invocation, `lines` is set to true. For `str := flag.String(...)`, the variable stores the string that the user enters after the `-str` flag.

### Multiple flags

If you use multiple flags in your application, use a `switch` statment to select the action based on the provided flags:

```go
switch {
case *flag1:
    // handle flag
case *flag2:
    // handle flag
default:
...
}
```
### Usage info

The default values for each flag are listed when the user uses the `-h` option. You can add a custom usage message with the `flag.Usage()` function. You have to assign `flag.Usage()` a immediately-executing function that prints info to STDOUT.

Place the `flag.Usage()` definition at the beginning of the main method:

```go
flag.Usage = func() {
    fmt.Fprintf(flag.CommandLine.Output(), "%s tool. Additional info\n", os.Args[0])
    fmt.Fprintf(flag.CommandLine.Output(), "Copyright 2022\n")
    fmt.Fprintln(flag.CommandLine.Output(), "Usage information:")
    // print the default settings for each flag
    flag.PrintDefaults()
}
```

```go
func cliFunc(r io.Reader, useLines bool) {}
cliFunc(os.Stdin, *lines)
```
## Time

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

## Building commands with os/exec

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

#### Example 1

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

#### Example 2

Create a slice literal to store the parameters:
```go
params := []string{}
params = append(params, arg1)
params = append(params, arg2)
// expand slice values into function
exec.Command(/path/, params...)
```

#### Useful exec. methods

```go
exec.LookPath(fileName string) // returns location of fileName in PATH or error
```

# Logging

Use logs to provide feedback for background processes. To create a logger, you need to create:
- [*log.logger](https://pkg.go.dev/log#Logger) type
- Logging destination ([w io.Writer](#interfaces))

By default, Go's `log` library sends messages to STDERR, but you can configure it to write to a file. It adds the date and time to each log entry, and you can add a prefix to the string to help searchability
```go
l := log.New(io.Writer, "LOGGER PREFIX: ", log.LstdFlags)
```
log.LstdFlags uses the default log flags, such as date and time.

# Tests

`iotest.TimeoutReader(r)` simulates a reading failure and returns a timeout error.

#### Integration tests

Integration tests test how the program behaves when interacted with from the outside world--how a user uses it. This means that you test the `main()` method.

In Go, you test the main method with the `TestMain()` function so you can set up and tear down resources more easily. For example, you might need to create a temporary file or build and execute a binary. You do not want to keep these artifacts in the program after testing.

Follow these general guidelines when running integration tests:
1. Check the machine with `runtime.GOOS`
2. Create the build command with `exec.Command()`, then use `.Run()` to execute that command. Check for errors
3. Run the tests with `m.Run()`
4. Clean up any artifacts with `os.Remove(artifactname)`

#### General flow

When you create a test, you need to set up an environment, execute the functionality that you are testing, then tear down any temporary files you created in the environment:

```go
func TestMethod(t *testing.T) {
    // set up env
    st := testStruct{}
    // test the functionality, including testing for errors
    st.MethodImTesting(...args)

    if err != nil {
        ...
    }
}
```

#### Test helpers with t.Helper()

Add `t.Helper()` to mark a function as a test helper.

You can defer a the cleanup function. If a helper function fails a test, Go prints the line in the test function that calls the helper function.

You can also return a cleanup function to make sure that your tests leave no test artifacts in the filesystem. There is also a `t.Cleanup()` method that registers a cleanup function.

#### Subtests with t.Run()

Use `t.Run()`, You can run subtests within a test function. `t.Run()` accepts two parameters: the name of the test, and an unnamed test function. Nest `t.Run()` under the main func Test* function to target functionality, such as different command line options:

```go
func TestMain(m *testing.M) {
    // build binary
    // build command to execute binary
    // run command
    // result := m.Run() // this runs the t.Run() tests
    // clean up tests
    t.Run("Subtest 1", func(t *testing.T) {
        // run subtest
    })
}
```

#### Package naming

Place `*_test.go` files in the same directory as the code that you are testing. When you declare the `package` in the test file, use the original package name followed by `_test`. For example:

```go
package original_test
```

#### Table tests

1. Define your test cases as a slice of anonymous struct that contains the data required to run your tests and the expected results
2. Iterate over this slice with a `for range` loop

Anonymous struct example:
```go
func TestFilterOut(t *testing.T) {
	testCases := []struct {
		name     string
		file     string
		ext      string
		minSize  int64
		expected bool
	}{
		{"FilterNoExtension", "testdata/dir.log", "", 0, false},
		{"FilterExtensionMatch", "testdata/dir.log", ".log", 0, false},
		{"FilterExtensionNoMatch", "testdata/dir.log", ".sh", 10, true},
		{"FilterExtensionSizeMatch", "testdata/dir.log", ".log", 10, false},
		{"FilterExtensionSizeNoMatch", "testdata/dir.log", ".log", 20, true},
	}
    ...
}
```

#### Utilities

Create a temporary file if you need to test an action like deleting a file from the file system. Use `os.CreateTemp()`. Be sure to clean up with `os.Remove(tempfile.Name())`:

```go
os.CreateTemp(".", )
```

#### Error handling

The test object (`*testing.T`) provides the following methods troubleshoot during testing

`t.Fatalf()` logs a formatted error and fails the test, then stops test execution:
```go
t.Fatalf("Error message: %s", err) // Logf() + FailNow()
```

`t.Errorf()` logs a formatted error and fails the test, but continues test execution:
```go
t.Errorf("Error message: %s", err) // Logf() + Fail()
```

#### Strategies

When testing file writes, use _goldenfiles_: files that contain the expected results and that you load during tests to validate output.

> **IMPORTANT**: Put goldenfiles, and other testing files, in a directory called `testdata`. Go tooling ignores this directory when building and compiling the program.

#### Integration testing with external resources

If you are testing external commands that modify the state of an external resource, the testing conditions change after each test.

## Templates

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

## Benchmarking

Running Go benchmarks is similar to running tests:
1. Write the benchmark functions using the `testing.B` type.
2. Run the benchmarks using the `go test` tool with the `-bench` parameter.

You want to benchmark your tool according to the main use case, so create test files to replicate your workload.


#### Example benchmark function

The following benchmark test runs a tool on all CSV test files:
```go
func BenchmarkRun(b *testing.B) {
	filenames, err := filepath.Glob("./testdata/benchmark/*.csv")
	if err != nil {
		b.Fatal(err)
	}

	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		if err := run(filenames, "avg", 2, io.Discard); err != nil {
			b.Error(err)
		}
	}
}
```
In the preceding example:
- `filepath.Glob()` matches a pattern to find all files with the `.cvs` file extension.
- `b.ResetTimer()` resets the benchmark clock. This ignores any time used to prepare for the benchmark's execution.
- `b.N` is the upper limit of the loop, which is adjusted to last about one second.
- `io.Discard` implements the `io.Writer` interface, but does not write output to any resource.

#### Benchmark run commands

Run the benchmark tool with the `-bench <regex>` parameter, where `regex` is a regular expression that matches the benchmark tests that you want to run. To skip regular tests in the test files, include `-run ^$` in the command. For example:
```go
$ go test -bench . -run ^$
goos: linux
goarch: amd64
pkg: colstats
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRun-12    	       2	 543565503 ns/op
PASS
ok  	colstats	1.588s
```
Run additional executions of the benchmark test with the `-benchtime` parameter. This parameter accepts a duration in time to run the benchmark, or a fixed number of executions. 

Run and save benchmarks to a file with the `tee` command. For example:
```
$ go test -bench . -benchtime=10x -run ^$ | tee benchresults00.txt
goos: linux
goarch: amd64
pkg: colstats
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRun-12    	      10	 546183149 ns/op
PASS
ok  	colstats	6.067s

$ cat benchresults00.txt 
goos: linux
goarch: amd64
pkg: colstats
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRun-12    	      10	 546183149 ns/op
PASS
ok  	colstats	6.067s
```
Compare two benchmark files to measure improvements after optimizations:
```go

```

## Profiling tools

The Go profiler gives you a breakdown of where your program spends its time executing. You can determine which functions consume the most time and target them for optimization.

Profile your tools using one of the two following options:
1. Add code directly to your program. This requires that you manually maintain profiling code.
2. Run the profiler integrated with the testing and benchmarking tools
   > The second option is easiest

Use the CPU profiler to create a profile file and a .test file:
```go
$ go test -bench . -benchtime=10x -run ^$ -cpuprofile cpu00.pprof
goos: linux
goarch: amd64
pkg: colstats
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRun-12    	      10	 555744851 ns/op
PASS
ok  	colstats	6.230s
```
In the preceding command, `cpu00.pprof` is the 'profile name'.

Analyze the profiling results with `go tool pprof` and the profile name:
```go
$ go tool pprof cpu00.pprof 
File: colstats.test
Type: cpu
Time: Dec 5, 2022 at 12:12am (EST)
Duration: 6.22s, Total samples = 7.18s (115.38%)
Entering interactive mode (type "help" for commands, "o" for options)
```
The `top` command shows where your program is spending the most time:
```go
(pprof) top
Showing nodes accounting for 4880ms, 67.97% of 7180ms total
Dropped 120 nodes (cum <= 35.90ms)
Showing top 10 nodes out of 87
      flat  flat%   sum%        cum   cum%
    1000ms 13.93% 13.93%     1090ms 15.18%  runtime.heapBitsSetType
     730ms 10.17% 24.09%     4340ms 60.45%  encoding/csv.(*Reader).readRecord
     620ms  8.64% 32.73%      620ms  8.64%  runtime.memmove
     520ms  7.24% 39.97%     2510ms 34.96%  runtime.mallocgc
     490ms  6.82% 46.80%      490ms  6.82%  indexbytebody
     420ms  5.85% 52.65%      420ms  5.85%  strconv.readFloat
     360ms  5.01% 57.66%      360ms  5.01%  runtime.memclrNoHeapPointers
     290ms  4.04% 61.70%      290ms  4.04%  runtime.nextFreeFast (inline)
     260ms  3.62% 65.32%      260ms  3.62%  runtime.procyield
     190ms  2.65% 67.97%      190ms  2.65%  syscall.Syscall
```
The `top` command does not sort by cumulative time. Add the `-cum` flag:
```go
(pprof) top -cum
Showing nodes accounting for 2.45s, 34.12% of 7.18s total
Dropped 120 nodes (cum <= 0.04s)
Showing top 10 nodes out of 87
      flat  flat%   sum%        cum   cum%
         0     0%     0%      6.24s 86.91%  colstats.BenchmarkRun
     0.01s  0.14%  0.14%      6.24s 86.91%  colstats.run
         0     0%  0.14%      6.24s 86.91%  testing.(*B).runN
         0     0%  0.14%      5.71s 79.53%  testing.(*B).launch
     0.13s  1.81%  1.95%      5.69s 79.25%  colstats.csv2float
     0.05s   0.7%  2.65%      4.73s 65.88%  encoding/csv.(*Reader).ReadAll
     0.73s 10.17% 12.81%      4.34s 60.45%  encoding/csv.(*Reader).readRecord
     0.52s  7.24% 20.06%      2.51s 34.96%  runtime.mallocgc
     0.01s  0.14% 20.19%      1.59s 22.14%  runtime.makeslice
        1s 13.93% 34.12%      1.09s 15.18%  runtime.heapBitsSetType
```
After all the testing functions, you can see that the csv2float function takes up most of the execution time. To investigate further, use the `list <regex>` command. This command shows the source code for any function that matches the `regex` and displays how much time is spent executing each line in the function:
```go
(pprof) list csv2float
Total: 7.18s
ROUTINE ======================== colstats.csv2float in /home/ryanseymour/Development/go-projects/command-line/colstats/csv.go
     130ms      5.69s (flat, cum) 79.25% of Total
         .          .     24:// any function that matches this signature is of this type
         .          .     25:type statsFunc func(data []float64) float64
         .          .     26:
         .          .     27:func csv2float(r io.Reader, column int) ([]float64, error) {
         .          .     28:	// Create the CSV Reader used to read in data from CSV files
         .       20ms     29:	cr := csv.NewReader(r)
         .          .     30:	// adjusting column arg for 0-based index
         .          .     31:	column--
         .          .     32:	// Read in all CSV data
         .      4.73s     33:	allData, err := cr.ReadAll()
         .          .     34:	if err != nil {
         .          .     35:		return nil, fmt.Errorf("Cannot read data from file: %w", err)
         .          .     36:	}
         .          .     37:
         .          .     38:	var data []float64
         .          .     39:	/*
         .          .     40:		convert [][]string to [][]float64
         .          .     41:	*/
         .          .     42:
         .          .     43:	// loop through all records
      60ms       60ms     44:	for i, row := range allData {
         .          .     45:		// skip the first row that contains the column headers
      10ms       10ms     46:		if i == 0 {
         .          .     47:			continue
         .          .     48:		}
         .          .     49:		// Checking number of cols in CSV file to verify flag value
         .          .     50:		if len(row) <= column {
         .          .     51:			// file does not have that many columns
         .          .     52:			return nil,
         .          .     53:				fmt.Errorf("%w: File has only %d columns", ErrInvalidColumn, len(row))
         .          .     54:		}
         .          .     55:		// Try to convert data read into a float number
      40ms      770ms     56:		v, err := strconv.ParseFloat(row[column], 64)
         .          .     57:		if err != nil {
         .          .     58:			return nil, fmt.Errorf("%w: %s", ErrNotNumber, err)
         .          .     59:		}
         .          .     60:
      20ms      100ms     61:		data = append(data, v)
         .          .     62:	}
         .          .     63:	return data, nil
         .          .     64:}
```
To view a relationship graph in a browser, use the `web` command:
```go
(pprof) web
(pprof) quit
```
#### Memory profiling

This is similar to running a benchmark, but use the `-memprofile` option:
```go
$ go test -bench . -benchtime=10x -run ^$ -memprofile mem00.pprof
goos: linux
goarch: amd64
pkg: colstats
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRun-12    	      10	 551000648 ns/op
PASS
ok  	colstats	6.049s
```
This command creates the `mem00.pprof` file in your current working directory.

View the results of the memory profile with `go tool pprof` and the `-alloc_space` option:
```go
$ go tool pprof -alloc_space mem00.pprof
File: colstats.test
Type: alloc_space
Time: Dec 5, 2022 at 12:35am (EST)
Entering interactive mode (type "help" for commands, "o" for options)
```
Use the `top` command with the and sort with the `-cum` flag to view the parts of the program that allocate the most memory:
```go
(pprof) top -cum
Showing nodes accounting for 5.10GB, 99.89% of 5.10GB total
Dropped 28 nodes (cum <= 0.03GB)
Showing top 10 nodes out of 11
      flat  flat%   sum%        cum   cum%
         0     0%     0%     5.10GB 99.92%  colstats.BenchmarkRun
    1.13GB 22.23% 22.23%     5.10GB 99.92%  colstats.run
         0     0% 22.23%     5.10GB 99.92%  testing.(*B).runN
         0     0% 22.23%     4.64GB 90.89%  testing.(*B).launch
    0.64GB 12.50% 34.73%     3.96GB 77.66%  colstats.csv2float
    2.05GB 40.11% 74.84%     3.29GB 64.43%  encoding/csv.(*Reader).ReadAll
    1.24GB 24.32% 99.16%     1.24GB 24.32%  encoding/csv.(*Reader).readRecord
         0     0% 99.16%     0.46GB  9.03%  testing.(*B).run1.func1
         0     0% 99.16%     0.04GB  0.74%  bufio.NewReader (inline)
    0.04GB  0.74% 99.89%     0.04GB  0.74%  bufio.NewReaderSize (inline)
(pprof) 
```
#### Total memory allocation

Use the -benchmem flag to view the total memory allocation for a program. The following command uses `tee` to send the output to STDOUT and the file parameter:
```go
$ go test -bench . -benchtime=10x -run ^$ -benchmem | tee benchresults00m.txt
goos: linux
goarch: amd64
pkg: colstats
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRun-12    	      10	 530825085 ns/op	495567618 B/op	 5041035 allocs/op
PASS
ok  	colstats	5.837s
```

#### Tracing

Tracing shows you how your application manages resources, such as network connections or file reads. 

Add the `-trace` option to the benchmark command:

```go
$ go test -bench . -benchtime=10x -run ^$ -trace trace01.out
goos: linux
goarch: amd64
pkg: colstats
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRun-12    	      10	 571258799 ns/op
PASS
ok  	colstats	6.253s
```

View the results with `go tool trace`. The following command opens the contents of the trace file on a random port in the browser:

```go
$ go tool trace trace01.out 
2022/12/05 18:48:49 Parsing trace...
2022/12/05 18:48:50 Splitting trace...
2022/12/05 18:48:51 Opening browser. Trace viewer is listening on http://127.0.0.1:46209
```
## Concurrency

Go provides two strategies that maintain data integrity when you are writing concurrent programs:
- Locks, such as Mutex
- goroutines and channels

#### Channels

Channels allow goroutines to communicate with each other.

```go
// create a channel with make
ch := make(chan type)
// Use an empty struct to create a channel for done. done channels
// only signal that processing is complete, and an empty struct does not allocate
// any memory
done := make(chan struct{})
```

#### WaitGroups

A WaitGroup is a mechanism that coordinates the goroutine execution. When you create a goroutine, add 1 to the WaitGroup. When a goroutine finishes, subtract 1 from the WaitGroup. Use `Wait()` to wait until all goroutines are finished so you can complete execution.

```go
wg := sync.WaitGroup{}
```

#### Scheduling contention and worker queues

This is when you create too many goroutines and they compete for work. The answer is to use worker queues.

When using worker queues, you create one goroutine per available CPU, and have another goroutine send jobs to be executed by the workers. So, the CPUs are the workers.

Use `runtime.NumCPU()` to determine the number of available CPUs:
```go

```

#### Goroutines

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

#### Locks

The [Locker interface](https://pkg.go.dev/sync@go1.19.4#Locker) has methods that lock and unlock an object. Use this when you want to prevent concurrent access to an object during operations.

`&sync.Mutex` implements the Locker interface:
```go
mu := &sync.Mutex{}
```


# Signals

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

# Sorting

The `sort` package provides functions that sort 

# Cobra CLI

TODO: Setup

1. Create the functions that the tool will use
2. Add the CLI option with `cobra-cli add <toolname>`
3. In `<toolname>.go`, update the fields in the &cobra.Command object as needed. Add some of the following fields, if necessary:
   - SilenceUsage:
   - Args: 
4. Update the `Run` field to `RunE`. `RunE` returns a function so you can test it. The signature returns an error.

#### Install

Download and install from the [Github repo](https://github.com/spf13/cobra).

1. `$ go get -u github.com/spf13/cobra@latest`
2. `$ go install github.com/spf13/cobra-cli@latest`

#### Start a project

Use the `cobra-cli` tool to init a project:
```bash
$ cobra-cli init <project-name>
```
Add subcommands to a project:
```bash
$ cobra add <subcommand-name>
```
This adds a new file with boilerplate code in the `/cmd` directory.

#### Add subcommands

Cobra has a flag package is an alias to `pflag`, a replacement for Go's standard flag package that includes POSIX compliance.

Persistent flags use the following structure:

```go
rootCmd.PersistentFlags().StringP(<command-name>, <short-hand>, <default>, <short-desc>)

// example
rootCmd.PersistentFlags().StringP("hosts-file", "f", "pScan.hosts", "pScan hosts file")
```

Command to create a subcommand:

```shell
$ cobra-cli add <subcommand-name> -p <parent-command-instance-var>
```
The `instance variable` is the name of the command variable in `root.go`:

```go 

var hostsCmd = &cobra.Command{
	Use:   "hosts",
	Short: "Manage the hosts list",
	Long: "...",
}
```

For example:

```shell
$ cobra-cli add list -p hostsCmd
```

#### Command completion and docs




## Viper

Install [Viper](https://github.com/spf13/viper):

```shell
$ go get github.com/spf13/viper
```

#### Initial setup

```go
// cmd/root.go

func init() {
    ...

	// replace dash with underscore for some OSs
	replacer := strings.NewReplacer("-", "_")
	viper.SetEnvKeyReplacer(replacer)
	// add prefix to host file env var
	viper.SetEnvPrefix("PSCAN")

	// bind key to the flag
	viper.BindPFlag("hosts-file", rootCmd.PersistentFlags().Lookup("host-file"))

	...
}
```

#### Persistent flags

Add these flags in the root.go file. Persistent flags are available to the command and all subcommands under that command.

# Network connections

# Clients

In Go, a single client can create multiple connections.

#### Creating clients

Go provides a default client, but you cannot customize it with functionality such as a connection timeout:

```go
func newClient() *http.Client {
	c := &http.Client{
		Timeout: 10 * time.Second,
	}
	return c
}
```

#### Model responses

You have to model responses with structs. Create a struct to model an individual resource and a struct to model the server response.

#### Sending requests

Create a generic method that can send any type of request and handle any response code. The `.Do()` method can send any type of request (GET, POST, PUT, DELETE, etc.).

A reqeust should perform the following:
- Create a request object with [.NewRequest(method, url string, body io.Reader)](https://pkg.go.dev/net/http#NewRequest)
- Set any content headers with [Header.Set(header-name, value)](https://pkg.go.dev/net/http#Header.Set)
- Execute the request with the .Do() method. Save the response in a var
- Close the response body (sooner than later)
- Check that the response code is what you expected. If not, use custom error messages with the `%w` formatting verb.
- If the request was successful, return `nil`

For example:

```go
func sendRequest(url, method, contentType string, expStatus int, body io.Reader) error {
    // create a new request
	req, err := http.NewRequest(method, url, body)
	if err != nil {
		return err
	}

    // Set any headers
	if contentType != "" {
		req.Header.Set("Content-Type", contentType)
	}

    // execute the request with .Do() and save the response in a var
	r, err := newClient().Do(req)
	if err != nil {
		return err
	}

    // make sure the response body is closed 
	defer r.Body.Close()

    // check status codes
	if r.StatusCode != expStatus {
		msg, err := io.ReadAll(r.Body)
		if err != nil {
			return fmt.Errorf("Cannot read body: %w", err)
		}
		err = ErrInvalidResponse
		if r.StatusCode == http.StatusNotFound {
			err = ErrNotFound
		}
		return fmt.Errorf("%w: %s", err, msg)
	}

    // return nil for a successful request
	return nil
}
```

#### CRUD functions

Use the following functions with the generic `sendRequest()` function for CRUD operations:

```go


// PATCH
// Use Sprintf to format a url with query parameters
func completeItem(apiRoot string, id int) error {
	u := fmt.Sprintf("%s/todo/%d?complete", apiRoot, id)

	return sendRequest(u, http.MethodPatch, "", http.StatusNoContent, nil)
}
```

#### Integration tests

When you run unit tests, you are using local resources that mock the live API. You can run these as much as you'd like. However, to run integration tests, you need to run your client against the actual API. To make sure that you do not make too many requests to the actual API, use build constraints.

Main challenge is that the test needs to be reproducible. 

Define build constraints at the top of the file:

```go
// +build integration

package cmd

// file contents...
```

To exclude a file from integration tests, use the `!` operator before `integration`:

```go
//go:build !integration

package cmd

// file contents
```
When you run the tests, use `-tags <tag-name>` in the command. For example:
```shell
$ go test -v ./cmd -tags integration
```

After you run the integration tests one time, add the `-count=1` tag to ensure that the test does not used cached results:

```shell
$ go test -v ./cmd -tags integration -count=1
```
# Servers

The `net/http` package provides the `ListenAndServer()` function that creates a default server. However, this function does not allow you to define timeouts to manage bad connections or server resources, so you should define custom server.

#### Custom servers

Create a custom server with the [`Server` type](https://pkg.go.dev/net/http#Server). The `Server` type is a struct with the following fields:

```go 
type Server struct {
	Addr string
	Handler Handler
	TLSConfig *tls.Config
	ReadTimeout time.Duration
	ReadHeaderTimeout time.Duration
	WriteTimeout time.Duration
	IdleTimeout time.Duration
	MaxHeaderBytes int
	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)
	ConnState func(net.Conn, ConnState)
	ErrorLog *log.Logger
	BaseContext func(net.Listener) context.Context
	ConnContext func(ctx context.Context, c net.Conn) context.Context
}
```

Implement the fields that you need when you define a custom server:

```go 
customServer := &http.Server{
    Addr:         fmt.Sprintf("%s:%d", *host, *port),
    Handler:      MuxFunc(*todoFile),
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
}
```

Start the server with `ListenAndServe`:

```go
if err := s.ListenAndServe(); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
}
```

#### Multiplexers

A multiplexer maps incoming requests to the proper handler functions using the request URL. `net/http` provides the DefaultServeMux function that returns the default multiplexer, but you should define a custom multiplexer for the following reasons:
- The default registers routes globally.
- You can add dependencies to the routes.
- Custom multiplexers allow integration testing

#### HTTP handlers

Handlers handle a request and responds to it. An object of type `Handler` satisfies the `Handler` field in a custom HTTP server.

You create the server, then use `HandleFunc` to register routes to handler functions. Then you use `HandlerFunc` to define the handler function for the route.

- `http.Handler` is an interface that has the `ServeHTTP(w ResponseWriter, r *Request)` signature.
- `http.HandlerFunc` is an adapter type that lets you define ordinary functions as HTTP handlers. It implements `ServeHTTP()`.
- `http.HandleFunc` registers a handler with a multiplexer server. It accepts two arguments: the path as a `string`, and the handler function. It implements `ServeHTTP()`.
- `http.NewServeMux()` returns a custom server. It implements `ServeHTTP()`.

#### Router functions

```go
func todoRouter(todoFile string, l sync.Locker) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		list := &todo.List{}

		l.Lock()
		defer l.Unlock()
		if err := list.Get(todoFile); err != nil {
			replyError(w, r, http.StatusInternalServerError, err.Error())
			return
		}

		if r.URL.Path == "" {
			switch r.Method {
			case http.MethodGet:
				getAllHandler(w, r, list)
			case http.MethodPost:
				addHandler(w, r, list, todoFile)
			default:
				message := "Method not supported"
				replyError(w, r, http.StatusMethodNotAllowed, message)
			}
			return
		}

		id, err := validateID(r.URL.Path, list)
		if err != nil {
			if errors.Is(err, ErrNotFound) {
				replyError(w, r, http.StatusNotFound, err.Error())
				return
			}
			replyError(w, r, http.StatusBadRequest, err.Error())
			return
		}

		switch r.Method {
		case http.MethodGet:
			getOneHandler(w, r, list, id)
		case http.MethodDelete:
			deleteHandler(w, r, list, id, todoFile)
		case http.MethodPatch:
			patchHandler(w, r, list, id, todoFile)
		default:
			message := "Method not supported"
			replyError(w, r, http.StatusMethodNotAllowed, message)
		}
	}
}
```

Multiplexer definition:

```go
func newMux(todoFile string) http.Handler {
	m := http.NewServeMux()
	mu := &sync.Mutex{}

	m.HandleFunc("/", rootHandler)

	t := todoRouter(todoFile, mu)

	m.Handle("/todo", http.StripPrefix("/todo", t))
	m.Handle("/todo/", http.StripPrefix("/todo/", t))

	return m
}
```

# HTTP general

#### Status codes

Use `http.StatusText()` to return the text for an HTTP status code. For example:
```go
http.StatusText(200)
```
Go provides status code constants to make code more human readable. For example:
`http.NotFound` when a client requests an unknown route
`http.StatusOK` 200
`http.StatusInternalServerError` return for standard server error
`http.StatusMethodNotAllowed` when a client uses an unsupported request method
`http.StatusBadRequest` 
`http.StatusCreated` 201. request succeeded and led to the creation of a resource

#### Request methods

Go provides constant values to identify request methods:

```go
http.MethodGet
http.MethodPost
```