---
title: "Basic types"
weight: 7
description: >
  Working with basic types in Go.
---

## Slices

The slice type is a dynamic array that can grow and shrink as you see fit. 

> **Slice vs. Array**
> If you specify a value inside the [] operator ([4]varName), then you are creating an array. If you do not specify a value, you create a slice.
> `array := [3]int{10, 20, 30}`
> `slice := []int{10, 20, 30}`
> 
> An _array_ is a value. They are of a fixed size, and the elements in the array are initialized to their zero value.
> 
> A _slice_ is a pointer to an array. It has no specified length, and its zero value is `nil`.
> 'Slicing' does not copy the slice's data, it creates a new slice value that points to the original array. So `s = slice[2:]` creates slice `s` that begins with second element of `slice`, and they both point to the same underlying data structure. So, modifying elements of a slice changes the  


### Internals

The slice has three fields:
- Pointer to the underlying array
- Length. Number of elements the slice can access from the array.
- Capacity. Size of the underlying array, or number of elements that the slice has available for growth. It is the 

You cannot create a slice with a capacity that's smaller than the length.

### Create slices 

Create slices using the following methods:
- `new()` function.
- `make([]T, len, cap)` function.
- _Slice literals_. This is the idiomatic way to create slices. It requires that you define the contents when you create the slice.
- _nil slice_. A nil slice is declared without any initialization.  The most common way to create slices, and can be used with many of the standard library and built-in functions that work with slices.
- _empty slice_. Useful when you want to represent an empty collection, such as when a database query returns zero results.

```go
// make
slice := make([]string, 5)          // create a slice of strings with 5 capacity
slice := make([]int, 3, 5)          // length 3, cap 5

// slice literals
slice := []int{10, 20, 30}          // slice literal
slice := []string{99: ""}           // initialize the index that represents the length and capacity you need

// nil slice
var slice []int                     // nil slice

// empty slice
slice := make([]int, 3)             // empty slice with make
slice := []int{}                    // slice literal to create empty slice of integers
```

### Functions

_copy(dest, src)_ 
: Copies the contents of the `src` slice into the `dest` slice. The `dest` and `src` slices must share the same underlying array. This is commonly used to increase the capacity of an existing slice.
: If `dest` and `src` are different lengths, it copies up to the smaller number of elements. 

_append(slice, value)_
: Appends `value` to the end of `slice`.
: When there’s no available capacity in the underlying array for a slice, the `append` function creates a new underlying array, copies the existing values that are being referenced, and assigns the new value. So, if you append to the 3rd index of a slice with length 2, you get a new underlying array of length 3 with a capacity doubled the original array.

_len(slice)_
: Returns the length of `slice`.

_cap(slice)_
: Returns the capacity of `slice`. When you expand a slice beyond its original capacity, the capacity is always doubled when the existing capacity of the slice is under 1,000 elements. 



```go
slice := []string{"Apple", "Orange", "Plum", "Banana", "Grape"}

capacity := cap(slice)              // 5
length := len(slice)                // 5


slice = append(slice, "Kiwi")
capacity = cap(slice)               // 10
length = len(slice)                 // 6

```

### Slicing (Working with indices)

Use the following syntax to 'slice' a slice into a new slice:

`slice`[_start_:_end_:[ _capacity_] ]

_start_
: Inclusive. The index position of the element that the new slice begins with.

_end_
: Non-inclusive. The index of the existing slice where you stop copying values to the new slice. The new slice ends with the value at `[end-1]`.

_capacity_
: The capacity for the new slice. When you append to a slice that goes beyond the slice capacity, the capacity is doubled when its length is less than 1,000.

#### Calculating slices

The start, end, and capacity values have a formula that you can use to correctly calculate slicing. For example, the following slice creates a new slice with 1 element, and the capacity of 2:

```go
newSlice := slice[2:3:4]
```
| Value | Description | Formula |
|:------|:------------|:--------|
| 2     | start. The first element in the original slice, inclusive. | |
| 3     | end. The index of the existing slice where the slicing stops, non-inclusive. | start + number of elements you want in the slice. |
| 4     | capacity. Size of the new slice. | start + number of elements to include in the capacity. | 

#### Examples

```go

slice := []int{10, 20, 30, 40, 50}
newSlice := slice[:4]               // newSlice == {slice[0], slice[1], slice[2], slice[3]}.
newSlice := slice[1:3]              // {20, 30}
newSlice[1] = 31                    // {20, 31}

// start and end indices
s := []int{0, 1, 2, 3, 4}
l := s[1:3]
fmt.Println(s, l)   // [0 1 2 3 4] [1 2]

l[0] = 8
fmt.Println(s, l)   // [0 8 2 3 4] [8 2]

// Slicing length and capacity
slice := []int{10, 20, 30, 40, 50}  // Length:   5
                                    // Capacity: 5

newSlice := slice[1:3]              // Length:   3 - 1 = 2
                                    // Capacity: 5 - 1 = 4



// Slice the third element and restrict the capacity.
// Contains a length of 1 element and capacity of 2 elements.
slice := source[2:3:4]
```

You can use the built-in function called append, which can grow a slice quickly with efficiency. You can also reduce the size of a slice by slicing out a part of the underlying memory

### Variadic slices:

The built-in function append is also a variadic function. Use the ... operator to append all the elements of one slice into another.

```go
s1 := []int{1, 2}
s2 := []int{3, 4}

// Append the two slices together and display the results.
fmt.Printf("%v\n", append(s1, s2...))

Output:
[1 2 3 4]
```

Use the `...` operator to expand a slice into a list of values:

```go
// accepts any variable number of string args
func getFile(r io.Reader, args ...string) {}
..
// the ... operator expands a slice into a list of values
t, err := getFile(os.Stdin, flag.Args()...) {}
```

### Iterating over slices

Use `for` with `range` to iterate over slices from the beginning.

> **IMPORTANT**
> Do not use pointers (`&value`) when you iterate with `range` because it returns the index and a copy of the value for each iteration, not a reference. A pointer is an address that contains the copy of the `value` that is being copied.

```go
for index, value := range <slice-name> {
    fmt.Printf("index: %d, value: %d", index, value)
}

// discard the index with a '_'
for -, value := range <slice-name> {
    fmt.Printf("value: %d", value)
}
```

To iterate over a slice from an index other than 0, use a traditional `for` loop:

```go 
for i := 2; i < len(slice); i++ {
    fmt.Printf("index: %d, value: %d", index, slice[i])
}
```
### Passing slices between functions

Pass slices by value, because slices only contain a pointer to the underlying array, its length, and capacity. This is why slices are great--no need for passing pointers.

On 64-bit machines, each component of the slice requires 8 bytes (24 total).

```go
bigSlice := make([]int, 1e9)

slice = fName(slice)

func fName(slice []int) []int {
    return slice
}
```