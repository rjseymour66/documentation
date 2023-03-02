---
title: "Concurrency"
weight: 140
description: >
  Channels, Goroutines, Scheduling, and Synchronization.
---

Concurrency is about _dealing with_ a lot of things at once. Parallelism is about _doing_ a lot of things at once. When you think of concurrency, think of resources that are in a waiting state--waiting for a process to act upon the resource.

Go uses a _fork-join_ concurrency model. At any point during execution, the program can split off a child branch of execution that runs concurrently with its parent. At some point in the future, the parent and child threads synchronize and join back together. These join points are what guarantee the program's correctness and remove race conditions.

## Goroutines

Most programming languages achieve concurrency with kernel-space processes. Go uses _goroutines_, which are a lightweight thread of execution spawned from a user-space thread. Goroutines run concurrently alongside other code. They have their own call stack that is a few kilobytes, which is managed by the Go runtime scheduler. The scheduler distributes the goroutines over multiple operating system threads that run on one or more processors.

Specifically, goroutines are _coroutines_: concurrent subroutines--functions, closures, or methods--that cannot be interrupted (preemptive). Their behavior is managed by the Go runtime. The runtime observes the goroutine runtime and suspends them automatically whtn they block and then resumes them when they unblock.

Every program has at least one goroutine: the main goroutine (`main` method). To create a goroutine, use the `go` keyword before any named or anonymous function:

```go
func main() {
	go PrintNum(5)
}

// PrintNum logs to the console each number from 1 to n.
func PrintNum(n int) {
	for i := 0; i < n; i++ {
		fmt.Println(i)
	}
}
```
In the previous example, nothing logs to the console. Because goroutines run independently of the `main` method, `main` does not wait for the scheduler to run the goroutine, so it exits before `PrintNum` executes.

To synchronize the main goroutine and its child goroutines, you must use concurrency primitives. Go provides traditional, low-level concurrency primitives in the [sync package](#sync-package), and higher-level primitives with [channels]().

## sync package

### Waitgroups

Use a `Waitgroup` if you are not concerned about the following:
- Result of the concurrent operation.
- You can collect the result of the concurrent operation with other means.

A `WaitGroup` is a concurrent-safe counter. You can add goroutines to the `WaitGroup`, remove goroutines, and then wait for all goroutines that the `WaitGroup` tracks to complete. `WaitGroup` has the following methods to track goroutines:
- `<x>.Add(n)`: increments the number of goroutines that the `WaitGroup` tracks by `n`.
- `defer <x>.Done()`: decrements the number of goroutines that the `WaitGroup` tracks by 1.
- `<x>.Wait()`: Blocks program execution until the counter reaches zero.

For example, the following `WaitGroup` `wg` demonstrates all of these methods:
```go
func main() {

	var wg sync.WaitGroup

	wg.Add(4)

	go launchGoroutine(&wg, 1)
	go launchGoroutine(&wg, 2)
	go launchGoroutine(&wg, 3)
	go launchGoroutine(&wg, 4)

	wg.Wait()

	fmt.Println("All goroutines are done running...")
}

func launchGoroutine(w *sync.WaitGroup, n int) {
	defer w.Done()
	fmt.Printf("#%d goroutine is running\n", n)

}
```
The previous example produces the following output:

```shell
$ go run main.go 
#4 goroutine is running
#1 goroutine is running
#2 goroutine is running
#3 goroutine is running
All goroutines are done running...
```

Because the `.Add` method registers a goroutine with a WaitGroup, you cannot call it within a goroutine. If you called `.Add` in a goroutine, then the program might not register it because it might reach the `Wait` method first. The `Wait` method cannot block for a goroutine that has not yet started. You can place `Done` in the goroutine function becuase the program does not reach `Done` until it launches the goroutine.

## Mutex

_Mutex_ stands for "mutual exclusion", and it is a way mechanism that handles concurrency through memory access synchronization. A mutex provies a concurrent-safe way to provide exclusive access to shared resources. The developer must coordinate memory access with a mutex.

A mutex handles memory synchronization with the `Lock()` and `Unlock()` methods. The call to `.Lock()` represents the beginning of the critical section that requires memory synchronization. The call to `Unlock()` indicates that the program reached the end of the critical section:

```go
// example
```
Critical sections indicate a bottleneck--it is expensive to enter and exit a critical section. One strategy to mitigate the memory sharing is to use a `sync.RWMutex`. The `sync.RWMutex` gives you more control over the memory. It can request a lock for reading, unless the lock is being held for writing. An arbitrary number of readers can hold a reader lock as long as nothing else is holding a writer lock.

```go
// sync.Locker is an interface with a Lock and Unlock method, so
// any mutex satisfies it.
l sync.Locker

var m sync.RWMutex
m.RLocker()

```


## Channels

A channel is a high-level synchronization mechanism. Channels are composable, typed conduits that communicate information between goroutines. They have the following characteristics:
- Typed, and can send and receive values of that type only.
- Synchronous--the sender must wait for the receiver to finish before sending more data, and vice versa.
- Values are ordered with FIFO.
- [Buffered or unbuffered](#buffered-and-unbuffered-channels).
- They are directional. Channels can be bidirectional or unidirectional. Prefer unidirectional to prevent bugs and complexity.

### Creating channels

You can decalare a channel, or instantiate one with `make`:

```go
var chStream chan any
chStream = make(chan any)
```
> The `any` type replaced the empty interface (`interface{}`) in Go 1.18. It represents any type.

### Sending and receiving

There are bidirectional (send _and_ receive) and unidirectional (send _or_ receive) channels. Most channels are bidirectional--Go can implicitly convert a bidirectional channel to unidirectional when needed. It is common to see a unidirectional channel as a function parameter or return type. The placement of the `<-` operator determines whether the channel sends or receives information.

When the `<-` operator is to the left of the channel name, it is a receiving channel. The program reads or receives information from a read channel. You cannot send data into a receiving channel, you can only read from it:
```go
var recChan <-chan any
recChan = make(<-chan any)
```

When the `<-` operator is to the right of the channel name, it is a sending channel. The program sends information to a send channel:
```go
var sendChan chan<- any
sendChan = make(chan<- any)
```

The following example shows a bidirectional channel that is implictly converted to unidirectional, depending on the calling function:
```go
// SendChan sends a string into a channel.
func SendChan(ch chan<- string, s string) {
	ch <- s
}

// RecChan returns a string from a channel.
func RecChan(ch <-chan string) string {
	return <-ch
}

func main() {
	bidirChan := make(chan string)
	go SendChan(bidirChan, "It's a string!")
	fmt.Println(RecChan(bidirChan))
}
```

Output: 

```shell
$ go run main.go 
It's a string!
```
### Blocking channels

When you run goroutines with lower-level primitives from the sync package, you have to register them to a [WaitGroup](#waitgroups) to ensure that they run before the main method exits. You do not have to register goroutines with channels because they synchronize with the Go runtime to ensure they run to completion.

By default, channels are _unbuffered_--they do not have defined capacity. When you send data into an unbuffered channel, the Go runtime blocks until there is a channel on another goroutine that can receive the data. A buffered send channel (`ch <-`) accepts data only if there is a corresponding receive channel (`<-ch`) ready. 

To demonstrate, the following example uses an unbuffered channel that results in a deadlock:
```go
func main() {
	unbufChan := make(chan string)
	unbufChan <- "Channel information!"
	fmt.Println(<-unbufChan)
}
```
The sending `unbufChan` blocks because it is unbuffered. The program needs a concurrent thread of execution that has a channel to receive the `"Channel information!"` string.

To fix this, create a goroutine as an anonymous function and send the string into `unbufChan`. The receive channel (`<-unbufChan`) blocks until the goroutine places a value in the channel, and then the program proceeds:

```go
func main() {
	unbufChan := make(chan string)
	go func() {
		unbufChan <- "Channel information!"
	}()
	fmt.Println(<-unbufChan)
}
```

Output:
```shell
$ go run main.go 
Channel information!
```
Any goroutine that attempts to write to a channel that is full blocks until the channel is emptied (read from). Any goroutine that attempts to read from a channel that is empty waits until at least one item is placed on it.

### Closing channels

Close a channel to signal that no more values will be sent over the channel. Use the `close` keyword:
```go
close(chanName)
```
You can read from a closed channel--this to support multiple downstream reads from a single upstream writer on a channel. A read from a closed channel returns multiple values so you can determine if the read value was placed on the channel by a writer in the process, or if it is a default value generated from a closed channel. For example:

```go
func main() {
	upstreamStream := make(chan string)
	go func() {
		upstreamStream <- "Write to open channel!"
	}()
	value, ok := <-upstreamStream
	fmt.Printf("(%v): %v\n", ok, value)
}
```
Output: 

```shell
$ go run main.go 
(true): Write to open channel!
```

When you read from a closed channel, Go returns `false` and the zero type for the channel:

```go
func main() {
	upstreamStream := make(chan int)
	go func() {
		upstreamStream <- 10
	}()
	close(upstreamStream)
	value, ok := <-upstreamStream
	fmt.Printf("(%v): %v\n", ok, value)
}
```
Output: 
```shell
$ go run main.go 
(false): 0
```








# Beginning other book

```go
// Use an empty struct to create a channel for done. done channels
// only signal that processing is complete, and an empty struct does not allocate
// any memory
done := make(chan struct{})
```

Go provides two strategies that maintain data integrity when you are writing concurrent programs:
- Locks, such as Mutex
- goroutines and channels

## Mutexes

`RLock()` blocks and waits if the associated object is locked for writing. This provides safe concurrent read and write operations while allowing multiple reads to improve performance.


## WaitGroups

A WaitGroup is a mechanism that coordinates the goroutine execution. When you create a goroutine, add 1 to the WaitGroup. When a goroutine finishes, subtract 1 from the WaitGroup. Use `Wait()` to wait until all goroutines are finished so you can complete execution.

```go
wg := sync.WaitGroup{}
```

## Scheduling contention and worker queues

This is when you create too many goroutines and they compete for work. The answer is to use worker queues.

When using worker queues, you create one goroutine per available CPU, and have another goroutine send jobs to be executed by the workers. So, the CPUs are the workers.

Use `runtime.NumCPU()` to determine the number of available CPUs:


## Goroutines

Because goroutines run independently of the `main()` function, go uses `WaitGroups`, a mechanism that blocks the `main()` method until all goroutines complete.

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

## Locks

The [Locker interface](https://pkg.go.dev/sync@go1.19.4#Locker) has methods that lock and unlock an object. Use this when you want to prevent concurrent access to an object during operations.

`&sync.Mutex` implements the Locker interface:
```go
mu := &sync.Mutex{}
```