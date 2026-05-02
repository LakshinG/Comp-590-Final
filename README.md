Comp-590 Final Project Concurrency Models 

Part 1: The Model Matrix
Erlang style Actor Model in Java
Java’s native threads and the java.util.concurrent package give us good tools to start with since we can easily create a dedicated thread to act as an isolated process, and use a LinkedBlockingQueue to handle asynchronous message passing. Because Java threads can block and wait on a queue indefinitely without going through CPU cycles, this setup naturally mimics how an Erlang process waits for new messages in its mailbox.
What requires simulation here is that the biggest issue or problem here is recreating Erlang’s OTP supervision model. In Java, there’s no built in framework to automatically monitor and restart threads when they fail. So the entire supervisor has to be built from scratch. This means writing explicit logic to track thread lifecycles, handle restarts, and ensure the state is properly wiped and initialized every single time.
```java
// Simulating an Actor's mailbox and processing loop
BlockingQueue<Object> mailbox = new LinkedBlockingQueue<>();
new Thread(() -> {
    while (true) {
        try {
            Object message = mailbox.take(); // Blocks until message arrives
            handleMessage(message);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}).start();
```

While this gets the basic loop working, forcing the programmer to manually wire up fault tolerance introduces a ton of complexity and a high risk of race conditions during teardown.
What cannot be represented well is the whole let it crash philosophy does not go well with how Java itself is built to work. Java is made around catching exceptions locally and fixing the problem exactly where it happens. We have a strict checked exception system that constantly forces you to handle errors instead of just abandoning the state. On top of that you can never guarantee true memory isolation. Since everything runs in a shared heap, object references can easily leak between threads, which completely breaks the Actor model's core functionality of not having shared mutable state. 

2. Go style CSP/Channels in Java
Java’s concurrency library makes basic channel communication pretty straightforward to pull off. A java.util.concurrent.BlockingQueue works great as an approximation of a buffered Go channel. The queue naturally handles the blocking behavior you need when a buffer fills up or empties out. Java’s SynchronousQueue is definitely the best way to copy a Go style unbuffered channel. It forces a point where the sender has to wait until a receiver is actually there to grab the data, which is exactly how Go's unbuffered channels work.
What requires simulation here is Go's select statement which lets a goroutine wait on whichever of multiple channels is ready first isn’t a clean process. Java just doesn't have a direct equivalent to this so you either have to use continuous polling which wastes CPU and introduces lag or build a shared multiplexing queue.  

```java
// Simulating a buffered channel
BlockingQueue<Integer> ch = new ArrayBlockingQueue<>(10);
ch.put(1); // blocks if full
Integer val = ch.take(); // blocks if empty
```

If we try to use multiplexing to simulate select, then you would completely lose the strict directional flow that makes Go channels safe. We also have to get rid of type safety to allow for different kinds of messages into the single queue which defeats the purpose of typed channels.
Go's rules can't be accurately represented well for simultaneous readiness in a select block. When multiple channels are ready at the same time Go's runtime explicitly handles fair pseudo random selection among them. Implementing this kind of fair choice algorithm guarantee in user space Java requires complicated custom locking mechanisms that would reduce performance and ruin the simplicity of the CSP model.

3. Erland Style Actor Model in Go
Goroutines are great for this because they act perfectly as lightweight isolated processes. They mirror the low memory footprint of Erlang processes, and you can create thousands of them without a problem. If you strictly use Go channels in a unidirectional way, they do a great job passing messages asynchronously between these isolated actors. Go's runtime scheduler is also highly optimized for handling tons of blocking goroutines at once
Erlang mailboxes require simulation since they are varied and use pattern matching for selective receive. To simulate a mailbox that can accept any data type in Go, you are forced to use an empty interface channel.

```go
// Simulating mailbox
mailbox := make(chan interface{}, 100)
func process(mailbox chan interface{}) {
    for msg := range mailbox {
        switch v := msg.(type) {
        case int:
            // process int
        default:
            // put it back or hold in temporary slice
        }
    }
}
```

Selective receive requires building a custom scan loop. Because Go channels are strictly FIFO, skipping a non matching message means it needs to be popped off the channel and manually stash it somewhere else. This tanks the performance of it to O(n) relative to the mailbox size.
What cannot be represented well is the type safety. By using chan interface to accept different message types, you completely throw away Go's compile time type safety. Also the best part of Erlang's VM level pattern matching for messages. Instead of clean syntax there is a lot of wordy and error prone type assertions cluttering up the logic.

4. Java style Shared Memory in Go
Because all goroutines share a single address space, the shared memory model maps completely natively. Go provides the sync package, which includes sync.Mutex and sync.RWMutex. This lets you lock down mutable state exactly the same way you would using synchronized blocks or explicit locks in Java.

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock() // Safely coordinates 
    c.v[key]++
}
```
The way you manage state is basically the same as in Java, where you just pick out your shared variables and make sure every single access is wrapped in a sync block. From a performance standpoint, Go’s using specific mutexes are really fast and give you that direct, low latency memory access that we could get from a shared memory model
Technically, nothing requires simulation for this one. The mechanisms are fully native, and Go's underlying memory model provides the exact same happens before guarantees for locks that Java does.
While the code runs, it heavily violates Go's design philosophy which is what we can’t represent well here. Even though it works, this style goes against the standard Go philosophy of communicating to share data. The language really wants you to use channels for coordination, so using shared memory and locks feels a bit anti pattern in this ecosystem. Just like in Java, the language doesn't magically prevent unsynchronized access natively, so you're still completely vulnerable to deadlocks and race conditions if you mess up your lock discipline.

5. Go-style CSP/Channels  in Elixir
The BEAM virtual machine makes this pretty easy on the surface. Both the spawn or GenServer have very lightweight, isolated processes that map perfectly to the concept of goroutines. Both Go and Elixir are designed around and to handle thousands of tiny units running at the same time, just passing little bits of data back and forth. BEAMS' sceduler handles these tiny processes with the exact same kind of efficienty that the GO runtime scheduler handles goroutines, making the build and teh way these two models work feel very similar.
What requires simulation here is blocking which is the main issue. Unbuffered Go channels use "rendezvous synchronization" which is where the sender literally stops and blocks until the receiver is ready. Elixir’s message passing is completely asynchronous so it sends the signal then forgets it. To simulate that synchronous block, we would have to build a strict call/reply mechanism.

```elixir
# Simulating a synchronous, blocking channel send via GenServer
GenServer.call(receiver_pid, {:send_channel, data})
```

The sender has to actively block and wait for a {:reply, ...}. This simulates the blocking nature of a Go channel, but it ties up the calling process and artificially throttles the throughput of the BEAM.
Go's select block uses random selection when multiple channels are ready at once. Elixir's receive block doesn't work like that at all which is what cannot be represented well. It looks at messages strictly sequentially from top to bottom based on pattern matching. You cannot cleanly represent true randomized selection of simultaneous events natively. Simulating it would mean pulling the whole mailbox into memory, shuffling it, and processing it out of order, which is incredibly slow and goes against erlang design principles.  
6. Java Style Shared Memory in Elixir
Absolutely nothing maps naturally here. The BEAM virtual machine strictly isolates processes, so there is no memory is ever shared between them. That lack of shared memory is not something that can be added on, it is more of a hard runtime constraint. Trying to find a Java way of doing this isnt possible, since the BEAM was built from the ground up to keep processes isolated. 
Here we have to simulate shared memory using an Agent.
```elixir
# Simulating shared mutable state using an Agent
{:ok, shared_state} = Agent.start_link(fn -> 0 end)

# "Mutating" the state
Agent.update(shared_state, fn state -> state + 1 end)
```

The way the Agent works is by wrapping the state inside a dedicated, isolated process and separating all access requests with sequential message passing. When testing concurrent designs like the comparing actor style process rings in Elixir to threaded environments in Java, the simulation costs become obviously large. The latency of a simple state mutation turns from a localized quick lock acquisition in Java to a full microsecond inter process message round trip on the BEAM. Even though that doesn’t seem like a big difference, in high contention scenarios this serialization significantly limits system throughput.
The main thing that defines the shared memory model is getting fast, direct access to the same memory space, which we don’t see here. An Agent is used to trick whoever is running it into thinking they have shared state, but actually it’s really just the Actor model pretending to be something else. You just can't force a shared memory model onto a virtual machine that was literally built to ban it so it could have less faults and errors.

Part 2: Best and Worst Judgments
2a. Best overall fit 
The best overall representation is the Java Style Shared Memory in Go. My reason for this being the best fit is that the language does not have to pretend to be something else and we do not need workarounds for it. Moreover, the retention of full performance and type safety only adds to this. Go easily wins because it uses a single shared address space and natively provides the exact synchronization primitives (sync.Mutex) needed to recreate Java's concurrency model. There is no need to write custom scan loops, there is no need to poll for messages, and you don't have to throw away compile time safety like in some other cases. The coordination tools map perfectly, allowing you to manage mutable state exactly as you would in Java.
A counterargument that might be said here is that using shared memory with locks is strongly discouraged in the Go. However, these types of preferences are just a social constraint and not a technical limit. The actual workings of shared mutable state are fully intact and supported directly by Go's runtime. This makes it the most technically accurate translation on the matrix.

2b. Worst overall fit
The worst overall representation is Java Style Shared Memory in Elixir.  My reason for this being the worst fit is that there is a very important and unfixable runtime constraint that essentially just breaks the architectural nature of the target model. By this rule Elixir is the worst fit because the BEAM virtual machine strictly isolates processes and explicitly bans shared mutable state.
There could be an argument that simulating shared data using an Agent does good enough but it's really just an illusion of that in place. An Agent serializes access by passing messages, which means you aren't accessing shared memory at all and that you are just sending messages back and forth to a single process that can only handle one request. This simulation destroys the whole point of the Java model which is to be direct and have high speed memory access. Replacing it with massive inter process communication lag. It might look like it works but it completely misses the physical reality of the original model

Part 3: Synthesis
The difficulty of forcing these concurrency models into foreign languages shows that they aren't just differing APIs. They represent fundamentally different philosophies about where bugs come from and how a system should handle danger. How portable a model is usually comes down to whether it relies on lightweight language syntax or deep, baked-in runtime architecture. When a model is tied strictly to the runtime, trying to port it exposes massive structural roadblocks.  
Models are controlled by strict runtime rules are rigid and incredibly hard to translate. The Actor model, for example, handles errors by isolating them, letting them crash, and relying on automated supervision. Moving this to Java is a shallow mechanical exercise right up until a failure actually happens. At that point, Java's lack of a supervisor forces you to build a mountain of manual infrastructure just to recover your state. The host language just doesn't have the fundamental theory of errors needed to support the model. Similarly, the BEAM's absolute ban on shared mutable state makes the Java model completely alien in Elixir. You simply cannot emulate low-latency shared memory in a system that was specifically built to eliminate it to prevent cascading failures.  
On the other hand, Go's CSP model leans heavily on specific language-level syntax, like the select statement. Because this scheduling behavior is baked right into the compiler rather than just offered as a standard library, moving it to Java or Elixir results in really clunky simulations. You are forced to use polling loops or synchronous call/replies, which kills both the efficiency and the elegance of the original paradigm.  
Ultimately, these translation issues prove that a concurrency model is a holistic contract between the language, the runtime, and the programmer. When you rip a model out of its native ecosystem, you don't just lose convenient syntax. You lose the automatic correctness, fault tolerance, and safety guarantees that the host environment was explicitly built to enforce. The friction of translation directly reflects how deeply these models shape the fundamental theory of computation within their native languages.
