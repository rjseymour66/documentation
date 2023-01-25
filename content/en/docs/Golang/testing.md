---
title: "Testing"
weight: 9
description: >
  Working with Testing in Go.
---


## Tests

`iotest.TimeoutReader(r)` simulates a reading failure and returns a timeout error.

### Integration tests

Integration tests test how the program behaves when interacted with from the outside world--how a user uses it. This means that you test the `main()` method.

In Go, you test the main method with the `TestMain()` function so you can set up and tear down resources more easily. For example, you might need to create a temporary file or build and execute a binary. You do not want to keep these artifacts in the program after testing.

Follow these general guidelines when running integration tests:
1. Check the machine with `runtime.GOOS`
2. Create the build command with `exec.Command()`, then use `.Run()` to execute that command. Check for errors
3. Run the tests with `m.Run()`
4. Clean up any artifacts with `os.Remove(artifactname)`

### General flow

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

### Test helpers with t.Helper()

Add `t.Helper()` to mark a function as a test helper.

You can defer a the cleanup function. If a helper function fails a test, Go prints the line in the test function that calls the helper function.

You can also return a cleanup function to make sure that your tests leave no test artifacts in the filesystem. There is also a `t.Cleanup()` method that registers a cleanup function.

### Subtests with t.Run()

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

### Package naming

Place `*_test.go` files in the same directory as the code that you are testing. When you declare the `package` in the test file, use the original package name followed by `_test`. For example:

```go
package original_test
```

### Table tests

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

### Utilities

Create a temporary file if you need to test an action like deleting a file from the file system. Use `os.CreateTemp()`. Be sure to clean up with `os.Remove(tempfile.Name())`:

```go
os.CreateTemp(".", )
```

### Error handling

The test object (`*testing.T`) provides the following methods troubleshoot during testing

`t.Fatalf()` logs a formatted error and fails the test, then stops test execution:
```go
t.Fatalf("Error message: %s", err) // Logf() + FailNow()
```

`t.Errorf()` logs a formatted error and fails the test, but continues test execution:
```go
t.Errorf("Error message: %s", err) // Logf() + Fail()
```

### Strategies

When testing file writes, use _goldenfiles_: files that contain the expected results and that you load during tests to validate output.

> **IMPORTANT**: Put goldenfiles, and other testing files, in a directory called `testdata`. Go tooling ignores this directory when building and compiling the program.

### Integration testing with external resources

If you are testing external commands that modify the state of an external resource, the testing conditions change after each test.