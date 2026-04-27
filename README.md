# Comp-590-Final

## Final Project: Concurrency Models Across Languages

### Part 1: The Model Matrix

#### 1. Actor Model (Erlang-style) in Java
**What maps naturally:** Java’s native threads and the `java.util.concurrent` package provide a baseline for concurrency. A thread can act as an isolated process, and a `LinkedBlockingQueue` can naturally serve as an unbounded, thread-safe mailbox for asynchronous message passing.

**What requires simulation:** Erlang’s OTP supervision model must be simulated entirely from scratch. Building a supervisor in Java requires explicit thread monitoring, restart logic, and state initialization.
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
While this simulates the message loop, the programmer is forced to manually build the fault-tolerance infrastructure that OTP provides automatically.

**What cannot be represented well:** The "let it crash" philosophy has no natural home in Java. Java’s architectural instinct is to catch exceptions locally and recover at the site of failure. In Erlang, the instinct is to allow the process to die so a supervisor can restart it with a clean state.

#### 2. CSP/Channels (Go-style) in Java
**What maps naturally:** Java’s `java.util.concurrent.BlockingQueue` serves as a highly effective approximation of a buffered Go channel. Similarly, a `SynchronousQueue` can approximate an unbuffered channel for rendezvous synchronization.

**What requires simulation:** Simulating Go's `select` statement is deeply problematic. `select` synchronizes on whichever of N channel operations is ready first. Because Java has no direct equivalent, simulating it requires polling (which wastes CPU and introduces latency) or building a shared multiplexing queue.
```java
// Simulating a buffered channel
BlockingQueue<Integer> ch = new ArrayBlockingQueue<>(10);
ch.put(1); // blocks if full
Integer val = ch.take(); // blocks if empty
```
**What cannot be represented well:** The semantics of Go's `select` regarding simultaneous readiness cannot be cleanly represented. When using a multiplexing queue to simulate `select`, you lose the strict directionality and type-safety of individual channels.

#### 3. Actor Model (Erlang-style) in Go
**What maps naturally:** Goroutines serve exceptionally well as lightweight, isolated processes. Go channels can be used to pass messages asynchronously between these entities.

**What requires simulation:** Erlang mailboxes are heterogeneous and support selective receive through pattern matching. To simulate a heterogeneous mailbox in Go, you must use a `chan interface{}`.
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
Simulating selective receive requires building a custom queue with a scan loop, which degrades performance to O(n) relative to the mailbox size.

**What cannot be represented well:** The elegance of pattern-matched dispatch is lost entirely. Furthermore, by utilizing `chan interface{}` to achieve message heterogeneity, the programmer completely sacrifices Go's compile-time type safety.

#### 4. Shared Memory (Java-style) in Go
**What maps naturally:** Because Go shares a single address space among goroutines, shared memory maps natively. Go directly provides `sync.Mutex` and `sync.RWMutex`, allowing for mutual exclusion exactly as Java does with `synchronized` blocks or `java.util.concurrent.locks`.
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
**What requires simulation / cannot be represented:** Architecturally, nothing requires simulation. The mechanisms are natively available. However, idiomatic Go strongly discourages this approach, favoring the philosophy of "Do not communicate by sharing memory; instead, share memory by communicating". While the code works natively, it violates community and language idioms.

#### 5. CSP/Channels (Go-style) in Elixir
**What maps naturally:** The BEAM provides incredibly lightweight processes (`spawn` or `GenServer`), which map perfectly to the concept of goroutines.

**What requires simulation:** Unbuffered Go channels provide "rendezvous synchronization," where the sender explicitly blocks until a receiver is ready. Elixir’s message passing is fundamentally asynchronous (`cast` or `!` operator). Simulating this synchrony requires using a call/reply mechanism.
```elixir
# Simulating a synchronous, blocking channel send via GenServer
GenServer.call(receiver_pid, {:send_channel, data})
```
The sender must actively wait for a `{:reply, ...}` tuple to simulate the blocking nature of an unbuffered Go channel.

**What cannot be represented well:** Go's `select` statement uses random selection among simultaneously ready channel cases. Elixir's `receive` block evaluates messages strictly sequentially from top to bottom based on pattern matching. True random selection of simultaneous events cannot be represented natively without writing complex, randomized mailbox-scanning logic.

#### 6. Shared Memory (Java-style) in Elixir
**What maps naturally:** Nothing maps naturally. The BEAM enforces strict isolation; absolutely no memory is shared between processes.

**What requires simulation:** You must simulate shared memory using an `Agent` (or an ETS table).
```elixir
# Simulating shared mutable state using an Agent
{:ok, shared_state} = Agent.start_link(fn -> 0 end)

# "Mutating" the state
Agent.update(shared_state, fn state -> state + 1 end)
```
An `Agent` works by wrapping state inside an isolated process and serializing all access requests through sequential message passing.

**What cannot be represented well:** True shared memory is a runtime constraint that is impossible on the BEAM. While the `Agent` simulation works, it eliminates the defining characteristic of the shared-memory model entirely: direct, low-latency access to a common address space. It is simply the actor model wearing a stateful mask.

---

### Part 2: Best and Worst Judgments

#### 2a. Best overall fit
The best overall representation is **Shared Memory (Java-style) in Go**. 

My criterion for the "best fit" is the absence of architectural workarounds and the retention of type safety and performance. By this criterion, Go is the clear winner because it features a single shared address space and natively provides the exact synchronization primitives (`sync.Mutex`) required to recreate Java's concurrency model. There is no need to write scan-loops, poll for messages, or sacrifice compile-time guarantees. 

A valid counterargument is that utilizing shared memory with locks is idiomatically discouraged in Go. However, idiomatic preference is a social constraint, not a technical one. The fundamental semantics of shared mutable state protected by locks are fully intact and highly performant in Go, making it the most accurate technical translation.

#### 2b. Worst overall fit
The worst overall representation is **Shared Memory (Java-style) in Elixir**. 

My criterion for the "worst fit" is the presence of a deep, unyielding runtime constraint that fundamentally changes the nature of the model. By this criterion, Elixir is the worst fit because the BEAM strictly isolates processes and does not permit shared mutable state. 

While a counterargument exists that you can simulate this using an `Agent`, this is a semantic illusion. An `Agent` serializes access through message passing, which means you are not actually accessing shared memory; you are querying an Actor. This simulation entirely destroys the core benefit of the Java model—direct, low-latency memory access—replacing it with inter-process communication overhead.

---

### Part 3: Synthesis

The difficulty of expressing each concurrency model in foreign languages reveals that these models are not merely differing APIs, but entirely different philosophies regarding where concurrent danger originates and how it must be managed. The portability of a model correlates heavily with whether it relies on lightweight language syntax or deep, prescriptive runtime architecture.

Models that rely on strict runtime isolation are incredibly difficult to port. For instance, the Actor model dictates a theory of error management based on "let it crash" and automated supervision. Translating this to Java is a shallow, mechanical exercise until failure occurs; at that point, Java's lack of runtime supervision forces the programmer to build massive amounts of manual infrastructure to recover state. Similarly, the BEAM's runtime prohibition of shared mutable state makes the Java model fundamentally foreign to Elixir, proving that you cannot emulate shared memory in a system specifically architected to eliminate it.

Conversely, Go's CSP model relies heavily on specific language-level constructs, namely the `select` statement for simultaneous event coordination. Because this is a language feature rather than just a library, moving it to Java or Elixir results in awkward simulations—like polling or synchronous call/replies—that sacrifice efficiency and elegance. 

Ultimately, these translation mismatches reveal that a concurrency model is a holistic contract. When you remove a model from its native language, you do not just lose convenience; you lose the automatic correctness, fault tolerance, and safety guarantees that the language's specific compiler and runtime were built to enforce.
