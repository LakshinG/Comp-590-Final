# Comp-590-Final

## Final Project: Concurrency Models Across Languages

### Part 1: The Model Matrix

#### 1. Actor Model (Erlang-style) in Java
**What maps naturally:** Java’s native threads and the java.util.concurrent package give us good tools to start with since we can easily create a dedicated thread to act as an isolated process, and use a LinkedBlockingQueue to handle asynchronous message passing. Because Java threads can block and wait on a queue indefinitely without chewing through CPU cycles, this setup naturally mimics how an Erlang process waits for new messages in its mailbox.

**What requires simulation:** The biggest issue or problem here is recreating Erlang’s OTP supervision model. In Java, there’s no built in framework to automatically monitor and restart threads when they fail. So the entire supervisor has to be built from scratch. This means writing explicit logic to track thread lifecycles, handle restarts, and ensure the state is properly wiped and initialized every single time.

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

**What cannot be represented well:** The whole "let it crash" philosophy fundamentally clashes with Java's architecture. Java is built around catching exceptions locally and fixing the problem exactly where it happens. The strict checked-exception system constantly forces you to handle errors rather than just abandoning the state. On top of that, you can never guarantee true memory isolation. Since everything runs in a shared heap, object references can easily leak between threads, which completely breaks the Actor model's core promise of not having shared mutable state.

#### 2. CSP/Channels (Go-style) in Java
**What maps naturally:** Java’s concurrency library makes basic channel communication pretty straightforward to pull off. A java.util.concurrent.BlockingQueue works great as an approximation of a buffered Go channel. The queue naturally handles the blocking behavior you need when a buffer fills up or empties out. If you need an unbuffered channel for rendezvous synchronization (where the sender waits until the receiver is ready), a SynchronousQueue perfectly maps to that requirement.  

**What requires simulation:** Simulating Go's select statement which lets a goroutine wait on whichever of multiple channels is ready first is a massive pain. Java just doesn't have a direct equivalent. You either have to use continuous polling (which wastes CPU and introduces lag) or build a shared multiplexing queue.  

```java
// Simulating a buffered channel
BlockingQueue<Integer> ch = new ArrayBlockingQueue<>(10);
ch.put(1); // blocks if full
Integer val = ch.take(); // blocks if empty
```

If you go the multiplexing route to simulate select, you completely lose the strict directional flow that makes Go channels safe. You also have to strip away type safety to allow different kinds of messages into the single queue, which defeats the purpose of typed channels.

**What cannot be represented well:** You simply cannot accurately represent Go's rules for simultaneous readiness in a select block. When multiple channels are ready at the same time, Go's runtime explicitly handles fair, pseudo random selection among them. Implementing this kind of fair choice algorithmic guarantee in user space Java requires incredibly complex, custom locking mechanisms that tank performance and ruin the simplicity of the CSP model.

#### 3. Actor Model (Erlang-style) in Go
**What maps naturally:** Goroutines are fantastic for this because they act perfectly as lightweight, isolated processes. They mirror the low memory footprint of Erlang processes, and you can spin up thousands of them without a problem. If you strictly use Go channels in a unidirectional way, they do a great job passing messages asynchronously between these isolated actors. Go's runtime scheduler is also highly optimized for handling tons of blocking goroutines at once

**What requires simulation:** Erlang mailboxes are heterogeneous and use pattern matching for selective receive. To simulate a mailbox that can accept any data type in Go, you are forced to use an empty interface channel (chan interface{}).

```go
// Simulating a heterogeneous mailbox
mailbox := make(chan interface{}, 100)

// Simulating selective receive requires draining the channel and re-queueing
func process(mailbox chan interface{}) {
    for msg := range mailbox {
        switch v := msg.(type) {
        case int:
            // process int
        default:
            // awkward simulation: put it back or hold in temporary slice
        }
    }
}
```

Selective receive requires building a custom scan loop. Because Go channels are strictly FIFO, skipping a non-matching message means you have to pop it off the channel and manually stash it somewhere else. This tanks your performance to $O(n)$ relative to the mailbox size.

**What cannot be represented well:** By using chan interface{} to accept different message types, you completely throw away Go's compile-time type safety. Furthermore, the elegance of Erlang's VM-level pattern-matched dispatch is lost entirely. Instead of clean syntax, you end up with verbose and error-prone type assertions cluttering up your business logic.

#### 4. Shared Memory (Java-style) in Go
**What maps naturally:** Because all goroutines share a single address space, the shared memory model maps completely natively. Go provides the sync package, which includes sync.Mutex and sync.RWMutex. This lets you lock down mutable state exactly the same way you would using synchronized blocks or explicit locks in Java.

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock() // Safely coordinates access to shared state
    c.v[key]++
}
```
The state management semantics are identical: you identify the shared variables and ensure every access is synchronized. Performance-wise, fine-grained mutexes in Go are highly optimized and give you the direct, low-latency memory access you expect from this model.

**What requires simulation / cannot be represented:** While the code runs perfectly, it heavily violates Go's design philosophy. Idiomatic Go pushes the mantra of communicating to share memory, rather than sharing memory to communicate. Just like in Java, the language doesn't magically prevent unsynchronized access natively, so you're still completely vulnerable to deadlocks and race conditions if you mess up your lock discipline. The technical fit is perfect, but the cultural and idiomatic fit is heavily discouraged.  

**What cannot be represented well:** While the technical implementation is native, this model runs contrary to Go's core design philosophy. Idiomatic Go strongly discourages this approach, favoring the mantra: "Do not communicate by sharing memory; instead, share memory by communicating." Furthermore, just like in Java, correctness depends entirely on programmer discipline. The language and compiler do relatively little to prevent unsynchronized access natively, meaning the standard risks of deadlocks, race conditions, and thread-starvation remain fully present, requiring reliance on external tooling like Go's race detector.  

#### 5. CSP/Channels (Go-style) in Elixir
**What maps naturally:** The BEAM virtual machine makes this pretty easy on the surface. Both the spawn or GenServer give you incredibly lightweight, isolated processes that map perfectly to the concept of goroutines. Both Go and Elixir are designed around the idea of spinning up massive numbers of concurrent entities and passing discrete data payloads between them. BEAMS' sceduler handles these tiny processes with the exact same kind of efficienty that the GO runtime scheduler handles goroutines, making the build and teh way these two models work feel very similar.

**What requires simulation:** The main issue is blocking. Unbuffered Go channels use "rendezvous synchronization"—the sender literally stops and blocks until the receiver is ready. Elixir’s message passing is completely asynchronous so it sends the signal then forgets it. To simulate that synchronous block, you have to build a strict call/reply mechanism.

```elixir
# Simulating a synchronous, blocking channel send via GenServer
GenServer.call(receiver_pid, {:send_channel, data})
```

The sender has to actively block and wait for a {:reply, ...}. This simulates the blocking nature of a Go channel, but it ties up the calling process and artificially throttles the throughput of the BEAM.

**What cannot be represented well:** Go's select block uses random selection when multiple channels are ready at once. Elixir's receive block doesn't work like that at all; it evaluates messages strictly sequentially from top to bottom based on pattern matching. You cannot cleanly represent true randomized selection of simultaneous events natively. Simulating it would mean pulling the whole mailbox into memory, shuffling it, and processing it out of order, which is incredibly slow and goes against OTP design principles.  

#### 6. Shared Memory (Java-style) in Elixir
**What maps naturally:** Absolutely nothing maps naturally here. The BEAM virtual machine strictly isolates processes, meaning no memory is ever shared between them. This isn't just a missing API; it's a hard runtime constraint, making the Java model totally foreign to Elixir.

**What requires simulation:** You must simulate shared memory using an `Agent` (or an ETS table).

```elixir
# Simulating shared mutable state using an Agent
{:ok, shared_state} = Agent.start_link(fn -> 0 end)

# "Mutating" the state
Agent.update(shared_state, fn state -> state + 1 end)
```

An Agent functions by wrapping the state inside a dedicated, isolated process and strictly serializing all access requests through sequential message passing. When benchmarking concurrent designs—such as comparing actor-style process rings in Elixir to threaded environments in Java—the simulation costs become bviously large. The latency of a simple state mutation transforms from a localized, nanosecond lock acquisition in Java to a full microsecond inter-process message round-trip on the BEAM. In high-contention scenarios, this serialization drastically limits system throughput.

**What cannot be represented well:** The defining characteristic of the shared-memory model—direct, low-latency access to a common address space—is completely eliminated. While the Agent provides the illusion of shared state to the programmer, it is structurally just the actor model applied to a stateful value. You cannot truly represent shared memory on a virtual machine explicitly engineered to prohibit it for the sake of fault tolerance.

---

### Part 2: Best and Worst Judgments

#### 2a. Best overall fit
The best overall representation is **Shared Memory (Java-style) in Go**. 

My criterion for the "best fit" is the absence of architectural workarounds and the retention of full performance and type safety. By this standard, Go easily wins because it uses a single shared address space and natively provides the exact synchronization primitives (sync.Mutex) needed to recreate Java's concurrency model. You don't have to write custom scan-loops, you don't have to poll for messages, and you don't have to throw away compile-time safety. The coordination tools map perfectly 1:1, allowing you to manage mutable state exactly as you would in Java.

A valid counterargument is that using shared memory with locks is strongly discouraged in the Go community. However, idiomatic preference is just a social constraint, not a technical limit. The actual semantics of shared mutable state are fully intact, fast, and supported directly by Go's runtime, making it the most technically accurate translation on the matrix.

#### 2b. Worst overall fit
The worst overall representation is **Shared Memory (Java-style) in Elixir**. 

My criterion for the "worst fit" is the presence of a deep, unyielding runtime constraint that fundamentally breaks the architectural nature of the target model. By this rule, Elixir is the worst fit because the BEAM virtual machine strictly isolates processes and explicitly bans shared mutable state.

You could argue that simulating shared data using an Agent gets the job done, but it's really just an illusion. An Agent serializes access by passing messages, which means you aren't accessing shared memory at all; you're just querying a bottlenecked Actor. This simulation entirely destroys the whole point of the Java model—direct, high-speed memory access—and replaces it with massive inter-process communication lag. It might work logically, but it completely misses the physical reality of the original model

---

### Part 3: Synthesis

The difficulty of forcing these concurrency models into foreign languages shows that they aren't just differing APIs. They represent fundamentally different philosophies about where bugs come from and how a system should handle danger. How portable a model is usually comes down to whether it relies on lightweight language syntax or deep, baked-in runtime architecture. When a model is tied strictly to the runtime, trying to port it exposes massive structural roadblocks.  

Models governed by strict runtime rules are rigid and incredibly hard to translate. The Actor model, for example, handles errors by isolating them, letting them crash, and relying on automated supervision. Moving this to Java is a shallow mechanical exercise right up until a failure actually happens. At that point, Java's lack of a supervisor forces you to build a mountain of manual infrastructure just to recover your state. The host language just doesn't have the fundamental theory of errors needed to support the model. Similarly, the BEAM's absolute ban on shared mutable state makes the Java model completely alien in Elixir. You simply cannot emulate low-latency shared memory in a system that was specifically built to eliminate it to prevent cascading failures.  

On the other hand, Go's CSP model leans heavily on specific language-level syntax, like the select statement. Because this scheduling behavior is baked right into the compiler rather than just offered as a standard library, moving it to Java or Elixir results in really clunky simulations. You are forced to use polling loops or synchronous call/replies, which kills both the efficiency and the elegance of the original paradigm.  

Ultimately, these translation issues prove that a concurrency model is a holistic contract between the language, the runtime, and the programmer. When you rip a model out of its native ecosystem, you don't just lose convenient syntax. You lose the automatic correctness, fault tolerance, and safety guarantees that the host environment was explicitly built to enforce. The friction of translation directly reflects how deeply these models shape the fundamental theory of computation within their native languages.
