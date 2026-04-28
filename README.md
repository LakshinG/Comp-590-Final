# Comp-590-Final

## Final Project: Concurrency Models Across Languages

### Part 1: The Model Matrix

#### 1. Actor Model (Erlang-style) in Java
**What maps naturally:** Java’s native threads and the java.util.concurrent package provide a functional baseline for concurrency. A dedicated Java thread can act as an isolated process, while a LinkedBlockingQueue naturally serves as an unbounded, thread-safe mailbox for asynchronous message passing. Because Java allows threads to block indefinitely without consuming excessive CPU cycles, waiting on a queue cleanly mimics an Actor waiting for a message.

**What requires simulation:** Erlang’s OTP supervision model, which provides structured fault tolerance by monitoring and restarting failed processes, must be simulated entirely from scratch. Building a supervisor in Java requires explicit thread lifecycle monitoring, custom restart logic, and rigorous state initialization routines.

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
While this simulates the message loop, the programmer is forced to manually build the fault-tolerance infrastructure that OTP provides automatically. This heavily increases code complexity and introduces a massive correctness risk, as custom supervision trees are prone to race conditions during thread teardown and startup.

**What cannot be represented well:** The core "let it crash" philosophy has no natural home in Java's architecture. Java’s structural instinct is to catch exceptions locally and recover gracefully at the exact site of the failure. The language’s strict checked-exception system actively fights the idea of abandoning state. Furthermore, true isolation cannot be guaranteed; because Java relies on a shared heap, object references can easily leak between "isolated" actor threads, entirely breaking the Actor model's guarantee against shared mutable state.

#### 2. CSP/Channels (Go-style) in Java
**What maps naturally:** Java’s robust concurrency library makes point-to-point communication relatively straightforward to approximate. A java.util.concurrent.BlockingQueue serves as a highly effective approximation of a buffered Go channel. The queue naturally handles the blocking and progress semantics required when a buffer fills up. Similarly, a SynchronousQueue effectively simulates an unbuffered channel, providing the exact rendezvous synchronization needed to ensure the sender blocks until the receiver is ready.

**What requires simulation:** Simulating Go's select statement—which synchronizes on whichever of $N$ channel operations is ready first—is deeply problematic. Java has no direct equivalent. Simulating it requires either continuous polling, which wastes CPU cycles and introduces severe latency spikes, or building a shared multiplexing queue.

```java
// Simulating a buffered channel
BlockingQueue<Integer> ch = new ArrayBlockingQueue<>(10);
ch.put(1); // blocks if full
Integer val = ch.take(); // blocks if empty
```

If you attempt to simulate select by wrapping multiple channels into a single multiplexed event queue, you lose the defining traits of the CSP model. You must strip away type-safety to allow different message types into the multiplexer, and you lose the strict directional flow (send-only or receive-only) that Go channels natively enforce.

**What cannot be represented well:** The defined semantics for simultaneous readiness in Go's select block cannot be accurately represented in Java. Go's runtime explicitly manages fair, pseudo-random selection when multiple channels are ready simultaneously. Implementing this fair-choice algorithmic guarantee in user-space Java requires highly complex, custom locking mechanisms that destroy performance and diverge fundamentally from the simplicity of the CSP paradigm.

#### 3. Actor Model (Erlang-style) in Go
**What maps naturally:** Goroutines serve exceptionally well as lightweight, isolated processes, easily mirroring the low memory footprint of Erlang processes. Go channels, when used in a strictly unidirectional manner, can be leveraged to pass messages asynchronously between these concurrent entities. The system scheduler in Go is also highly efficient at handling thousands of blocking goroutines, matching the concurrency scale expected in the Actor model.

**What requires simulation:** Erlang mailboxes are inherently heterogeneous and support selective receive through pattern matching. To simulate a heterogeneous mailbox in Go, you are forced to use an empty interface channel `chan interface{}`.

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

Simulating selective receive requires building a custom queue with a manual scan loop. Because Go channels are strictly FIFO , skipping a non-matching message means you must dequeue it and manually stash it in a temporary slice or requeue it. This degrades performance to $O(n)$ relative to the mailbox size and introduces a heavy memory management burden.  

**What cannot be represented well:** By utilizing chan interface{} to achieve message heterogeneity, the programmer completely sacrifices Go's compile-time type safety, which is one of the language's primary design pillars. Furthermore, the elegance and safety of Erlang's VM-level pattern-matched dispatch are lost entirely, replaced by verbose and error-prone type assertions that clutter the business logic.

#### 4. Shared Memory (Java-style) in Go
**What maps naturally:** Because Go threads share a single address space, the shared memory model maps completely natively. Go directly provides the sync package, featuring sync.Mutex and sync.RWMutex, which allows for manual mutual exclusion exactly as Java does with its synchronized blocks or explicit locks.

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
The state management semantics are identical: the programmer is fully responsible for identifying shared mutable state and ensuring that all accesses are properly synchronized. From a performance standpoint, fine-grained mutexes in Go are highly optimized and provide direct, low-latency access to the shared heap without the overhead of message passing. 

**What requires simulation / cannot be represented:** Architecturally, nothing requires simulation. The necessary coordination mechanisms are natively available and supported by Go's underlying memory model, which provides clear happens-before guarantees for lock acquisition and release.

**What cannot be represented well:** While the technical implementation is native, this model runs contrary to Go's core design philosophy. Idiomatic Go strongly discourages this approach, favoring the mantra: "Do not communicate by sharing memory; instead, share memory by communicating." Furthermore, just like in Java, correctness depends entirely on programmer discipline. The language and compiler do relatively little to prevent unsynchronized access natively, meaning the standard risks of deadlocks, race conditions, and thread-starvation remain fully present, requiring reliance on external tooling like Go's race detector.  

#### 5. CSP/Channels (Go-style) in Elixir
**What maps naturally:** The BEAM virtual machine provides incredibly lightweight, isolated processes via spawn or OTP's GenServer. These map perfectly to the concept of goroutines, as both are designed to be spun up in the tens of thousands without exhausting system resources. Both systems also share a core reliance on passing discrete payloads between concurrent entities.

**What requires simulation:** Unbuffered Go channels provide "rendezvous synchronization," meaning the sender explicitly blocks and waits until a receiver is ready to accept the payload. Elixir’s native message passing, however, is fundamentally asynchronous (via cast or the ! operator). Simulating this synchronous rendezvous requires implementing a strict call/reply mechanism.

```elixir
# Simulating a synchronous, blocking channel send via GenServer
GenServer.call(receiver_pid, {:send_channel, data})
```

The sender must actively block and wait for a {:reply, ...} tuple. This simulates the blocking nature of an unbuffered Go channel, but at the cost of tying up the calling process and artificially throttling the throughput of the BEAM.

**What cannot be represented well:** Go's select statement relies on random selection among simultaneously ready channel operations to ensure fairness and prevent resource starvation. Elixir's receive construct, however, evaluates messages strictly sequentially from top to bottom based on pattern matching. True randomized selection of simultaneous events cannot be represented cleanly. Simulating it would require pulling the entire mailbox contents into memory, manually shuffling them, and processing them out of order, which fundamentally breaks OTP design principles and introduces massive performance penalties.

#### 6. Shared Memory (Java-style) in Elixir
**What maps naturally:** Absolutely nothing maps naturally. The BEAM virtual machine enforces strict architectural isolation; no memory is ever shared between processes. This is a hard runtime constraint, not merely a missing library, making the Java model fundamentally foreign.

**What requires simulation:** You must simulate shared memory using an `Agent` (or an ETS table).

```elixir
# Simulating shared mutable state using an Agent
{:ok, shared_state} = Agent.start_link(fn -> 0 end)

# "Mutating" the state
Agent.update(shared_state, fn state -> state + 1 end)
```

An Agent functions by wrapping the state inside a dedicated, isolated process and strictly serializing all access requests through sequential message passing. When benchmarking concurrent designs—such as comparing actor-style process rings in Elixir to threaded environments in Java—the simulation costs become glaringly obvious. The latency of a simple state mutation transforms from a localized, nanosecond lock acquisition in Java to a full microsecond inter-process message round-trip on the BEAM. In high-contention scenarios, this serialization drastically limits system throughput.

**What cannot be represented well:** The defining characteristic of the shared-memory model—direct, low-latency access to a common address space—is completely eliminated. While the Agent provides the illusion of shared state to the programmer, it is structurally just the actor model applied to a stateful value. You cannot truly represent shared memory on a virtual machine explicitly engineered to prohibit it for the sake of fault tolerance.

---

### Part 2: Best and Worst Judgments

#### 2a. Best overall fit
The best overall representation is **Shared Memory (Java-style) in Go**. 

My criterion for the "best fit" is the absence of architectural workarounds and the retention of full performance and type safety. By this criterion, Go is the clear winner because it features a single shared address space and natively provides the exact synchronization primitives (sync.Mutex, sync.RWMutex) required to recreate Java's concurrency model. There is no need to write scan-loops, poll for messages, or sacrifice compile-time guarantees to achieve the desired outcome. The coordination mechanisms map 1:1, allowing the developer to safely manage mutable state just as they would in Java.

A valid counterargument is that utilizing shared memory with locks is idiomatically discouraged in Go. However, idiomatic preference is a social constraint, not a technical one. The fundamental semantics of shared mutable state protected by locks are fully intact and highly performant in Go, making it the most accurate technical translation.

#### 2b. Worst overall fit
The worst overall representation is **Shared Memory (Java-style) in Elixir**. 

My criterion for the "worst fit" is the presence of a deep, unyielding runtime constraint that fundamentally changes the nature of the model. By this criterion, Elixir is the worst fit because the BEAM strictly isolates processes and does not permit shared mutable state. 

While a counterargument exists that you can easily simulate shared data using an Agent, this is ultimately a semantic illusion. An Agent serializes access through message passing, which means you are not actually accessing shared memory; you are querying a bottlenecked Actor. This simulation entirely destroys the core benefit of the Java model—direct, low-latency memory access—replacing it with massive inter-process communication overhead. The simulation works logically, but it misses the fundamental physical semantics of the original model.

---

### Part 3: Synthesis

The difficulty of expressing each concurrency model in foreign languages reveals that these models are not merely differing APIs, but entirely different philosophies regarding where concurrent danger originates and how it must be managed. The portability of a model correlates heavily with whether its features rely on lightweight language syntax or deep, prescriptive runtime architecture. When a model is tied to the runtime, attempting to port it inevitably exposes deep structural mismatches.  Models that rely on strict runtime rules are incredibly rigid and difficult to port. For instance, the Actor model dictates a theory of error management based entirely on isolation, "let it crash," and automated supervision. Translating this to Java is a shallow, mechanical exercise only until a failure occurs; at that exact moment, Java's lack of runtime supervision forces the programmer to build massive amounts of manual infrastructure to recover state. The host language lacks the fundamental theory of errors required to support the model. Similarly, the BEAM's absolute runtime prohibition of shared mutable state makes the Java model completely foreign to Elixir. It proves that you cannot emulate low-latency shared memory in a system that was specifically architected to eliminate shared memory to prevent cascading state failures.

Conversely, Go's CSP model relies heavily on highly specific, language-level constructs, namely the select statement, which acts as a fair scheduler for simultaneous event coordination. Because this behavior is baked into the language rather than just offered as a standard library, moving it to Java or Elixir results in incredibly awkward simulations. The programmer is forced to use polling routines or synchronous call/replies, sacrificing both efficiency and the elegance of the original paradigm.

Ultimately, these translation mismatches reveal that a concurrency model is a holistic, binding contract between the language, the runtime, and the programmer. When you remove a model from its native ecosystem, you do not just lose convenience or syntax; you lose the automatic correctness, fault tolerance, and safety guarantees that the host environment was explicitly built to enforce. The difficulty in translation is a direct reflection of how deeply these models shape the fundamental theory of computation within their respective languages. 
