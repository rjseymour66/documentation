---
title: "Reading data"
weight: 5
description: >
  Working with files in golang.
---

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

#### Opening a file

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

## Reading data

#### Reading from a file

Read data from a file with the `os` package. `ReadFile` reads the contents of a file and returns a `nil` error:
```go
os.ReadFile(filename)
```

Extract the last element in a filepath--generally, the file--with the filepath.Base() function:
```go
filename := filepath.Base("/path/to/home.html")
// filename == home.html
```

#### Scanner for lines and words

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
#### Reading CSV data

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