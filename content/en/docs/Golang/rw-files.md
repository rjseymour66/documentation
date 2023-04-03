---
title: "Reading, writing, files"
weight: 50
description: >
  Working with files. Reading and writing data, too.
---

## Reading files

Read data from a file with the `os` package. `ReadFile` reads the contents of a file and returns a `nil` error:
```go
fileContents, err := os.ReadFile("filename")
	if err != nil {
		if errors.Is(err, os.ErrNotExist) {
			return nil
		}
		return err
	}
if len(fileContents) == 0 {
    return nil
}
```
The previous example returns `nil` if the file does not exist. If your program creates a file after this code runs--for example, with `os.WriteFile`--then the program returns `nil` and continues.

### Scanner for lines and words

The Scanner type accepts an `io.Reader` and reads data that is delimited by spaces or new lines.

By default, it reads lines, but you can configure it to read words:

```go
scanner := bufio.NewScanner(r)
// scan words
scanner.Split(bufio.ScanWords)
```
Use the `.Scan()` function with a default scanner to read a single line. Check the `.Err()` value, then return the text with the `.Text()` function:

```go
s.Scan()

if err := s.Err(); err != nil {
    return "", err
}

if len(s.Text()) == 0 {
    return "", fmt.Errorf("Input cannot be empty")
}

return s.Text(), nil
```

Use `.Scan()` in a for loop to read lines or tokens, depending on the `.Split()` configuration:
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









## Buffers and bytes

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



## Writing files

Write data to a file with the `os` package. `WriteFile` writes to an existing file or creates one, if necessary:

```go
os.WriteFile(name string, data []byte, perm FileMode) error

```
> **Linux permissions**: Set Linux file permissions for the file owner, group, and all other users (`owner-group-others`). The permission options are read, write, and execute. Use octal values to set permssions:  
  read: 4  
  write: 2  
  execute: 1  

When you assign permissions in an programming language, you have to tell the program that you are using the octal base. You do this by beginning the number with a `0`. So, `0644` permissions means that the file owner has read and write permissions, and the group and all other users have read permissions.

### io.Writer

Commonly named `w` or `out`. Examples of `io.Writer`:
- os.Stdout
- bytes.Buffer (implements `io.Writer` (and `io.Reader`) as a pointer receiver, so use `&`)
- files (type os.File implements `io.Writer`)
- gzip.Writer

> Use a file or `os.Stdout` in the program, and `bytes.Buffer` when testing.

### Archiving files

The following is a GZIP writer example:
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


### tabWriter

`tabwriter.Writer` writes tabulated data with formatted columns with consistent widths using tabs (`\t`).

https://pkg.go.dev/text/tabwriter#pkg-overview

## Temp files


If you need to create a temp file:
```go
temp, err := os.CreateTemp("", "pattern*.extension")
if err != nil {
    // handle error
}
defer temp.Close()
defer os.Remove(temp.name())
```
- First parameter is the directory that you want to create the temporary file in. If it is left blank, it uses the `/tmp` directory.
- The second parameter defines the file name. Use a `*` character to tell the program to generate a random number to make the name unique.

Always close and remove temp files with `defer`, unless you are creating a test helper. For test helpers, see `t.Cleanup()`.


## Logging

Use logs to provide feedback for background processes. To create a logger, you need to create:
- [*log.Logger](https://pkg.go.dev/log#Logger) type
- Logging destination ([w io.Writer](#interfaces))

By default, Go's `log` library sends messages to STDERR, but you can configure it to write to a file. It adds the date and time to each log entry, and you can add a prefix to the string to help searchability
```go
l := log.New(io.Writer, "LOGGER PREFIX: ", log.LstdFlags)
```
log.LstdFlags uses the default log flags, such as date and time.

## File operations

#### Get file name

Get the name of the file:
```go
name := fileName.Name()
```
#### File extension

Return the file extension with `filepath.Ext(file)`:
```go
ext := filepath.Ext(file)
```
#### Delete a file

Remove a file:
```go
if err := os.Remove; err != nil {
	// handle err
}
```

### File metadata

Use [fs.FileInfo](https://pkg.go.dev/io/fs#FileInfo) to examine file metadata, such as the name, length in bytes, if it is a directory, etc. To return the `FileInfo` file attributes for a file, use `os.Stat(filename)`:
```go
info, err := os.Stat(fileName)
if err != nil {
	// handle error
}
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

## Directory paths

The `filepath` library creates cross-platform filepaths--they compile correctly for each supported OS.

Get the relative directory path between a root and target path:
```go
relDir, err := filepath.Rel(root, filepath.Dir(path))
if err ...
```

Extract the last element in a filepath--generally, the file--with the filepath.Base() function:
```go
filename := filepath.Base("/path/to/home.html")
// filename == home.html
```