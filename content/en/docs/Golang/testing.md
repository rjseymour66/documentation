---
title: "Testing"
weight: 9
description: >
  Working with Testing in Go.
---

## Package naming

Place `*_test.go` files in the same directory as the code that you are testing. When you declare the `package` in the test file, use the original package name followed by `_test`. For example:

```go
package original_test
```

## Write a test

Each test function must use the `TestXxx(t *testing.T)` signature, or the Go compiler does not recognize and run the test. For example, the following test verifies the functionality of a simple `Add(a, b int) int` function:

```go
func TestAdd(t *testing.T) {
    ...
}
```

Each test contain three main sections:

Arrange
: Set up the test inputs and expected values.

Act
: Execute the portion of code that you are testing.

Assert
: Verify that the code returned the correct values. You can use the `got`/`want` or `got`/`expected` formatting.

### Arrange

A common way to arrange a test is to use table tests. Table tests are a way to provide multiple test cases that you loop over and test during the `Act` stage. To set up a table test, complete the following:
1. Create a `testCase` struct that models the inputs and expected outputs of the test:
   ```go
   type testCase struct {
	   a        int
	   b        int
	   expected int
   }
   ```
2. Use a map literal with a `string` key and `testCase` value. The `string` key is the name of the test, and `testCase` is the test values:
   ```go
   tt := map[string]testCase{   // tt for table tests
        "test one": {
            a:        4,
            b:        5,
            expected: 9,
        },
        "test two": {
            a:        -4,
            b:        15,
            expected: 11,
        },
        "test three": {
            a:        5,
            b:        1,
            expected: 6,
        },
   }
   ```
### Act

Within the same `TestAdd()` function, write a `for range` loop. This is where you execute each test case with the code that you are testing. Use the `t.Run()` subtest method in the `for range` loop to run each individual test case with a name. `t.Run()` accepts two parameters: the name of the test, and an unnamed test function:

```go
for name, tc := range tt {
	t.Run(name, func(t *testing.T) {
		// act
		got := add(tc.a, tc.b)
		...
	})
}
```
In the previous example, `name` is the key in the `tt` map, and `tc` is the `testCase` struct in the `tt` map.

### Assert

In the assert step, you compare the actual values (what you got in the Act step) with the expected value, which is usually a field in the `testCase` struct. Asserts are generally `if` statements that return a formatted error with `t.Errorf` when the `got` and `expected` values do not match:

```go
for name, tc := range tt {
	t.Run(name, func(t *testing.T) {
        ...
		// assert
		if got != tc.expected {
			t.Errorf("expected %d, got %d", tc.expected, got)
		}
	})
}
```

## Testing dependencies

### Stubs
### Mocks
### Fakes

## Unit tests 

Unit tests test simple pieces of code, such as functions or methods.

### Table-driven tests

Also called data-driven and parameterized tests. They verify code with varying inputs. You can also implement subtests that run tests in isolation.

Imagine table-driven tests as actual tables, where the headers are struct fields, and the rows become individual slices in the test cases:

| product     | rating  | price |
|:------------|---------|-------|
| prod one    | 5       | 20    |
| prod two    | 10      | 30    |
| prod three  | 15      | 40    |

You can represent this in a test as follows:

```go
func TestTable(t *testing.T) {
    type product struct {
        product string
        rating  int
        price   float64
    }
    testCases := []product {
        {"prod one", 5, 20},
        {"prod two", 10, 30},
        {"prod three", 15, 40},
    }
}
```

Reuse assertion logic and naming makes each test identifiable.
`t.Run()` defines a subtest. It accepts the name of the test, and then a testing function:
```go
for _, tt := range testCases {
    t.Run(tt.name, func (t *testing.T){
        ...
    })
}
```

### Parallel testing 

Tests that have `t.Parallel()` on the first line of the `t.Run()` function.

Go pauses all tests with `t.Parallel()` and then runs them when other tests complete. `GOMAXPROCS` determines how many tests run in parallel. By default, `GOMAXPROCS` is set to the number of CPUs on the machine.


### Skipping tests 

Use `t.Skip()` to skip tests during execution. Use this with `testing.Short()` and the `-test.short` argument to tell the testing package to skip specific tests.

This helps separate unit tests from integration tests. Integration tests take longer than unit tests, so you can designate a test as a unit test as follows:
```go
func TestIntegrationTest(t *testing.T)  {
    if testing.Short() {
        t.Skip("Skip integration tests")
    }
    // continue test ...
}
```

To skip this test, use `-test.short` when you execute the tests:
```shell 
$ go test -v -test.short
```

### Cleanup 

Use `t.Cleanup()` to clean up testing resources. Do not use the standard `defer` functions:

```go
func TestWithResources(t *testing.T)  {
    // testing...
    t.Cleanup(func() {
        // cleanup logic
    })
    // continue test ...
}
```

### Helpers 

A test helper can improve readability of your tests. Unfortunately, when a test fails within the test helper, the testing package logs helper function line number after failure--this makes it difficult to pinpoint where the test failed.

Use the `t.Helper()` function to designate a function as a helper function. the testing pacakge ignores the helper function line number and instead logs the line number of the failing test:

```go
func helper(t *testing.T)  {
    t.Helper()
    // helper logic 
}

func TestWithHelper(t *testing.T)  {
    // testing...
    helper(t)
    // continue test ...
}
```

Helper functions accept an instance of the testing.T type, so make sure you pass the `t` testing instance to the helper in the `TestXxx` function. This provides the helper with access to the testing instance as the rest of the test function.

### Temporary directories

The testing type provides the `TempDir()` function to create a temporary directory that is deleted automatically when testing completes. This means that you do not have to write cleanup logic:

```go 
func TestTempDir(t *testing.T)  {
    tmpDir := t.TempDir()
    
    // testing logic 
}
```

## Coverage tests 

Go can generate a test report that details what code is sufficiently tested.

In its simplest form, you can display the percentage of tested code with the `cover` flag:

```shell 
go test -cover
```

By default, the testing package only tests packages with `*_test.go` files. To test all packages, use the `coverpkg` flag: 

```shell 
$ go test ./... -coverpkg=./...
```

Generate a coverage report file with the `coverprofile` flag and a custom filename:

```shell 
$ go test -coverprofile=coverage_output_filename
```

To format the coverage output, use `go tool` with `cover` and the `html` flag:

```shell 
$ go tool cover -html=html_coverage_output_filename
```

## Benchmark tests 

Benchmark tests measure code performance. These tests require the following: 
- Test name must start with `BenchmarkXxx`
- Accepts the `*testing.B` parameter
- Must use a `for` loop with `b.N` as the upper bound

`b.N` is adjusted at runtime. The test completes when the execution time of each iteration is statistically stable:

```go 
func BenchmarkHelloWorld(b *testing.B)  {
    for i := 0; i < b.N; i++ {
        HelloWorld(i)
    }
}
```

## Fuzz tests 

Fuzz tests use random input to detect bugs and edge cases. Fuzz tests require the following:
- Test name must start with `FuzzXxx`
- Accepts the `*testing.F` parameter
- Each function must define the initial value (seed corpus) with `f.Add()`
- There must be a Fuzz target

```go 
func FuzzHelloWorld(f *testing.F)  {
    f.Add(5)
    f.Fuzz(func(t *testing.T, a int)  {
        HelloWorld(a)
    })
}
```

## Integration tests

Integration tests test how the program behaves when interacted with from the outside world--how a user uses it. This means that you test the `main()` method.

In Go, you test the main method with the `TestMain()` function so you can set up and tear down resources more easily. For example, you might need to create a temporary file or build and execute a binary. You do not want to keep these artifacts in the program after testing.

Follow these general guidelines when running integration tests:
1. Check the machine with `runtime.GOOS`
2. Create the build command with `exec.Command()`, then use `.Run()` to execute that command. Check for errors
3. Run the tests with `m.Run()`
4. Clean up any artifacts with `os.Remove(artifactname)`


## Utilities

Create a temporary file if you need to test an action like deleting a file from the file system. Use `os.CreateTemp()`. Be sure to clean up with `os.Remove(tempfile.Name())`:

```go
os.CreateTemp(".", )
```

## Error handling and logging

The testing package provides methods troubleshoot during testing. The most useful are `t.Errorf()` and `t.Fatalf()`. The following table describes all available `t.*` test methods:

| Method          | Description |
|-----------------|:------------|
| `t.Log()`        | Log a message. |
| `t.Logf()`       | Log a formatted message.|
| `t.Fail()`       | Mark the test as failed, but continue test execution. |
| `t.FailNow()`    | Mark the test as failed, and immediately stop execution. |
| `t.Error()`      | Combination of `Log()` and `Fail()`. |
| `t.Errorf()`     | Combination of `Logf()` and `Fail()`. |
| `t.Fatal()`      | Combination of `Log()` and `FailNow()`. |
| `t.Fatalf()`     | Combination of `Logf()` and `FailNow()`. |


## Strategies

When testing file writes, use _goldenfiles_: files that contain the expected results and that you load during tests to validate output.

> **IMPORTANT**: Put goldenfiles, and other testing files, in a directory called `testdata`. Go tooling ignores this directory when building and compiling the program.

`iotest.TimeoutReader(r)` simulates a reading failure and returns a timeout error.

## Integration testing with external resources

If you are testing external commands that modify the state of an external resource, the testing conditions change after each test.