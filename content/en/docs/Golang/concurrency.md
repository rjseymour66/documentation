---
title: "Concurrency"
weight: 14
description: >
  Working with Concurrency in Go.
---

Go provides two strategies that maintain data integrity when you are writing concurrent programs:
- Locks, such as Mutex
- goroutines and channels

## Mutexes

`RLock()` blocks and waits if the associated object is locked for writing. This provides safe concurrent read and write operations while allowing multiple reads to improve performance.

## Channels

Channels allow goroutines to communicate with each other.

```go
// create a channel with make
ch := make(chan type)
// Use an empty struct to create a channel for done. done channels
// only signal that processing is complete, and an empty struct does not allocate
// any memory
done := make(chan struct{})
```

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

## Locks

The [Locker interface](https://pkg.go.dev/sync@go1.19.4#Locker) has methods that lock and unlock an object. Use this when you want to prevent concurrent access to an object during operations.

`&sync.Mutex` implements the Locker interface:
```go
mu := &sync.Mutex{}
```