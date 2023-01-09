---
title: "Basic data structures"
weight: 1
description: >
  Descibes the basic data structures in Go.
---

## Formatting verbs

[Formatting verbs](https://pkg.go.dev/fmt#hdr-Printing)

For errors, use `%w` to decorate the original error with additional information for the users. Essentially, you can customize the error message while also returning the default Go error:
```go
if err != nil {
    return nil, fmt.Errorf("Cannot read data from file: %w", err)
}
```
For more info, read [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-add-extra-information-to-errors-in-go).

## Idioms

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

## Arrays

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

## Slices

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
## Maps

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