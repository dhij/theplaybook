---
title: "Demystifying Goroutines - Channels, WaitGroup, Cancellation"
date: 2023-09-09T13:51:42-04:00
draft: false
tags: ["golang", "goroutine", "go channel"]
categories: ["go"]
---

If you have rough ideas on how goroutines work but never took the time to learn the idiomatic approach and why we want to run goroutines certain way, this blog will be of help. In this post, I will be summarizing what I have learned from `Chapter 8: Goroutines and Channels` of the [Go Programming Language](https://amzn.to/45LKKfM) book as well as [The Go Blog on Go Concurrency Patterns](https://go.dev/blog/pipelines).

This blog post will be helpful if you are not familiar with any of the below:
- How to wait for the other goroutines to finish instead of exiting on the main goroutine
- Unbuffered vs. buffered channels and the pros & cons of each
- Running loops in parallel (goroutines inside the loop)
- Unbounded parallelism & semaphores
- Multiplexing with `select` & goroutine cancellation

We will also answer some of the questions such as:
- What is a goroutine leak and how do we prevent it?
- Do we need to close all the channels and goroutines?
- Why do we want to have our waitGroup in its own goroutine?   

You should already know that a goroutine is an activity of a concurrent Go program and channel is a communicating mechanism between the goroutines. We all wish running a concurrent program is as simple as prepending `go` to a method. However, there is often a little more we need to make goroutines work as intended. Let's have a look at different use cases starting from the first bullet point on the list above. 

### How to wait for other goroutines to finish instead of exiting on the main goroutine

```golang
func main() {
    // ...
    done := make(chan struct{})
    go func() {
        // ...
        done <- struct{}{} // signal the main goroutine
    }()
    // ...
    <- done // wait for background goroutine to finish 
}
```

In the code snippet above, we define an unbuffered channel called `done`. The channel is unbuffered because we are not assigning any capacity/size to the channel. We will take a look at buffered channels in another example later. The main characteristic of an unbuffered channel is that it blocks until another goroutine executes a corresponding receive on the same channel, causing the sending and receiving goroutines to be "synchronous". With the unbuffered channel in the example above, we can ensure that the program waits for the background goroutine to finish before exiting. You can find the code example from the book [here](https://github.com/adonovan/gopl.io/blob/master/ch8/netcat3/netcat.go#L17-L31).

### Unbuffered vs. buffered channels

Let's have a look at an example from the book where an unbuffered channel might lead to a problem.

```golang
func mirroredQuery() string {
	responses := make(chan string)
	go func() { responses <- "response 1" }()
	go func() { responses <- "response 2" }()
	go func() { responses <- "response 3" }()
	
	return <-responses // return the quickest response
}
```

With unbuffered channels, there might be a scenario like the above where the two slower goroutines (as to which of the three would be the fastest is non-deterministic) would be stuck trying to send their responses on a channel from which no goroutine will ever receive. This is known as a `goroutine leak` and it is important to ensure that the goroutines terminate themselves when no longer needed since leaked goroutines are not automatically collected.

However, we would not encounter a `goroutine leak` if we use a buffered channel instead as such: `responses := make(chan string, 3)` where we assign a capacity of 3 to the channel. That buffered channel would hold up to three string values and block until a space is made available by another goroutine's receive. And even if the sending channel closes, the values queued up in the channel can be handled later by the receiving channels.

Although unbuffered channels provide stronger synchronization guarantees, it would require prudence on our end to avoid goroutine leaks. If the synchronization does not matter as much and you know the upper bound on the number of values that will be sent on the channel, a buffered channel could be a better option. However, it is worth noting that failure to allocate sufficient buffer capacity would cause the program to deadlock.

As we looked at an example of a goroutine leak, you might have also wondered if we have to close all the channels as well. The answer is that you do not need to close every channel unless it is important to tell the receiving goroutines that all data have been sent. Channels that the garbage collector determines to be unreachable will have its resources reclaimed whether or not it is closed. However, closing an already-closed channel causes a panic, as does closing a nil channel. 

### Running loops in parallel

At first glance, looping in parallel might appear to be as simple as:

```golang
func parallelLoop() {
	for _, f := range filenames {
		go func(f string) {
			// ...
		}(f)
	}
}
```

However, when you run the function above, you will observe that the function exits almost immediately. The above doesn’t work because `parallelLoop()` returns before it has finished all its work. We have to change the inner goroutine to report its completion to the outer goroutine by sending an event on a shared channel as below:

```golang
func parallelLoop() {
	ch := make(chan struct{})
	for _, f := range filenames {
		go func(f string) {
			// ...
			ch <- struct{}{}
		}(f)
	}

	for range filenames {
		<-ch
	}
}
```

One other thing worth highlighting is that the `f` is passed into the goroutine as an explicit argument as such: `go func(f string){...}(f)` and not directly used in the goroutine. Explicit parameters are used for goroutines in a loop to ensure that we use the value of `f` that is current when the go statement is executed.

However, the above example does not take error handling into consideration. Now let’s have a look at the idiomatic approach to looping in parallel using the `sync.WaitGroup` and handling errors appropriately:

```golang
func makeThumbnails(filesnames <-chan string) int64 {
	sizes := make(chan int64)
	var wg sync.WaitGroup
	for f := range filenames {
		wg.Add(1)
		go func(f string) {
			defer wg.Done()
			// run operations to retrieve the size of the file
			sizes <- size
		}(f)
	}

	// closer
	go func() {
		wg.Wait()
		close(sizes)
	}

	var total int64
	for size := range sizes {
		total += size
	}

	return total
}
```

Above is a snippet of code where we retrieve the size of different files in parallel and compute the total size. The example used in the book can be found [here](https://github.com/adonovan/gopl.io/blob/master/ch8/thumbnail/thumbnail_test.go#L117-L146). There are a few things we want to pay attention from the above:

1. `wg.Add(1)` must be called before the worker goroutine starts, not within it—this ensures that the `Add` happens before the closer goroutine calls `Wait`
2. defer is used on `wg.Done()` to ensure that Done is called even in the error cases
3. The closer goroutine that waits for the workers must be created before the closing of the sizes channel
4. The closer goroutine must be concurrent with the loop over sizes
    - If the wait operation was placed before the loop in the main goroutine: it would never end
    - If the wait operation was placed after the loop in the main goroutine: the loop would never terminate because there is nothing closing the `sizes` channel and the wait operation will be unreachable

### Unbounded parallelism

If there is a limiting factor in the system, such as the number of CPU cores, the number of spindles and heads for local disk I/O operations, or the bandwidth of the network, we want to limit the number of parallel uses of the resource to match the level of parallelism that is available.

We can limit parallelism using a buffered channel of capacity `n` to model a concurrency primitive called a `counting semaphore`. Conceptually sending a value into the channel acquires a token and receiving a value from the channel releases the token, ensuring that at most n sends can occur without an intervening receive. Let's have a look at an example:

```golang
var tokens = make(chan struct{}, 20)
func doSomething() {
	tokens <- struct{}{} // acquire the token
	// ..
	<-tokens // release the token
}
```

Alternatively, you can use the golang [semaphore package](https://pkg.go.dev/golang.org/x/sync/semaphore) and call `Acquire` and `Release` on the semaphore that is equivalent of the token concept above. 

```golang
var sem = semaphore.NewWeighted(int64(10))

sem.Acquire(ctx, 1) // equivalent to sem <- 1 (using channel approach)
sem.Release(1) // equivalent to <- sem (using channel approach)
```

There is a blog post that covers semaphore in greater detail [here](https://medium.com/@deckarep/gos-extended-concurrency-semaphores-part-1-5eeabfa351ce).

### Multiplexing with select & cancellation

The select statement comes in handy when we need to wait for an event on one of the many channels. The select statement can be used with a ticker to run a loop every n seconds/minutes/hours as below:

```golang
ticker := time.NewTicker(5 * time.Second)

go func() {
	for ... {
		// some operations
		select {
		case <- ticker.C:
		case <- done:
			return
		default:
		}
	}
}
```

Notice how we have a case for a receive operation on a `done` channel. That is useful when want the main goroutine to tell the other goroutines (above goroutine in this case) to abandon the values they are trying to send. Otherwise those goroutines with work left will be stuck trying to send their responses on a channel from which no goroutine will ever receive, leading to resource leak as mentioned earlier. In order for that to work, we will need to close the done channel at the end of the `main()` as done below:

```golang
func main() {
    // Set up a done channel that's shared by the whole pipeline,
    // and close that channel when this pipeline exits, as a signal
    // for all the goroutines we started to exit
    done := make(chan struct{})
    defer close(done)          

		// some operations
		go func(done <-chan struct{}){
			...
		}(done)

    // done will be closed by the deferred call
}
```

Here are a few other details about the multiplexing with the `select` statement that you might find useful:
- If multiple cases are ready, `select` picks one at a random.
- If there is a case in the select statement where the channel can optionally be nil (depending on the flag passed in), the case is effectively disabled

I hope this post has provided some context for you to get started with concurrency in Go. If you would like to learn more about them, make sure you check out `Chapter 8: Goroutines and Channels` of the [Go Programming Language](https://amzn.to/45LKKfM) book as well as [The Go Blog on Go Concurrency Patterns](https://go.dev/blog/pipelines).