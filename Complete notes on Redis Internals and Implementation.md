# Complete Notes on Redis Internals and Implementation

> A comprehensive guide to understanding Redis from the ground up - covering architecture, implementation details, data structures, algorithms, and advanced concepts.

---

## Table of Contents

1. [Introduction and Overview](#introduction-and-overview)
2. [Building Redis from Scratch](#building-redis-from-scratch)
3. [Core Commands Implementation](#core-commands-implementation)
4. [Memory Management & Eviction](#memory-management--eviction)
5. [Advanced Features](#advanced-features)
6. [Internal Data Structures](#internal-data-structures)
7. [Advanced Algorithms](#advanced-algorithms)

---

# Part 1: Introduction and Overview

## Chapter 1: Course Introduction and Redis Capabilities

### What is Redis?

**Redis** (Remote Dictionary Server) is an open-source, in-memory data structure store that serves multiple roles:
- **Database**: Persistent storage with configurable durability
- **Cache**: High-performance caching layer
- **Message Broker**: Pub/Sub messaging system
- **Streaming Engine**: Real-time data processing

### Why Learn Redis Internals?

Understanding Redis internals provides:

1. **Deep Database Knowledge**: Learn how production-grade databases handle concurrency, persistence, and memory management
2. **Engineering Skills**: Master advanced concepts like:
   - Event loops and I/O multiplexing
   - Single-threaded systems at scale
   - Memory-efficient data structures
   - Approximation algorithms
3. **Performance Optimization**: Understand trade-offs between memory, CPU, and throughput
4. **System Design Expertise**: Learn battle-tested patterns for high-performance systems

### Redis Core Capabilities

#### 1. Rich Data Structures

Redis natively supports advanced data structures:

- **Strings**: Binary-safe strings up to 512MB
- **Lists**: Ordered collections with fast head/tail operations
- **Sets**: Unique, unordered collections
- **Sorted Sets**: Sets with score-based ordering
- **Hashes**: Field-value pairs (like objects/dictionaries)
- **Bitmaps**: Bit-level operations on strings
- **HyperLogLog**: Probabilistic cardinality estimation
- **Geospatial Indexes**: Location-based queries
- **Streams**: Append-only log data structure

**Real-world applications:**
- Real-time chat systems
- Message buffers and queues
- Gaming leaderboards
- Session management
- Media streaming
- Real-time analytics

#### 2. Atomic Operations

**Key Characteristic**: Every Redis command is **atomic**.

When a command executes:
- It runs to completion without interruption
- No other command from any connection can interleave
- No need for explicit locks (mutexes, semaphores)

**Example Operations:**
```bash
# Atomic string operations
SET counter 10
INCR counter          # Returns 11, guaranteed atomic

# Atomic list operations
LPUSH queue "task1"   # Safe even with concurrent clients

# Atomic set operations
SADD tags "redis" "database"  # Union is atomic
```

This atomicity ensures **data correctness** even with thousands of concurrent clients, eliminating complex concurrency control in application code.

#### 3. In-Memory Storage

Redis stores **all data in RAM**, which is the primary reason for its exceptional performance:

**Advantages:**
- Sub-millisecond latency (typically microseconds)
- Millions of operations per second on commodity hardware
- No disk I/O bottlenecks during reads/writes

**Considerations:**
- RAM is more expensive than disk
- Memory capacity limits dataset size
- Requires persistence mechanisms for durability

#### 4. Configurable Persistence

Redis offers multiple persistence strategies to prevent data loss:

##### RDB (Redis Database Snapshots)

**How it works:**
- Periodically dumps the entire dataset to disk
- Creates point-in-time snapshots
- Uses `fork()` to create child process for background saving

**Advantages:**
- Compact single-file backups
- Fast restart times
- No performance impact during normal operations

**Disadvantages:**
- Potential data loss between snapshots (e.g., 5 minutes of data)
- Full snapshot becomes costly with large datasets

**Configuration Example:**
```bash
# Save snapshot every 5 minutes if at least 100 keys changed
save 300 100

# Trigger manual snapshot
BGSAVE
```

##### AOF (Append-Only File)

**How it works:**
- Logs every write operation
- Stores commands in RESP format
- Can be replayed to reconstruct state

**Advantages:**
- Minimal data loss (down to 1 second)
- Human-readable log format
- Automatic rewriting for optimization

**Disadvantages:**
- Larger file sizes than RDB
- Slower restart times (must replay log)

**Example AOF Content:**
```
*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
*2\r\n$4\r\nINCR\r\n$7\r\ncounter\r\n
```

**Background Rewriting:**
```bash
# Optimize AOF file by rewriting with minimal commands
BGREWRITEAOF
```

##### Hybrid Approach

Redis can use both RDB and AOF:
- RDB for fast restarts
- AOF for durability

#### 5. Replication

Redis supports **synchronous replication** to replica servers:

**Use Cases:**
- High availability (failover)
- Read scaling (distribute reads)
- Data durability (replicas persist independently)

#### 6. Transactions

Redis provides **transaction support** via MULTI/EXEC:

```bash
MULTI              # Begin transaction
SET key1 "value1"  # Queued
SET key2 "value2"  # Queued
INCR counter       # Queued
EXEC               # Execute all atomically
```

**Characteristics:**
- Commands execute atomically as a batch
- **No rollback** support (by design)
- Simple and predictable behavior
- Single-threaded execution prevents conflicts

**Alternative: DISCARD**
```bash
MULTI
SET key "value"
DISCARD            # Abort transaction, clear queue
```

#### 7. Pub/Sub Messaging

Redis implements a **publish-subscribe** pattern:

**Publisher:**
```bash
PUBLISH news:tech "Redis 7.0 released"
```

**Subscriber:**
```bash
SUBSCRIBE news:tech
# Receives: "Redis 7.0 released"
```

**Features:**
- Push-based message delivery
- Multiple subscribers per channel
- Pattern matching for subscriptions
- Fire-and-forget (no message persistence)

#### 8. Automatic Key Expiration (TTL)

Keys can have **Time-To-Live** (TTL) for automatic deletion:

```bash
# Set key with 60-second expiration
SET session:user123 "data" EX 60

# Check remaining time
TTL session:user123   # Returns seconds remaining

# After 60 seconds
GET session:user123   # Returns (nil)
```

**Use Cases:**
- Session management
- Temporary authentication tokens
- Rate limiting
- Cache invalidation

#### 9. Eviction Policies

When memory is full, Redis uses **eviction strategies** to make space:

**Available Policies:**
- `noeviction`: Reject new writes
- `allkeys-lru`: Evict least recently used keys (any key)
- `allkeys-lfu`: Evict least frequently used keys (any key)
- `allkeys-random`: Evict random keys
- `volatile-lru`: LRU on keys with TTL only
- `volatile-lfu`: LFU on keys with TTL only
- `volatile-random`: Random eviction of keys with TTL
- `volatile-ttl`: Evict keys closest to expiration

**Configuration:**
```bash
# Set memory limit
CONFIG SET maxmemory 1gb

# Set eviction policy
CONFIG SET maxmemory-policy allkeys-lru
```

---

### The DiceDB Project

This course guides you through re-implementing Redis in **Go (Golang)** to create **DiceDB**:

**Project Goals:**
1. Drop-in replacement for Redis core features
2. Understand production database engineering
3. Learn single-threaded, high-performance architecture

**Source Code:**
- Repository: `github.com/dicedb/dice`
- Open source
- Step-by-step implementation

**Features Implemented:**
- Custom event loop
- RESP protocol
- Core commands (PING, GET, SET, INCR, DEL, etc.)
- AOF persistence
- Command pipelining
- Eviction strategies (LRU, LFU, random)
- Transactions
- Auto-expiration
- Graceful shutdown
- Object encodings

**Learning Outcomes:**
- Master event loops and I/O multiplexing
- Implement serialization protocols
- Build persistence mechanisms
- Design memory-efficient data structures
- Approximate algorithms (LRU, LFU, HyperLogLog)
- Signal handling and graceful shutdown

---

## Course Structure

### Chapter Overview

**8 Chapters • 9 Hours of Content**

#### Chapter 1: Foundation (Files 01-02)
- Course introduction
- What makes Redis special
- Architecture overview

#### Chapter 2: Network & Protocol (Files 03-08)
- TCP server fundamentals
- RESP protocol
- Event loops and I/O multiplexing
- Concurrent client handling

#### Chapter 3: Core Operations (Files 09-10)
- GET, SET, TTL implementation
- DEL, EXPIRE, auto-deletion

#### Chapter 4: Optimization (Files 11-13)
- Eviction strategies
- Command pipelining
- AOF persistence

#### Chapter 5: Object System (Files 14-15)
- Object encodings
- INCR implementation
- INFO command
- Statistics

#### Chapter 6: LRU Algorithm (Files 16-17)
- Approximated LRU theory
- LRU implementation

#### Chapter 7: Advanced Topics (Files 18-21)
- Memory capping
- Custom allocators (jemalloc/tcmalloc)
- Graceful shutdown
- Transactions

#### Chapter 8: Data Structures & Algorithms (Files 22-27)
- List internals (ziplist, quicklist)
- Set internals (intset)
- Geospatial queries (geohash)
- String internals (SDS)
- HyperLogLog
- LFU and approximate counting

---

### Prerequisites

**Required:**
- Basic to intermediate Go programming
- Linux-based development environment
- Curiosity and willingness to learn

**Environment Setup:**
- **Linux**: Native support
- **Windows**: Use WSL (Windows Subsystem for Linux)
- **macOS**: Use Docker container (for system calls)

**Why Linux?**
- Course uses Linux kernel system calls
- Event loop requires `epoll` (Linux) or equivalents
- Direct syscall interface demonstration

---

### Development Approach

**Code Walkthroughs (Not Live Coding):**
- Pre-written, curated code
- Detailed explanations of design decisions
- Focus on essential concepts over exhaustive coverage

**Focused Implementation:**
- Core commands, not every Redis command
- Fundamental data structures
- Production-ready patterns

**Support:**
- Discord community for async questions
- Bi-weekly Zoom sessions (30 minutes)
- Access to full source code

---

## Chapter 2: What Makes Redis Special - Architecture and Concurrency

### The Concurrency Challenge

The fundamental challenge for any server handling multiple clients is: **How do we efficiently manage thousands of concurrent connections on a single machine?**

Two primary approaches exist:
1. **Multi-threading** (traditional approach)
2. **I/O Multiplexing with Event Loops** (Redis's approach)

---

### Approach 1: Multi-Threading Model

#### How It Works

**Concept**: Spawn a new thread for each incoming client connection.

```go
// Pseudo-code for multi-threaded server
func main() {
    listener := net.Listen("tcp", ":6379")

    for {
        conn := listener.Accept()  // Block until new connection

        // Spawn new thread for this client
        go handleClient(conn)
    }
}

func handleClient(conn net.Conn) {
    for {
        command := readCommand(conn)
        result := executeCommand(command)
        writeResponse(conn, result)
    }
}
```

#### Advantages

- **True Concurrency**: Multiple threads execute simultaneously on multiple CPU cores
- **Blocking Operations**: When one thread blocks on I/O, others continue executing
- **Intuitive Model**: Straightforward programming model

#### Critical Problems

##### 1. Race Conditions and Data Integrity

**The `counter++` Problem:**

Consider a simple counter increment operation:

```go
// Shared global variable
var counter int = 10

// Two threads execute this simultaneously
counter++  // NOT ATOMIC!
```

**What actually happens:**
```
Thread 1                Thread 2
--------------------------------------
Read counter (10)       Read counter (10)
Add 1 (11)              Add 1 (11)
Write counter = 11      Write counter = 11
```

**Expected Result**: 12
**Actual Result**: 11 (data corruption!)

**The problem**: `counter++` involves THREE operations:
1. **READ** from memory
2. **INCREMENT** in CPU register
3. **WRITE** back to memory

These operations are not atomic, creating a **race condition**.

##### 2. Required Solution: Synchronization Primitives

To prevent race conditions, we need **locks**:

```go
var counter int = 10
var mutex sync.Mutex

// Thread-safe increment
func incrementCounter() {
    mutex.Lock()         // Acquire lock
    counter++            // Critical section
    mutex.Unlock()       // Release lock
}
```

**Synchronization Primitives:**
- **Mutexes**: Mutual exclusion locks
- **Semaphores**: Counting locks
- **Read-Write Locks**: Multiple readers, exclusive writers
- **Condition Variables**: Thread synchronization

**Pessimistic Locking**: Only ONE thread can enter the critical section at a time.

##### 3. Performance Penalties

**Problems with locks:**

1. **Unnecessary Blocking**
   - Thread A holds lock
   - Thread B ready to execute but must wait
   - CPU cycles wasted

2. **Context Switching Overhead**
   - OS constantly switches between threads
   - Save/restore thread state
   - Flush CPU caches
   - Expensive operation

3. **Deadlocks**
   ```go
   // Thread 1
   mutex1.Lock()
   mutex2.Lock()  // Wait forever

   // Thread 2
   mutex2.Lock()
   mutex1.Lock()  // Wait forever
   ```

4. **Code Complexity**
   - Which data needs protection?
   - Which locks to acquire in what order?
   - Debugging race conditions is extremely difficult
   - Inconsistent in-memory state is hard to reproduce

##### 4. Memory Overhead

Each thread consumes:
- Stack space (typically 1-2 MB per thread)
- Thread control block
- Context switch overhead

**10,000 connections = 10,000 threads = ~10-20 GB RAM just for threads!**

---

### Approach 2: I/O Multiplexing with Event Loops (Redis's Approach)

#### The Core Insight

**Key Observation**: Most server time is spent **waiting for I/O**, not computing.

```
Traditional blocking:
-------------------------------------------------
Accept Connection → [BLOCK] → Read Data → [BLOCK] → Process → [BLOCK] → Write Response
                     (waiting)            (waiting)            (waiting)
```

**Redis's Insight**: Instead of blocking, let the OS tell us when I/O is ready.

---

#### What is an Event Loop?

**Definition**: A single-threaded pattern that monitors multiple I/O sources and dispatches events when they're ready.

**Common Misconceptions Addressed:**

❌ Event loop is NOT a separate process
❌ Event loop is NOT a separate thread
❌ Event loop is NOT where CPU instructions run

✅ Event loop IS a thin layer for I/O monitoring
✅ Event loop IS what makes single-threaded servers handle thousands of connections
✅ Event loop IS how Node.js, Redis, and Nginx achieve high performance

---

#### System Call: epoll (Linux I/O Multiplexing)

##### Unix/Linux File Descriptor Concept

**Everything is a File** in Unix:
- Regular files
- Network sockets
- Devices
- Pipes
- Memory

**File Descriptor (FD)**: An integer identifying an open file/socket (e.g., `3`, `42`, `1024`)

##### epoll System Calls

Redis uses `epoll` on Linux (equivalents: `kqueue` on BSD/macOS, `IOCP` on Windows).

**Three Core Functions:**

1. **`epoll_create1()`**
   ```c
   int epfd = epoll_create1(0);
   ```
   - Creates an epoll instance
   - Returns epoll file descriptor

2. **`epoll_ctl()`**
   ```c
   epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &event);
   ```
   - Registers file descriptors to monitor
   - Tells epoll what events to watch for (read, write)

3. **`epoll_wait()`**
   ```c
   int num_events = epoll_wait(epfd, events, MAX_EVENTS, timeout);
   ```
   - **BLOCKS** until at least one FD is ready
   - Returns list of ready file descriptors
   - This is the "wait" in event loop

---

#### How I/O Actually Works (Kernel Level)

**Network Data Reception Flow:**

```
1. Network Card receives packet
   ↓
2. Hardware triggers CPU interrupt
   ↓
3. Kernel handles interrupt, copies data to kernel buffer
   ↓
4. Kernel knows which process/socket owns this data
   ↓
5. When process scheduled on CPU, kernel copies data to user space
   ↓
6. Application (Redis) can now read data from socket
```

**epoll leverages kernel's knowledge:**
- Kernel maintains buffers for each socket
- Kernel knows when data is available
- `epoll_wait()` queries kernel: "Which sockets have data?"
- Kernel responds with ready file descriptors

---

#### Redis Event Loop Flow

**Initialization:**
```c
// Create epoll instance
int epfd = epoll_create1(0);

// Register server listening socket
epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &event);
```

**Main Loop:**
```c
while (true) {
    // Wait for events (BLOCKS here)
    int num_events = epoll_wait(epfd, events, MAX_EVENTS, -1);

    // Process each ready file descriptor
    for (int i = 0; i < num_events; i++) {
        int fd = events[i].data.fd;

        if (fd == server_fd) {
            // New client connection
            int client_fd = accept(server_fd, ...);

            // Register new client with epoll
            epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &event);
        }
        else {
            // Existing client has data
            read(fd, buffer, size);         // Won't block!
            result = executeCommand(buffer);
            write(fd, result, len);         // Send response
        }
    }
}
```

**Critical Point**: When `epoll_wait()` returns a file descriptor, the `read()` call is **guaranteed NOT to block** because the kernel confirmed data is available.

---

#### Visual Comparison

**Multi-Threading Model:**
```
Client 1 → Thread 1 → [Waiting for I/O]
Client 2 → Thread 2 → [Executing Command]
Client 3 → Thread 3 → [Waiting for I/O]
Client 4 → Thread 4 → [Blocked on Lock]
...
Client 1000 → Thread 1000 → [Waiting for I/O]

CPU: Context switching between 1000 threads
Memory: 1000 thread stacks
```

**Event Loop Model (Redis):**
```
Event Loop (Single Thread):
↓
epoll_wait() → [Client 3 ready, Client 7 ready, Client 42 ready]
↓
Process Client 3 command (fast, in-memory)
↓
Process Client 7 command (fast, in-memory)
↓
Process Client 42 command (fast, in-memory)
↓
Back to epoll_wait()

CPU: One thread, no context switches
Memory: Single stack
```

---

### Why Redis's Model is Superior

#### 1. No Synchronization Overhead

**Single-threaded execution = No race conditions**
- No mutexes needed
- No semaphores needed
- No deadlocks possible
- No pessimistic locking

**Data integrity is automatic:**
```bash
# These are atomic without any locks
INCR counter
LPUSH queue "task"
SADD users "alice"
```

#### 2. No Context Switching

- One thread runs continuously
- No kernel thread scheduler overhead
- No CPU cache thrashing
- Predictable performance

#### 3. Efficient CPU Utilization

**The Secret**: Network I/O is **slow**, memory operations are **fast**.

**Timing breakdown:**
- Network latency: 0.1ms to 100ms
- Disk I/O: 1ms to 10ms
- Memory access: 100 nanoseconds
- Redis command execution: 1-10 microseconds

**While waiting for network I/O** (client sending next command), Redis:
1. Processes commands from other clients
2. Executes in-memory operations (extremely fast)
3. Returns to wait for more I/O

**Redis exploits network latency to do useful work.**

#### 4. Low Memory Footprint

**Memory usage:**
- Multi-threading: ~10-20 GB for 10,000 connections
- Event loop: ~10-50 MB for 10,000 connections

**Memory saved can store actual data!**

#### 5. Scalability

**Redis can handle:**
- 10,000+ concurrent connections
- 100,000+ requests per second
- All on a **single thread**

**Benchmark comparison:**
```bash
# Redis with event loop
$ redis-benchmark -n 100000 -c 200 -q
SET: 37,735 requests per second

# Traditional threaded server
# Would struggle with 200 concurrent threads
```

---

### When Multi-Threading Makes Sense

**Note**: Multi-threading isn't always wrong. It's appropriate when:

1. **CPU-bound workload** (not I/O-bound)
   - Heavy computation
   - Data processing
   - Encryption/compression

2. **Blocking operations are necessary**
   - Disk-heavy databases (MySQL, PostgreSQL)
   - File processing

3. **Parallel computation needed**
   - Map-reduce operations
   - Scientific computing

**Redis's workload**:
- Extremely fast in-memory operations
- I/O-bound (network wait dominates)
- Perfect fit for event loop

---

### Key Takeaways

1. **Atomicity Without Locks**: Single-threaded execution provides automatic data integrity
2. **Event Loop ≠ Concurrency**: Redis handles thousands of clients concurrently despite single thread
3. **I/O Multiplexing**: `epoll` (Linux), `kqueue` (macOS), `IOCP` (Windows) enable efficient monitoring
4. **Memory Operations Are Fast**: Redis commands execute in microseconds
5. **Network Latency Is Slow**: Event loop fills this time with other clients' work
6. **Simplicity**: No locks, no deadlocks, no race conditions to debug

**Redis's Architecture Decision**: Optimize for the common case (I/O wait) rather than rare cases (CPU-bound), resulting in exceptional performance for in-memory operations.

---

---

# Part 2: Building Redis from Scratch

## Chapter 3: Writing a Simple TCP Echo Server

Before implementing Redis's complex event loop, let's understand the basics by building a simple TCP echo server in Go. This foundation will help us appreciate the limitations of blocking I/O and motivate the need for asynchronous I/O.

### Project Setup

**Structure:**
```
redis-clone/
├── go.mod        # No external dependencies
├── main.go       # Entry point
└── server/
    └── tcp.go    # Server implementation
```

**Starting the server:**
```bash
go run main.go
```

---

### Command-Line Flags

**Configuration via flags:**

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    // Define command-line flags
    port := flag.Int("port", 7379, "Port to listen on")
    host := flag.String("host", "0.0.0.0", "Host address to bind")

    flag.Parse()

    fmt.Printf("Starting server on %s:%d\n", *host, *port)

    // Start server
    startServer(*host, *port)
}
```

**Usage:**
```bash
# Default: 0.0.0.0:7379
go run main.go

# Custom port
go run main.go -port 8080

# Specific host
go run main.go -host 127.0.0.1 -port 6379
```

**Why `0.0.0.0`?**
- Accepts connections from ANY network interface
- `127.0.0.1`: Localhost only
- `0.0.0.0`: All interfaces (localhost, external IPs)

---

### TCP Server Implementation (Synchronous/Blocking)

```go
package server

import (
    "fmt"
    "io"
    "net"
    "log"
)

func StartTCPServer(host string, port int) error {
    // Step 1: Create TCP listener
    address := fmt.Sprintf("%s:%d", host, port)
    listener, err := net.Listen("tcp", address)
    if err != nil {
        return fmt.Errorf("failed to bind to %s: %v", address, err)
    }
    defer listener.Close()

    log.Printf("Server listening on %s", address)

    // Track concurrent clients
    concurrentClients := 0

    // Step 2: Accept connections in infinite loop
    for {
        // BLOCKING CALL: Wait for new connection
        conn, err := listener.Accept()
        if err != nil {
            log.Printf("Error accepting connection: %v", err)
            continue
        }

        concurrentClients++
        log.Printf("New client connected from %s. Total clients: %d",
                   conn.RemoteAddr(), concurrentClients)

        // Step 3: Handle client communication
        for {
            // Read command from client
            command, err := readCommand(conn)
            if err != nil {
                if err == io.EOF {
                    log.Printf("Client %s disconnected", conn.RemoteAddr())
                } else {
                    log.Printf("Error reading from %s: %v", conn.RemoteAddr(), err)
                }

                conn.Close()
                concurrentClients--
                log.Printf("Total clients: %d", concurrentClients)
                break  // Exit inner loop
            }

            // Log received command
            log.Printf("Received command: %s", command)

            // Echo command back to client
            respond(conn, command)
        }
    }
}

// readCommand reads data from connection
func readCommand(conn net.Conn) (string, error) {
    buffer := make([]byte, 1024)

    // BLOCKING CALL: Wait for data
    n, err := conn.Read(buffer)
    if err != nil {
        return "", err
    }

    return string(buffer[:n]), nil
}

// respond sends response back to client
func respond(conn net.Conn, response string) error {
    _, err := conn.Write([]byte(response))
    return err
}
```

---

### Server Execution Flow

**Step-by-step execution:**

1. **`net.Listen()`**: Creates TCP listener on specified host:port
   - Binds to network interface
   - Begins listening for connections
   - Returns `listener` object

2. **Outer Loop**: `listener.Accept()`
   - **BLOCKS** until a client connects
   - Returns `net.Conn` object for that specific client
   - Increments `concurrentClients` counter

3. **Inner Loop**: Handle client communication
   - Calls `readCommand(conn)`
   - **BLOCKS** waiting for data from client
   - When data arrives, process it (echo back)
   - Loop continues until client disconnects

4. **Error Handling**:
   - `io.EOF`: Client disconnected gracefully
   - Other errors: Connection issues
   - Close connection, decrement counter, break inner loop

---

### Testing with netcat

**Connect to server:**

```bash
# Terminal 1: Start server
go run main.go

# Terminal 2: Connect with netcat
nc localhost 7379
```

**Interactive session:**
```
$ nc localhost 7379
hello                    # You type
hello                    # Server echoes back
world                    # You type
world                    # Server echoes back
hello world              # You type
hello world              # Server echoes back
^C                       # Ctrl+C to disconnect
```

**Server logs:**
```
Server listening on 0.0.0.0:7379
New client connected from 127.0.0.1:52341. Total clients: 1
Received command: hello
Received command: world
Received command: hello world
Client 127.0.0.1:52341 disconnected
Total clients: 0
```

---

### Critical Limitation: Single-Threaded Blocking

**The Problem Revealed:**

```bash
# Terminal 1: Start server
go run main.go

# Terminal 2: First client connects
nc localhost 7379
hello                    # Works fine

# Terminal 3: Second client tries to connect
nc localhost 7379        # HANGS! Cannot connect
```

**Why Second Client Can't Connect:**

The server is **blocked** in the inner loop:
1. `Accept()` accepted first client
2. Entered inner loop calling `Read()` on first client's connection
3. `Read()` blocks waiting for data from first client
4. Server **cannot** go back to outer loop to call `Accept()` for second client
5. Second client's connection request queued but never accepted

**Disconnect first client:**
```bash
# Terminal 2: Ctrl+C (disconnect first client)

# Terminal 3: NOW second client connects!
hello                    # Now it works
```

**Server logs show the sequence:**
```
New client connected from 127.0.0.1:52341. Total clients: 1
# [First client active, server blocked in inner loop]
Client 127.0.0.1:52341 disconnected. Total clients: 0
New client connected from 127.0.0.1:52342. Total clients: 1
# [Second client finally accepted]
```

---

### Interacting with Redis CLI

**Connecting Redis CLI to our echo server:**

```bash
# Terminal 1: Start echo server
go run main.go

# Terminal 2: Connect Redis CLI
redis-cli -p 7379
```

**What happens:**

```bash
127.0.0.1:7379> PING
# Server receives and echoes: *1\r\n$4\r\nPING\r\n
```

**Server logs show RESP protocol:**
```
New client connected from 127.0.0.1:52343. Total clients: 1
Received command: *1
$7
COMMAND

Received command: *1
$4
PING
```

**RESP Format Observed:**
```
*1          # Array with 1 element
$4          # Bulk string of length 4
PING        # The command
\r\n        # CRLF line ending
```

**More complex command:**
```bash
127.0.0.1:7379> SET k v
```

**Server receives:**
```
*3          # Array with 3 elements
$3          # Bulk string length 3
SET         # Command
$1          # Bulk string length 1
k           # Key
$1          # Bulk string length 1
v           # Value
\r\n        # CRLF endings
```

This demonstrates that Redis CLI communicates using the **RESP protocol** over raw TCP connections.

---

### Visualizing the Blocking Problem

**Synchronous server behavior:**

```
Timeline:
---------------------------------------------------------------------------
Server:    [Accept] → [Read Client 1] → [Blocked...] → [Client 1 done] → [Accept Client 2]
Client 1:           Connected ←-------- Active ------→ Disconnect
Client 2:                      [Waiting...waiting...waiting...] Connected!
```

**Key observations:**

1. ✅ Server handles ONE client at a time
2. ❌ Other clients cannot connect during active session
3. ❌ Scalability is ZERO (only 1 client supported)
4. ❌ Network utilization is poor (idle during waits)

---

### Summary of Blocking Server

**What We Built:**
- Simple TCP echo server in Go
- Accepts connections on configurable host:port
- Reads data from clients
- Echoes data back
- Handles disconnections gracefully

**What We Learned:**

1. **`net.Listen()`**: Creates TCP listener
2. **`listener.Accept()`**: Blocking call, waits for connections
3. **`conn.Read()`**: Blocking call, waits for data
4. **Nested Loops**: Outer for accepting, inner for handling
5. **File Descriptors**: Connections represented as `net.Conn` objects

**Critical Limitation Identified:**

❌ **Single client support**: Server blocks on first client, cannot accept new connections

**Solution Preview:**

To support multiple concurrent clients, we need:
1. **Either**: Spawn thread per client (multi-threading approach)
2. **Or**: Use I/O multiplexing with event loops (Redis's approach)

In the next chapter, we'll implement the **asynchronous event loop** using `epoll`, enabling thousands of concurrent connections on a single thread.

---

## Chapter 4: Speaking Redis's Language - The RESP Protocol

### What is RESP?

**RESP** (REdis Serialization Protocol) is a simple, efficient protocol for client-server communication in Redis.

**Key Characteristics:**
- **Request-Response Protocol**: Both requests and responses use RESP encoding
- **Binary-Safe**: Can transmit any sequence of bytes
- **Human-Readable**: Easy to read and debug
- **Simple to Parse**: Minimal overhead, fast processing
- **Performant**: Compact representation with prefixed lengths

---

### RESP Design Principles

**1. Type Prefix**: Every data type starts with a special character
**2. CRLF Termination**: Data ends with `\r\n` (Carriage Return Line Feed)
**3. Prefixed Length**: Bulk strings and arrays specify length upfront
**4. Low Overhead**: Minimal extra bytes

**General Format:**
```
<type_indicator><data><CRLF>
```

---

### RESP Data Types

#### 1. Simple Strings

**Format:** `+<string>\r\n`

**Use Case:** Simple responses with low overhead

**Examples:**
```
+OK\r\n
+PONG\r\n
+QUEUED\r\n
```

**Overhead:** 1 byte (`+`) + 2 bytes (`\r\n`) = 3 bytes

**Characteristics:**
- Cannot contain `\r` or `\n` characters
- Used for short, simple responses
- Fast to parse (no length calculation needed)

---

#### 2. Errors

**Format:** `-<error_message>\r\n`

**Use Case:** Error responses

**Examples:**
```
-ERR unknown command 'FOOBAR'\r\n
-WRONGTYPE Operation against a key holding the wrong kind of value\r\n
-KEY NOT FOUND\r\n
```

**Structure:**
```
-          # Error indicator
<message>  # Error description
\r\n       # Terminator
```

---

#### 3. Integers

**Format:** `:<integer>\r\n`

**Use Case:** Numeric responses (counts, boolean values, timestamps)

**Examples:**
```
:0\r\n          # Zero/false
:1\r\n          # One/true
:1729\r\n       # Any 64-bit integer
:-1\r\n         # Negative integer
:1000000\r\n    # Large integer
```

**Range:** 64-bit signed integers (-2^63 to 2^63-1)

**Use Cases:**
- INCR/DECR return values
- List/Set cardinality
- Boolean results (0 = false, 1 = true)
- TTL values

---

#### 4. Bulk Strings

**Format:** `$<length>\r\n<data>\r\n`

**Use Case:** Binary-safe strings (can contain any bytes, including `\0`, `\r\n`)

**Examples:**

**String "PONG":**
```
$4\r\n
PONG\r\n
```

**Breakdown:**
```
$           # Bulk string indicator
4           # Length in bytes
\r\n        # CRLF
PONG        # Actual data (4 bytes)
\r\n        # Terminating CRLF
```

**String "hello world":**
```
$11\r\n
hello world\r\n
```

**Empty string:**
```
$0\r\n
\r\n
```
- Length is 0
- Still has data section (empty)
- Followed by CRLF

**Null value:**
```
$-1\r\n
```
- Length of -1 indicates NULL
- No data section
- Represents absence of value

---

#### Why Prefixed Length Matters

**Problem with null-terminated strings (C strings):**
```c
char* str = "hello\0world";  // Only "hello" visible
```

**Bulk strings are binary-safe:**
```
$11\r\n
hello\0world\r\n
```
- Parser reads exactly 11 bytes after length
- `\0` is just another byte
- Can transmit images, binary data, etc.

**Performance benefit:**
- Know exactly how many bytes to read
- Allocate buffer of exact size
- Single read operation (no scanning for terminator)

---

#### 5. Arrays

**Format:** `*<count>\r\n<element1><element2>...<elementN>`

**Use Case:** Commands, multiple responses, nested structures

**Examples:**

**Array with 3 elements (all bulk strings):**
```
*3\r\n
$3\r\nSET\r\n
$1\r\nk\r\n
$1\r\nv\r\n
```

**Breakdown:**
```
*3          # Array with 3 elements
\r\n
$3\r\nSET\r\n   # Element 1: Bulk string "SET"
$1\r\nk\r\n     # Element 2: Bulk string "k"
$1\r\nv\r\n     # Element 3: Bulk string "v"
```

**Mixed-type array (string, integer, string):**
```
*3\r\n
$1\r\nA\r\n      # Bulk string "A"
:200\r\n         # Integer 200
$3\r\ncat\r\n    # Bulk string "cat"
```

**Nested arrays:**
```
*2\r\n
*3\r\n
:1\r\n:2\r\n:3\r\n    # Inner array [1, 2, 3]
*2\r\n
$5\r\nHello\r\n
$5\r\nWorld\r\n       # Inner array ["Hello", "World"]
```

**Empty array:**
```
*0\r\n
```

**Null array:**
```
*-1\r\n
```

---

### RESP in Action: Redis Commands

#### PING Command

**Client sends:**
```
*1\r\n
$4\r\n
PING\r\n
```

**Server responds:**
```
+PONG\r\n
```

---

#### SET Command

**Client sends: `SET key value`**
```
*3\r\n
$3\r\nSET\r\n
$3\r\nkey\r\n
$5\r\nvalue\r\n
```

**Server responds:**
```
+OK\r\n
```

---

#### GET Command

**Client sends: `GET key`**
```
*2\r\n
$3\r\nGET\r\n
$3\r\nkey\r\n
```

**Server responds (key exists):**
```
$5\r\n
value\r\n
```

**Server responds (key doesn't exist):**
```
$-1\r\n
```

---

#### INCR Command

**Client sends: `INCR counter`**
```
*2\r\n
$4\r\nINCR\r\n
$7\r\ncounter\r\n
```

**Server responds:**
```
:42\r\n
```

---

### Why RESP is Efficient

**1. Compact Representation**
- Minimal overhead (1-3 bytes per data type)
- No verbose XML/JSON wrapping
- Binary-safe without base64 encoding

**2. Fast Parsing**
- Type identified by first byte
- Prefixed lengths enable single-pass parsing
- No backtracking or lookahead needed

**3. Buffer Optimization**
- Know exact buffer size upfront
- Avoid repeated allocations
- Single read from network

**4. Human-Readable**
- Easy to debug with `nc` or `telnet`
- Clear structure for learning
- Simple to implement

**Example: Reading bulk string**
```go
// Read "$4\r\n"
length := parseLength()  // Get 4

// Allocate buffer of exact size
buffer := make([]byte, length)

// Read exactly 4 bytes
read(buffer)  // "PONG"

// Read trailing "\r\n"
read(2)
```

**Single network read, optimal buffer allocation!**

---

## Chapter 5: Implementing RESP Decoder in Go

### Decoder Architecture

**Goal:** Parse RESP-encoded byte stream into Go data structures

**Main Function:**
```go
func Decode(data []byte) (interface{}, error)
```

**Input:** Raw bytes from network
**Output:** Decoded Go value (string, int, array, etc.)

---

### Helper Function: `decode1`

**Purpose:** Decode the *first* RESP value from byte slice

**Signature:**
```go
func decode1(data []byte) (value interface{}, delta int, error error)
```

**Returns:**
- `value`: Decoded Go value
- `delta`: Number of bytes consumed
- `error`: Parsing error (if any)

**Why return delta?**
- Multiple RESP values may be concatenated
- Need to know where next value starts
- Enables parsing pipelined commands

---

### Type Detection

**First byte determines type:**

```go
func decode1(data []byte) (interface{}, int, error) {
    if len(data) == 0 {
        return nil, 0, errors.New("empty data")
    }

    switch data[0] {
    case '+':
        return decodeSimpleString(data)
    case '-':
        return decodeError(data)
    case ':':
        return decodeInteger(data)
    case '$':
        return decodeBulkString(data)
    case '*':
        return decodeArray(data)
    default:
        return nil, 0, fmt.Errorf("unknown RESP type: %c", data[0])
    }
}
```

---

### Decoding Simple Strings

**Format:** `+<string>\r\n`

**Implementation:**
```go
func decodeSimpleString(data []byte) (string, int, error) {
    // Start from position 1 (skip '+')
    pos := 1

    // Find '\r\n'
    for pos < len(data) {
        if data[pos] == '\r' && pos+1 < len(data) && data[pos+1] == '\n' {
            // Extract string between '+' and '\r\n'
            str := string(data[1:pos])
            return str, pos + 2, nil  // +2 for \r\n
        }
        pos++
    }

    return "", 0, errors.New("incomplete simple string")
}
```

**Example:**
```
Input:  []byte("+PONG\r\n")
Output: "PONG", 7, nil
```

---

### Decoding Errors

**Format:** `-<error_message>\r\n`

**Implementation:**
```go
func decodeError(data []byte) (string, int, error) {
    // Reuse simple string logic (structure is identical)
    str, delta, err := decodeSimpleString(data)
    if err != nil {
        return "", 0, err
    }

    return str, delta, nil
}
```

**Example:**
```
Input:  []byte("-ERR unknown command\r\n")
Output: "ERR unknown command", 22, nil
```

---

### Decoding Integers

**Format:** `:<integer>\r\n`

**Implementation:**
```go
func decodeInteger(data []byte) (int64, int, error) {
    pos := 1  // Skip ':'
    var value int64 = 0
    negative := false

    // Handle negative sign
    if data[pos] == '-' {
        negative = true
        pos++
    }

    // Parse digits
    for pos < len(data) && data[pos] >= '0' && data[pos] <= '9' {
        digit := int64(data[pos] - '0')
        value = value*10 + digit
        pos++
    }

    // Apply sign
    if negative {
        value = -value
    }

    // Verify '\r\n'
    if pos+1 >= len(data) || data[pos] != '\r' || data[pos+1] != '\n' {
        return 0, 0, errors.New("invalid integer format")
    }

    return value, pos + 2, nil
}
```

**Example:**
```
Input:  []byte(":1729\r\n")
Output: 1729, 7, nil

Input:  []byte(":-42\r\n")
Output: -42, 6, nil
```

---

### Helper: Reading Length

**Used by bulk strings and arrays:**

```go
func readLength(data []byte, pos int) (int, int, error) {
    start := pos
    length := 0

    // Parse digits
    for pos < len(data) && data[pos] >= '0' && data[pos] <= '9' {
        digit := int(data[pos] - '0')
        length = length*10 + digit
        pos++
    }

    // Verify '\r\n'
    if pos+1 >= len(data) || data[pos] != '\r' || data[pos+1] != '\n' {
        return 0, 0, errors.New("invalid length format")
    }

    delta := pos - start + 2  // +2 for \r\n
    return length, delta, nil
}
```

---

### Decoding Bulk Strings

**Format:** `$<length>\r\n<data>\r\n`

**Implementation:**
```go
func decodeBulkString(data []byte) (string, int, error) {
    // Read length
    length, delta, err := readLength(data, 1)  // Start at position 1 (skip '$')
    if err != nil {
        return "", 0, err
    }

    pos := 1 + delta

    // Handle NULL bulk string
    if length == -1 {
        return "", pos, nil  // Return empty string for NULL
    }

    // Extract data
    if pos+length+2 > len(data) {
        return "", 0, errors.New("incomplete bulk string")
    }

    str := string(data[pos : pos+length])
    pos += length

    // Verify trailing '\r\n'
    if data[pos] != '\r' || data[pos+1] != '\n' {
        return "", 0, errors.New("bulk string missing trailing CRLF")
    }

    return str, pos + 2, nil
}
```

**Example:**
```
Input:  []byte("$4\r\nPONG\r\n")
Output: "PONG", 10, nil

Input:  []byte("$-1\r\n")
Output: "", 5, nil  // NULL value
```

---

### Decoding Arrays

**Format:** `*<count>\r\n<element1><element2>...`

**Implementation:**
```go
func decodeArray(data []byte) ([]interface{}, int, error) {
    // Read element count
    count, delta, err := readLength(data, 1)  // Skip '*'
    if err != nil {
        return nil, 0, err
    }

    pos := 1 + delta

    // Handle NULL array
    if count == -1 {
        return nil, pos, nil
    }

    // Allocate result array
    result := make([]interface{}, count)

    // Decode each element recursively
    for i := 0; i < count; i++ {
        value, elementDelta, err := decode1(data[pos:])
        if err != nil {
            return nil, 0, fmt.Errorf("error decoding array element %d: %v", i, err)
        }

        result[i] = value
        pos += elementDelta
    }

    return result, pos, nil
}
```

**Example:**
```
Input:  []byte("*3\r\n$3\r\nSET\r\n$1\r\nk\r\n$1\r\nv\r\n")
Output: []interface{}{"SET", "k", "v"}, 26, nil
```

---

### Complete Decoder

```go
package resp

import (
    "errors"
    "fmt"
)

// Decode parses RESP data and returns first value
func Decode(data []byte) (interface{}, error) {
    value, _, err := decode1(data)
    return value, err
}

// decode1 decodes first RESP value, returns value, bytes consumed, error
func decode1(data []byte) (interface{}, int, error) {
    if len(data) == 0 {
        return nil, 0, errors.New("empty data")
    }

    switch data[0] {
    case '+':
        return decodeSimpleString(data)
    case '-':
        return decodeError(data)
    case ':':
        return decodeInteger(data)
    case '$':
        return decodeBulkString(data)
    case '*':
        return decodeArray(data)
    default:
        return nil, 0, fmt.Errorf("unknown RESP type: %c", data[0])
    }
}

// ... (decoding functions as shown above)
```

---

### Handling Nested Structures

**Recursive array decoding automatically handles nesting:**

```
*2\r\n           # Array with 2 elements
*3\r\n:1\r\n:2\r\n:3\r\n    # Element 1: Array [1, 2, 3]
$5\r\nHello\r\n             # Element 2: Bulk string "Hello"
```

**Parsing flow:**
1. `decodeArray` sees `*2` → allocate array of size 2
2. Recursively call `decode1` for element 1:
   - Sees `*3` → allocate nested array
   - Recursively decodes integers 1, 2, 3
3. Recursively call `decode1` for element 2:
   - Sees `$5` → decode bulk string "Hello"
4. Return: `[]interface{}{[]interface{}{1, 2, 3}, "Hello"}`

---

### Key Design Patterns

**1. Delta Return Pattern**
- Every decoding function returns bytes consumed
- Enables sequential parsing: `pos += delta`
- Clean handling of concatenated values

**2. Recursive Descent**
- Arrays recursively call `decode1`
- Naturally handles any nesting depth
- Simple, elegant code

**3. Type Switching**
- First byte determines parser
- O(1) type detection
- Extensible for new types

**4. Error Propagation**
- Errors bubble up from inner decoders
- Provides context (which element failed)
- Early return on error

---

### Testing the Decoder

```go
func main() {
    // Test simple string
    data1 := []byte("+PONG\r\n")
    result1, _ := Decode(data1)
    fmt.Println(result1)  // Output: PONG

    // Test integer
    data2 := []byte(":42\r\n")
    result2, _ := Decode(data2)
    fmt.Println(result2)  // Output: 42

    // Test bulk string
    data3 := []byte("$5\r\nHello\r\n")
    result3, _ := Decode(data3)
    fmt.Println(result3)  // Output: Hello

    // Test array
    data4 := []byte("*3\r\n$3\r\nSET\r\n$1\r\nk\r\n$1\r\nv\r\n")
    result4, _ := Decode(data4)
    fmt.Println(result4)  // Output: [SET k v]
}
```

---

### Summary

**What We Built:**
- Complete RESP decoder in Go
- Handles all 5 RESP data types
- Supports nested structures
- Binary-safe parsing

**Key Concepts:**
1. **Type Prefix Detection**: First byte determines type
2. **Prefixed Length**: Enables optimal buffer allocation
3. **Delta Pattern**: Track bytes consumed for sequential parsing
4. **Recursive Descent**: Natural handling of nested arrays
5. **Binary Safety**: Bulk strings can contain any bytes

**Next Steps:**
- Implement RESP encoder
- Integrate with TCP server
- Build command parser on top of RESP decoder

---

## Chapter 6: Implementing the PING Command

### Integrating RESP with TCP Server

Now that we have a RESP decoder, let's implement our first Redis command: **PING**.

**Architecture:**
```
TCP Server → Read bytes → RESP Decoder → Command Handler → RESP Encoder → Response
```

---

### RESP Encoder Implementation

**Purpose:** Convert Go values into RESP-encoded bytes

```go
package resp

import (
    "fmt"
    "strconv"
)

// EncodeSimpleString encodes a simple string
func EncodeSimpleString(s string) []byte {
    return []byte(fmt.Sprintf("+%s\r\n", s))
}

// EncodeError encodes an error message
func EncodeError(msg string) []byte {
    return []byte(fmt.Sprintf("-%s\r\n", msg))
}

// EncodeInteger encodes an integer
func EncodeInteger(n int64) []byte {
    return []byte(fmt.Sprintf(":%d\r\n", n))
}

// EncodeBulkString encodes a bulk string
func EncodeBulkString(s string) []byte {
    if s == "" {
        return []byte("$-1\r\n")  // NULL bulk string
    }
    return []byte(fmt.Sprintf("$%d\r\n%s\r\n", len(s), s))
}

// EncodeArray encodes an array of values
func EncodeArray(values []interface{}) []byte {
    if values == nil {
        return []byte("*-1\r\n")  // NULL array
    }

    result := fmt.Sprintf("*%d\r\n", len(values))
    for _, v := range values {
        result += string(Encode(v))
    }
    return []byte(result)
}

// Encode encodes any value based on type
func Encode(value interface{}) []byte {
    switch v := value.(type) {
    case string:
        return EncodeBulkString(v)
    case int:
        return EncodeInteger(int64(v))
    case int64:
        return EncodeInteger(v)
    case []interface{}:
        return EncodeArray(v)
    case error:
        return EncodeError(v.Error())
    case nil:
        return []byte("$-1\r\n")
    default:
        return EncodeError(fmt.Sprintf("unsupported type: %T", v))
    }
}
```

---

### PING Command Handler

**Redis PING Behavior:**
- **Without argument**: Returns `PONG`
- **With argument**: Returns the argument

**Examples:**
```bash
127.0.0.1:6379> PING
PONG

127.0.0.1:6379> PING "hello"
"hello"

127.0.0.1:6379> PING hello world
(error) ERR wrong number of arguments for 'ping' command
```

**Implementation:**

```go
func evalPING(args []string) []byte {
    if len(args) == 0 {
        // No arguments: return PONG
        return resp.EncodeSimpleString("PONG")
    }

    if len(args) == 1 {
        // One argument: echo it back
        return resp.EncodeBulkString(args[0])
    }

    // Too many arguments
    return resp.EncodeError("ERR wrong number of arguments for 'ping' command")
}
```

---

### Command Dispatcher

**Parse and route commands:**

```go
func evalAndRespond(cmd []interface{}, conn net.Conn) {
    // Validate command structure
    if len(cmd) == 0 {
        conn.Write(resp.EncodeError("ERR empty command"))
        return
    }

    // Extract command name
    commandName := strings.ToUpper(cmd[0].(string))

    // Extract arguments
    args := make([]string, len(cmd)-1)
    for i := 1; i < len(cmd); i++ {
        args[i-1] = cmd[i].(string)
    }

    // Route to handler
    var response []byte
    switch commandName {
    case "PING":
        response = evalPING(args)
    case "COMMAND":
        // Redis CLI sends this on connect
        response = resp.EncodeArray([]interface{}{})
    default:
        response = resp.EncodeError(fmt.Sprintf("ERR unknown command '%s'", commandName))
    }

    // Send response
    conn.Write(response)
}
```

---

### Updated TCP Server with RESP

```go
func StartTCPServer(host string, port int) error {
    address := fmt.Sprintf("%s:%d", host, port)
    listener, err := net.Listen("tcp", address)
    if err != nil {
        return err
    }
    defer listener.Close()

    log.Printf("Server listening on %s", address)

    for {
        conn, err := listener.Accept()
        if err != nil {
            log.Printf("Error accepting connection: %v", err)
            continue
        }

        log.Printf("Client connected: %s", conn.RemoteAddr())

        // Handle client communication
        for {
            // Read data from connection
            buffer := make([]byte, 4096)
            n, err := conn.Read(buffer)
            if err != nil {
                if err == io.EOF {
                    log.Printf("Client %s disconnected", conn.RemoteAddr())
                } else {
                    log.Printf("Error reading: %v", err)
                }
                conn.Close()
                break
            }

            // Decode RESP command
            command, err := resp.Decode(buffer[:n])
            if err != nil {
                log.Printf("RESP decode error: %v", err)
                conn.Write(resp.EncodeError(fmt.Sprintf("ERR protocol error: %v", err)))
                continue
            }

            // Evaluate and respond
            evalAndRespond(command.([]interface{}), conn)
        }
    }
}
```

---

### Testing PING Command

**Test 1: PING without arguments**

```bash
$ redis-cli -p 7379
127.0.0.1:7379> PING
PONG
```

**Server logs:**
```
Client connected: 127.0.0.1:54321
Received command: [PING]
Sent response: +PONG\r\n
```

---

**Test 2: PING with argument**

```bash
127.0.0.1:7379> PING "hello world"
"hello world"
```

**Server logs:**
```
Received command: [PING hello world]
Sent response: $11\r\nhello world\r\n
```

---

**Test 3: Unknown command**

```bash
127.0.0.1:7379> FOOBAR
(error) ERR unknown command 'FOOBAR'
```

---

### Summary

**What We Built:**
- RESP encoder (complement to decoder)
- PING command handler
- Command dispatcher/router
- Complete request-response cycle

**Key Achievements:**
1. ✅ Parse RESP commands from clients
2. ✅ Route commands to handlers
3. ✅ Generate RESP responses
4. ✅ Handle errors gracefully

**Remaining Limitation:**
- ❌ Still single-threaded blocking
- ❌ Cannot handle multiple concurrent clients

**Next:** Implement event loop for true concurrency!

---

## Chapter 7: I/O Multiplexing and Event Loops

### The Async Revolution

**Problem:** Our blocking server handles ONE client at a time.

**Solution:** **I/O Multiplexing** - Monitor multiple file descriptors (sockets) simultaneously and handle whichever is ready.

---

### Unix System Calls for I/O Multiplexing

**Historical Evolution:**

1. **`select()`** (1983, BSD Unix)
   - Limited to 1024 file descriptors
   - O(n) scan of all FDs
   - Cross-platform

2. **`poll()`** (1997, System V Unix)
   - No FD limit
   - Still O(n) scan
   - Better than select

3. **`epoll()`** (2002, Linux 2.5.44)
   - No practical FD limit
   - **O(1) for ready FDs**
   - Linux-specific
   - **Redis uses this!**

4. **`kqueue()`** (2000, FreeBSD)
   - BSD/macOS equivalent
   - Similar performance to epoll

5. **`IOCP`** (Windows)
   - I/O Completion Ports
   - Proactive model

---

### epoll API Deep Dive

#### 1. `epoll_create1()` - Create epoll Instance

**Signature:**
```c
int epoll_create1(int flags);
```

**Purpose:** Create an epoll instance (returns file descriptor)

**Go Example:**
```go
import "golang.org/x/sys/unix"

// Create epoll instance
epfd, err := unix.EpollCreate1(0)
if err != nil {
    log.Fatal("epoll_create1 failed:", err)
}
defer unix.Close(epfd)
```

**Returns:** Epoll file descriptor (e.g., `7`, `8`, `9`)

---

#### 2. `epoll_ctl()` - Control epoll

**Signature:**
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

**Operations:**
- `EPOLL_CTL_ADD`: Register new FD
- `EPOLL_CTL_MOD`: Modify existing FD
- `EPOLL_CTL_DEL`: Remove FD

**Event Types:**
- `EPOLLIN`: Readable (data available)
- `EPOLLOUT`: Writable (can send data)
- `EPOLLERR`: Error condition
- `EPOLLHUP`: Hangup (disconnect)
- `EPOLLET`: Edge-triggered mode

**Go Example:**
```go
// Register client socket for read events
event := unix.EpollEvent{
    Events: unix.EPOLLIN,  // Monitor for incoming data
    Fd:     int32(clientFd),
}

err := unix.EpollCtl(epfd, unix.EPOLL_CTL_ADD, clientFd, &event)
if err != nil {
    log.Fatal("epoll_ctl failed:", err)
}
```

---

#### 3. `epoll_wait()` - Wait for Events

**Signature:**
```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

**Purpose:** Block until file descriptors are ready

**Parameters:**
- `epfd`: Epoll file descriptor
- `events`: Buffer to receive ready events
- `maxevents`: Maximum events to return
- `timeout`: Milliseconds (-1 = infinite, 0 = non-blocking)

**Returns:** Number of ready file descriptors

**Go Example:**
```go
// Allocate buffer for events
events := make([]unix.EpollEvent, 128)

// Wait for events (blocks here)
n, err := unix.EpollWait(epfd, events, -1)  // -1 = wait forever
if err != nil {
    log.Fatal("epoll_wait failed:", err)
}

// Process ready file descriptors
for i := 0; i < n; i++ {
    fd := int(events[i].Fd)
    // Handle fd (read data, accept connection, etc.)
}
```

---

### Event Loop Architecture

**Conceptual Flow:**

```
┌───────────────────────────────────────┐
│  Initialize                           │
│  - Create TCP listener                │
│  - Create epoll instance              │
│  - Register listener with epoll       │
└───────────────────────────────────────┘
                  ↓
┌───────────────────────────────────────┐
│  Main Event Loop                      │
│                                       │
│  while true:                          │
│    events = epoll_wait()  ← BLOCKS   │
│                                       │
│    for each ready_fd in events:      │
│      if ready_fd == listener:        │
│        accept new client             │
│        register client with epoll    │
│      else:                            │
│        read data from client         │
│        process command               │
│        send response                 │
└───────────────────────────────────────┘
```

---

### Go Implementation: Async TCP Server

```go
package server

import (
    "fmt"
    "log"
    "net"
    "golang.org/x/sys/unix"
)

const MaxEvents = 128

func StartAsyncTCPServer(host string, port int) error {
    // Step 1: Create TCP listener
    address := fmt.Sprintf("%s:%d", host, port)
    listener, err := net.Listen("tcp", address)
    if err != nil {
        return fmt.Errorf("failed to bind: %v", err)
    }
    defer listener.Close()

    log.Printf("Async server listening on %s", address)

    // Get listener's file descriptor
    listenerFile, err := listener.(*net.TCPListener).File()
    if err != nil {
        return fmt.Errorf("failed to get listener FD: %v", err)
    }
    listenerFd := int(listenerFile.Fd())

    // Step 2: Create epoll instance
    epfd, err := unix.EpollCreate1(0)
    if err != nil {
        return fmt.Errorf("epoll_create1 failed: %v", err)
    }
    defer unix.Close(epfd)

    // Step 3: Register listener with epoll
    event := unix.EpollEvent{
        Events: unix.EPOLLIN,
        Fd:     int32(listenerFd),
    }
    if err := unix.EpollCtl(epfd, unix.EPOLL_CTL_ADD, listenerFd, &event); err != nil {
        return fmt.Errorf("failed to register listener: %v", err)
    }

    // Buffer for events
    events := make([]unix.EpollEvent, MaxEvents)

    // Step 4: Main event loop
    for {
        // Wait for events (BLOCKS HERE)
        n, err := unix.EpollWait(epfd, events, -1)
        if err != nil {
            log.Printf("epoll_wait error: %v", err)
            continue
        }

        log.Printf("epoll_wait returned %d events", n)

        // Process each ready file descriptor
        for i := 0; i < n; i++ {
            fd := int(events[i].Fd)

            if fd == listenerFd {
                // New client connection
                handleNewClient(epfd, listener)
            } else {
                // Existing client has data
                handleClientData(epfd, fd)
            }
        }
    }
}

func handleNewClient(epfd int, listener net.Listener) {
    // Accept new connection
    conn, err := listener.Accept()
    if err != nil {
        log.Printf("Accept error: %v", err)
        return
    }

    log.Printf("New client connected: %s", conn.RemoteAddr())

    // Get client's file descriptor
    connFile, err := conn.(*net.TCPConn).File()
    if err != nil {
        log.Printf("Failed to get client FD: %v", err)
        conn.Close()
        return
    }
    clientFd := int(connFile.Fd())

    // Register client with epoll
    event := unix.EpollEvent{
        Events: unix.EPOLLIN,
        Fd:     int32(clientFd),
    }
    if err := unix.EpollCtl(epfd, unix.EPOLL_CTL_ADD, clientFd, &event); err != nil {
        log.Printf("Failed to register client: %v", err)
        conn.Close()
        return
    }

    log.Printf("Client FD %d registered with epoll", clientFd)
}

func handleClientData(epfd int, fd int) {
    // Read data
    buffer := make([]byte, 4096)
    n, err := unix.Read(fd, buffer)
    if err != nil || n == 0 {
        // Client disconnected
        log.Printf("Client FD %d disconnected", fd)
        unix.EpollCtl(epfd, unix.EPOLL_CTL_DEL, fd, nil)
        unix.Close(fd)
        return
    }

    log.Printf("Received %d bytes from FD %d", n, fd)

    // Echo data back
    unix.Write(fd, buffer[:n])
}
```

---

### Event Loop Execution Flow

**Scenario: 3 clients connect**

```
Time 0: Server starts
        - Listener FD: 3
        - epoll FD: 4
        - epoll monitoring: [3]

Time 1: Client A connects
        - epoll_wait() returns → FD 3 ready
        - Accept → Client A FD: 5
        - Register FD 5 with epoll
        - epoll monitoring: [3, 5]

Time 2: Client B connects
        - epoll_wait() returns → FD 3 ready
        - Accept → Client B FD: 6
        - Register FD 6 with epoll
        - epoll monitoring: [3, 5, 6]

Time 3: Client A sends data
        - epoll_wait() returns → FD 5 ready
        - Read from FD 5
        - Process command
        - Write response to FD 5

Time 4: Client B sends data, Client A sends data
        - epoll_wait() returns → FD 5, FD 6 ready (2 events)
        - Process FD 5
        - Process FD 6

Time 5: Client C connects, Client B sends data
        - epoll_wait() returns → FD 3, FD 6 ready (2 events)
        - Accept new client (FD 7)
        - Process FD 6
        - epoll monitoring: [3, 5, 6, 7]
```

---

### Testing Concurrent Clients

**Terminal 1: Start server**
```bash
go run main.go
```

**Terminal 2: Client A**
```bash
nc localhost 7379
hello from A
hello from A      # Echo
```

**Terminal 3: Client B (simultaneous)**
```bash
nc localhost 7379
hello from B
hello from B      # Echo
```

**Terminal 4: Client C (simultaneous)**
```bash
nc localhost 7379
hello from C
hello from C      # Echo
```

**Server logs:**
```
Async server listening on 0.0.0.0:7379
epoll_wait returned 1 events
New client connected: 127.0.0.1:52341
Client FD 5 registered with epoll
epoll_wait returned 1 events
New client connected: 127.0.0.1:52342
Client FD 6 registered with epoll
epoll_wait returned 1 events
New client connected: 127.0.0.1:52343
Client FD 7 registered with epoll
epoll_wait returned 2 events    # Clients A and B send data simultaneously
Received 13 bytes from FD 5
Received 13 bytes from FD 6
epoll_wait returned 1 events
Received 13 bytes from FD 7
```

**All clients active concurrently!** ✅

---

### Key Insights

**1. Single-Threaded Concurrency**
- One thread, one event loop
- Handles thousands of clients
- No context switching

**2. Non-Blocking Reads**
- `epoll_wait()` only returns ready FDs
- `read()` guaranteed not to block
- No wasted CPU cycles

**3. Fair Scheduling**
- Each iteration processes ALL ready FDs
- No client starvation
- Round-robin handling

**4. Memory Efficiency**
- No thread-per-client overhead
- Minimal kernel state
- Scales to 10,000+ connections

---

## Chapter 8: Handling Multiple Concurrent Clients

### Integrating Event Loop with RESP

**Goal:** Combine async I/O with RESP protocol handling

**Challenges:**
1. Incomplete RESP messages (partial reads)
2. Multiple commands in one read (pipelining)
3. Per-client state management

---

### Client State Management

**Problem:** Need to track incomplete data per client

**Solution:** Maintain a buffer per file descriptor

```go
type ClientState struct {
    fd     int
    buffer []byte  // Accumulated data
}

var clients = make(map[int]*ClientState)
```

---

### Handling Partial Reads

**Scenario:**

```
Read 1: "*3\r\n$3\r\nSE"           # Incomplete command
Read 2: "T\r\n$1\r\nk\r\n$1\r\nv\r\n"  # Completion
```

**Strategy:**
1. Accumulate data in client buffer
2. Attempt RESP decode
3. If incomplete, wait for more data
4. If complete, process and clear buffer

```go
func handleClientData(epfd int, fd int) {
    // Get or create client state
    client, exists := clients[fd]
    if !exists {
        client = &ClientState{fd: fd, buffer: []byte{}}
        clients[fd] = client
    }

    // Read new data
    tempBuffer := make([]byte, 4096)
    n, err := unix.Read(fd, tempBuffer)
    if err != nil || n == 0 {
        // Client disconnected
        delete(clients, fd)
        unix.EpollCtl(epfd, unix.EPOLL_CTL_DEL, fd, nil)
        unix.Close(fd)
        return
    }

    // Append to client buffer
    client.buffer = append(client.buffer, tempBuffer[:n]...)

    // Try to decode RESP command
    command, err := resp.Decode(client.buffer)
    if err != nil {
        // Incomplete data, wait for more
        log.Printf("Incomplete RESP from FD %d, waiting for more data", fd)
        return
    }

    // Process command
    response := evalAndRespond(command.([]interface{}))

    // Send response
    unix.Write(fd, response)

    // Clear buffer (command processed)
    client.buffer = []byte{}
}
```

---

### Command Pipelining Support

**Redis supports pipelining:** Client sends multiple commands without waiting for responses.

**Example:**
```bash
echo -e "PING\r\nPING\r\nPING\r\n" | nc localhost 7379
+PONG\r\n
+PONG\r\n
+PONG\r\n
```

**Challenge:** Multiple commands in single `read()` call

**Solution:** Process commands in loop until buffer exhausted

```go
func handleClientData(epfd int, fd int) {
    client, exists := clients[fd]
    if !exists {
        client = &ClientState{fd: fd, buffer: []byte{}}
        clients[fd] = client
    }

    // Read new data
    tempBuffer := make([]byte, 4096)
    n, err := unix.Read(fd, tempBuffer)
    if err != nil || n == 0 {
        delete(clients, fd)
        unix.EpollCtl(epfd, unix.EPOLL_CTL_DEL, fd, nil)
        unix.Close(fd)
        return
    }

    client.buffer = append(client.buffer, tempBuffer[:n]...)

    // Process ALL complete commands in buffer
    for {
        command, delta, err := resp.Decode1(client.buffer)  // Returns bytes consumed
        if err != nil {
            // Incomplete, wait for more data
            break
        }

        // Process command
        response := evalAndRespond(command.([]interface{}))
        unix.Write(fd, response)

        // Remove processed command from buffer
        client.buffer = client.buffer[delta:]

        // If buffer empty, break
        if len(client.buffer) == 0 {
            break
        }
    }
}
```

---

### Complete Async Server with RESP

```go
package main

import (
    "log"
    "net"
    "strings"
    "golang.org/x/sys/unix"
    "your-project/resp"
)

type ClientState struct {
    fd     int
    buffer []byte
}

var clients = make(map[int]*ClientState)

func main() {
    // Create listener
    listener, _ := net.Listen("tcp", "0.0.0.0:7379")
    listenerFile, _ := listener.(*net.TCPListener).File()
    listenerFd := int(listenerFile.Fd())

    // Create epoll
    epfd, _ := unix.EpollCreate1(0)
    defer unix.Close(epfd)

    // Register listener
    event := unix.EpollEvent{Events: unix.EPOLLIN, Fd: int32(listenerFd)}
    unix.EpollCtl(epfd, unix.EPOLL_CTL_ADD, listenerFd, &event)

    // Event loop
    events := make([]unix.EpollEvent, 128)
    for {
        n, _ := unix.EpollWait(epfd, events, -1)

        for i := 0; i < n; i++ {
            fd := int(events[i].Fd)

            if fd == listenerFd {
                handleNewClient(epfd, listener)
            } else {
                handleClientData(epfd, fd)
            }
        }
    }
}

func handleNewClient(epfd int, listener net.Listener) {
    conn, _ := listener.Accept()
    connFile, _ := conn.(*net.TCPConn).File()
    clientFd := int(connFile.Fd())

    log.Printf("Client %d connected", clientFd)

    event := unix.EpollEvent{Events: unix.EPOLLIN, Fd: int32(clientFd)}
    unix.EpollCtl(epfd, unix.EPOLL_CTL_ADD, clientFd, &event)

    clients[clientFd] = &ClientState{fd: clientFd, buffer: []byte{}}
}

func handleClientData(epfd int, fd int) {
    client := clients[fd]

    tempBuffer := make([]byte, 4096)
    n, err := unix.Read(fd, tempBuffer)
    if err != nil || n == 0 {
        log.Printf("Client %d disconnected", fd)
        delete(clients, fd)
        unix.EpollCtl(epfd, unix.EPOLL_CTL_DEL, fd, nil)
        unix.Close(fd)
        return
    }

    client.buffer = append(client.buffer, tempBuffer[:n]...)

    // Process all complete commands
    for {
        command, delta, err := resp.Decode1(client.buffer)
        if err != nil {
            break  // Incomplete
        }

        response := evalCommand(command.([]interface{}))
        unix.Write(fd, response)

        client.buffer = client.buffer[delta:]
        if len(client.buffer) == 0 {
            break
        }
    }
}

func evalCommand(cmd []interface{}) []byte {
    if len(cmd) == 0 {
        return resp.EncodeError("ERR empty command")
    }

    commandName := strings.ToUpper(cmd[0].(string))
    args := cmd[1:]

    switch commandName {
    case "PING":
        if len(args) == 0 {
            return resp.EncodeSimpleString("PONG")
        }
        return resp.EncodeBulkString(args[0].(string))
    case "COMMAND":
        return resp.EncodeArray([]interface{}{})
    default:
        return resp.EncodeError("ERR unknown command '" + commandName + "'")
    }
}
```

---

### Performance Testing

**Benchmark with `redis-benchmark`:**

```bash
redis-benchmark -p 7379 -t PING -n 100000 -c 50 -q
```

**Results:**
```
PING_INLINE: 45,000 requests per second
PING_BULK: 44,500 requests per second
```

**With 50 concurrent connections!**

---

### Summary

**What We Built:**
- Complete async TCP server
- epoll-based event loop
- Multi-client support
- RESP protocol integration
- Command pipelining
- Per-client state management

**Key Achievements:**
1. ✅ Handles thousands of concurrent clients
2. ✅ Single-threaded, no locks
3. ✅ Non-blocking I/O
4. ✅ Efficient CPU utilization
5. ✅ Redis-compatible protocol

**Architecture Highlights:**
- **Event Loop**: `epoll_wait()` monitors all sockets
- **State Management**: Per-client buffers for partial reads
- **Pipelining**: Process multiple commands per read
- **Scalability**: O(1) per-event, O(ready_fds) per iteration

---

# Part 3: Core Commands Implementation

## Chapter 9: Implementing GET, SET, and TTL

With our event loop and RESP protocol working, we can now implement Redis's core data storage commands.

### In-Memory Data Store

**Goal:** Implement a key-value store with expiration support

**Data Structure:**

```go
// Global in-memory store
var store = make(map[string]*Object)

// Object represents a Redis value with metadata
type Object struct {
    Value      interface{}
    Type       ObjectType
    ExpiresAt  int64  // Unix milliseconds, 0 = no expiration
}

type ObjectType int

const (
    TypeString ObjectType = iota
    TypeList
    TypeSet
    TypeHash
    TypeZSet
)
```

---

### SET Command

**Syntax:** `SET key value [EX seconds] [PX milliseconds] [NX|XX]`

**Options:**
- `EX seconds`: Set expiration in seconds
- `PX milliseconds`: Set expiration in milliseconds
- `NX`: Only set if key doesn't exist
- `XX`: Only set if key already exists

**Examples:**
```bash
SET name "Alice"           # Simple set
SET counter 42 EX 60       # Expires in 60 seconds
SET flag "value" NX        # Set only if doesn't exist
```

**Implementation:**

```go
func evalSET(args []string) []byte {
    if len(args) < 2 {
        return resp.EncodeError("ERR wrong number of arguments for 'set' command")
    }

    key := args[0]
    value := args[1]

    var expiryMs int64 = 0
    var nx, xx bool = false, false

    // Parse options
    for i := 2; i < len(args); i++ {
        option := strings.ToUpper(args[i])

        switch option {
        case "EX":
            if i+1 >= len(args) {
                return resp.EncodeError("ERR syntax error")
            }
            seconds, err := strconv.ParseInt(args[i+1], 10, 64)
            if err != nil || seconds <= 0 {
                return resp.EncodeError("ERR invalid expire time in set")
            }
            expiryMs = time.Now().UnixMilli() + (seconds * 1000)
            i++ // Skip next arg

        case "PX":
            if i+1 >= len(args) {
                return resp.EncodeError("ERR syntax error")
            }
            ms, err := strconv.ParseInt(args[i+1], 10, 64)
            if err != nil || ms <= 0 {
                return resp.EncodeError("ERR invalid expire time in set")
            }
            expiryMs = time.Now().UnixMilli() + ms
            i++

        case "NX":
            nx = true

        case "XX":
            xx = true

        default:
            return resp.EncodeError("ERR syntax error")
        }
    }

    // Check NX condition
    if nx {
        if _, exists := store[key]; exists {
            return resp.EncodeBulkString("")  // nil reply
        }
    }

    // Check XX condition
    if xx {
        if _, exists := store[key]; !exists {
            return resp.EncodeBulkString("")  // nil reply
        }
    }

    // Store the value
    store[key] = &Object{
        Value:     value,
        Type:      TypeString,
        ExpiresAt: expiryMs,
    }

    return resp.EncodeSimpleString("OK")
}
```

---

### GET Command

**Syntax:** `GET key`

**Returns:**
- Bulk string: Value if key exists
- Null: If key doesn't exist or expired

**Implementation:**

```go
func evalGET(args []string) []byte {
    if len(args) != 1 {
        return resp.EncodeError("ERR wrong number of arguments for 'get' command")
    }

    key := args[0]

    // Check if key exists
    obj, exists := store[key]
    if !exists {
        return resp.EncodeBulkString("")  // nil reply
    }

    // Check if expired
    if obj.ExpiresAt > 0 && time.Now().UnixMilli() >= obj.ExpiresAt {
        // Key expired, delete it
        delete(store, key)
        return resp.EncodeBulkString("")  // nil reply
    }

    // Return value
    return resp.EncodeBulkString(obj.Value.(string))
}
```

---

### TTL Command

**Syntax:** `TTL key`

**Returns:**
- `-2`: Key doesn't exist
- `-1`: Key exists but has no expiration
- `N`: Seconds until expiration

**Implementation:**

```go
func evalTTL(args []string) []byte {
    if len(args) != 1 {
        return resp.EncodeError("ERR wrong number of arguments for 'ttl' command")
    }

    key := args[0]

    // Check if key exists
    obj, exists := store[key]
    if !exists {
        return resp.EncodeInteger(-2)  // Key doesn't exist
    }

    // No expiration set
    if obj.ExpiresAt == 0 {
        return resp.EncodeInteger(-1)
    }

    // Calculate remaining time
    now := time.Now().UnixMilli()
    if now >= obj.ExpiresAt {
        // Already expired
        delete(store, key)
        return resp.EncodeInteger(-2)
    }

    remainingMs := obj.ExpiresAt - now
    remainingSeconds := remainingMs / 1000

    return resp.EncodeInteger(remainingSeconds)
}
```

---

### PTTL Command

**Syntax:** `PTTL key`

**Returns milliseconds instead of seconds:**

```go
func evalPTTL(args []string) []byte {
    if len(args) != 1 {
        return resp.EncodeError("ERR wrong number of arguments for 'pttl' command")
    }

    key := args[0]

    obj, exists := store[key]
    if !exists {
        return resp.EncodeInteger(-2)
    }

    if obj.ExpiresAt == 0 {
        return resp.EncodeInteger(-1)
    }

    now := time.Now().UnixMilli()
    if now >= obj.ExpiresAt {
        delete(store, key)
        return resp.EncodeInteger(-2)
    }

    remainingMs := obj.ExpiresAt - now
    return resp.EncodeInteger(remainingMs)
}
```

---

### Testing Storage Commands

**Test 1: Basic SET/GET**
```bash
127.0.0.1:7379> SET name "Redis"
OK
127.0.0.1:7379> GET name
"Redis"
127.0.0.1:7379> GET nonexistent
(nil)
```

**Test 2: Expiration**
```bash
127.0.0.1:7379> SET temp "value" EX 5
OK
127.0.0.1:7379> TTL temp
(integer) 5
127.0.0.1:7379> GET temp
"value"
# Wait 5 seconds
127.0.0.1:7379> GET temp
(nil)
127.0.0.1:7379> TTL temp
(integer) -2
```

**Test 3: NX/XX Options**
```bash
127.0.0.1:7379> SET key1 "value1" NX
OK
127.0.0.1:7379> SET key1 "value2" NX
(nil)
127.0.0.1:7379> SET key1 "value2" XX
OK
127.0.0.1:7379> GET key1
"value2"
```

---

## Chapter 10: Implementing DEL, EXPIRE, and Auto-Deletion

### DEL Command

**Syntax:** `DEL key [key ...]`

**Returns:** Number of keys deleted

**Implementation:**

```go
func evalDEL(args []string) []byte {
    if len(args) < 1 {
        return resp.EncodeError("ERR wrong number of arguments for 'del' command")
    }

    deletedCount := 0

    for _, key := range args {
        if _, exists := store[key]; exists {
            delete(store, key)
            deletedCount++
        }
    }

    return resp.EncodeInteger(int64(deletedCount))
}
```

**Examples:**
```bash
127.0.0.1:7379> SET k1 "v1"
OK
127.0.0.1:7379> SET k2 "v2"
OK
127.0.0.1:7379> DEL k1 k2 k3
(integer) 2
```

---

### EXPIRE Command

**Syntax:** `EXPIRE key seconds`

**Returns:**
- `1`: Expiration was set
- `0`: Key doesn't exist

**Implementation:**

```go
func evalEXPIRE(args []string) []byte {
    if len(args) != 2 {
        return resp.EncodeError("ERR wrong number of arguments for 'expire' command")
    }

    key := args[0]
    seconds, err := strconv.ParseInt(args[1], 10, 64)
    if err != nil {
        return resp.EncodeError("ERR value is not an integer or out of range")
    }

    // Check if key exists
    obj, exists := store[key]
    if !exists {
        return resp.EncodeInteger(0)
    }

    // Set expiration
    obj.ExpiresAt = time.Now().UnixMilli() + (seconds * 1000)

    return resp.EncodeInteger(1)
}
```

---

### PEXPIRE Command

**Syntax:** `PEXPIRE key milliseconds`

```go
func evalPEXPIRE(args []string) []byte {
    if len(args) != 2 {
        return resp.EncodeError("ERR wrong number of arguments for 'pexpire' command")
    }

    key := args[0]
    ms, err := strconv.ParseInt(args[1], 10, 64)
    if err != nil {
        return resp.EncodeError("ERR value is not an integer or out of range")
    }

    obj, exists := store[key]
    if !exists {
        return resp.EncodeInteger(0)
    }

    obj.ExpiresAt = time.Now().UnixMilli() + ms
    return resp.EncodeInteger(1)
}
```

---

### PERSIST Command

**Syntax:** `PERSIST key`

**Removes expiration from key**

```go
func evalPERSIST(args []string) []byte {
    if len(args) != 1 {
        return resp.EncodeError("ERR wrong number of arguments for 'persist' command")
    }

    key := args[0]

    obj, exists := store[key]
    if !exists {
        return resp.EncodeInteger(0)
    }

    if obj.ExpiresAt == 0 {
        return resp.EncodeInteger(0)  // Already persistent
    }

    obj.ExpiresAt = 0
    return resp.EncodeInteger(1)
}
```

---

### Auto-Deletion: Passive vs Active Expiry

Redis uses **two strategies** for expiring keys:

#### 1. Passive Expiry (Lazy Deletion)

**When:** Key is accessed (GET, SET, etc.)

**Already implemented:** Our GET command checks expiration before returning

```go
// In GET command
if obj.ExpiresAt > 0 && time.Now().UnixMilli() >= obj.ExpiresAt {
    delete(store, key)
    return resp.EncodeBulkString("")
}
```

**Advantage:** No background overhead
**Disadvantage:** Keys not accessed remain in memory

---

#### 2. Active Expiry (Background Deletion)

**When:** Periodically in background

**Redis Strategy:**
1. Sample N random keys with expiration
2. Delete all expired keys found
3. If >25% were expired, repeat

**Implementation:**

```go
// Background goroutine for active expiry
func startActiveExpiry() {
    ticker := time.NewTicker(100 * time.Millisecond)

    go func() {
        for range ticker.C {
            deleteExpiredKeys()
        }
    }()
}

func deleteExpiredKeys() {
    const sampleSize = 20
    const expiryThreshold = 0.25

    now := time.Now().UnixMilli()

    for {
        expiredCount := 0
        sampledCount := 0

        // Sample random keys with expiration
        for key, obj := range store {
            if obj.ExpiresAt == 0 {
                continue  // Skip keys without expiration
            }

            sampledCount++

            if now >= obj.ExpiresAt {
                delete(store, key)
                expiredCount++
            }

            if sampledCount >= sampleSize {
                break
            }
        }

        // Stop if we didn't find enough expired keys
        if sampledCount == 0 || float64(expiredCount)/float64(sampledCount) < expiryThreshold {
            break
        }
    }
}
```

**Explanation:**
1. Every 100ms, sample up to 20 random keys
2. Delete expired ones
3. If ≥25% were expired, likely more exist → sample again
4. Continue until <25% expired

**Why 25% threshold?**
- Balance between CPU usage and memory cleanup
- Redis's tested heuristic

---

### Optimized Expiry Tracking

**Problem:** Scanning entire store is inefficient

**Solution:** Maintain separate index of keys with expiration

```go
var store = make(map[string]*Object)
var expiryIndex = make(map[string]int64)  // key -> expiresAt

func setExpiry(key string, expiresAt int64) {
    if obj, exists := store[key]; exists {
        obj.ExpiresAt = expiresAt
        if expiresAt > 0 {
            expiryIndex[key] = expiresAt
        } else {
            delete(expiryIndex, key)
        }
    }
}

func deleteExpiredKeys() {
    const sampleSize = 20
    const expiryThreshold = 0.25

    now := time.Now().UnixMilli()

    for {
        expiredCount := 0
        sampledCount := 0

        // Sample from expiry index only
        for key, expiresAt := range expiryIndex {
            sampledCount++

            if now >= expiresAt {
                delete(store, key)
                delete(expiryIndex, key)
                expiredCount++
            }

            if sampledCount >= sampleSize {
                break
            }
        }

        if sampledCount == 0 || float64(expiredCount)/float64(sampledCount) < expiryThreshold {
            break
        }
    }
}
```

---

### Testing Expiration

**Test Active Expiry:**
```bash
# Set 100 keys with 1-second expiration
127.0.0.1:7379> for i in {1..100}; do redis-cli SET key$i value EX 1; done

# Check key count
127.0.0.1:7379> DBSIZE
(integer) 100

# Wait 2 seconds
# Active expiry should delete them

# Check again
127.0.0.1:7379> DBSIZE
(integer) 0
```

---

## Chapter 11: Eviction Strategies and Implementing Simple-First

### Why Eviction?

**Problem:** Redis is an in-memory database. What happens when memory is full?

**Options:**
1. **Reject new writes** (noeviction)
2. **Evict existing keys** (various policies)

---

### Redis Eviction Policies

#### 1. noeviction (Default)

**Behavior:** Return errors for write commands when memory limit reached

```bash
SET key value
-OOM command not allowed when used memory > 'maxmemory'
```

#### 2. allkeys-random

**Behavior:** Evict random keys from entire keyspace

**Use Case:** All keys equally important, no pattern to access

#### 3. allkeys-lru

**Behavior:** Evict least recently used keys (any key)

**Use Case:** Some keys accessed more frequently than others

#### 4. allkeys-lfu

**Behavior:** Evict least frequently used keys

**Use Case:** Track access frequency, not recency

#### 5. volatile-random

**Behavior:** Evict random keys among those with TTL

**Use Case:** Only evict temporary data

#### 6. volatile-lru

**Behavior:** LRU among keys with TTL

#### 7. volatile-lfu

**Behavior:** LFU among keys with TTL

#### 8. volatile-ttl

**Behavior:** Evict keys with shortest TTL first

---

### Configuration

```go
type EvictionPolicy int

const (
    NoEviction EvictionPolicy = iota
    AllKeysRandom
    AllKeysLRU
    AllKeysLFU
    VolatileRandom
    VolatileLRU
    VolatileLFU
    VolatileTTL
)

var config = struct {
    MaxMemory      int64
    EvictionPolicy EvictionPolicy
}{
    MaxMemory:      100 * 1024 * 1024,  // 100 MB
    EvictionPolicy: NoEviction,
}
```

---

### Memory Tracking

**Problem:** How do we know when memory limit is reached?

**Solution:** Track allocated memory

```go
var usedMemory int64

func trackMemoryAlloc(size int64) {
    atomic.AddInt64(&usedMemory, size)
}

func trackMemoryFree(size int64) {
    atomic.AddInt64(&usedMemory, -size)
}

func getUsedMemory() int64 {
    return atomic.LoadInt64(&usedMemory)
}

func isMemoryFull() bool {
    if config.MaxMemory == 0 {
        return false  // No limit
    }
    return getUsedMemory() >= config.MaxMemory
}
```

**Tracking allocations:**
```go
func storeKey(key string, obj *Object) {
    // Calculate size
    size := int64(len(key)) + getObjectSize(obj)

    store[key] = obj
    trackMemoryAlloc(size)
}

func deleteKey(key string) {
    if obj, exists := store[key]; exists {
        size := int64(len(key)) + getObjectSize(obj)
        delete(store, key)
        trackMemoryFree(size)
    }
}
```

---

### Implementing allkeys-random (Simple-First Eviction)

**Simplest eviction policy:** Pick random keys and delete them

```go
func evictKeysRandom(count int) int {
    evicted := 0

    for key := range store {
        deleteKey(key)
        evicted++

        if evicted >= count {
            break
        }
    }

    return evicted
}
```

**Triggering eviction:**
```go
func ensureMemoryLimit() error {
    if !isMemoryFull() {
        return nil
    }

    if config.EvictionPolicy == NoEviction {
        return errors.New("OOM command not allowed when used memory > 'maxmemory'")
    }

    // Evict keys until memory is below limit
    for isMemoryFull() {
        evicted := evictKeysRandom(10)  // Evict 10 at a time

        if evicted == 0 {
            // No keys left to evict
            return errors.New("OOM unable to evict keys")
        }
    }

    return nil
}
```

**Integrate with SET command:**
```go
func evalSET(args []string) []byte {
    // ... parse arguments ...

    // Check memory before setting
    if err := ensureMemoryLimit(); err != nil {
        return resp.EncodeError("ERR " + err.Error())
    }

    // Proceed with SET
    storeKey(key, &Object{
        Value:     value,
        Type:      TypeString,
        ExpiresAt: expiryMs,
    })

    return resp.EncodeSimpleString("OK")
}
```

---

### Testing Eviction

**Configure memory limit:**
```bash
CONFIG SET maxmemory 1mb
CONFIG SET maxmemory-policy allkeys-random
```

**Fill memory:**
```bash
# Insert keys until memory full
for i in {1..10000}; do
    redis-cli SET key$i "$(head -c 1000 /dev/urandom | base64)"
done
```

**Observe eviction:**
```bash
INFO memory
# used_memory: 1048576
# evicted_keys: 245
```

---

### Summary

**What We Implemented:**
1. ✅ GET/SET/TTL commands
2. ✅ DEL/EXPIRE/PERSIST commands
3. ✅ Passive expiry (lazy deletion)
4. ✅ Active expiry (background deletion)
5. ✅ Memory tracking
6. ✅ Simple random eviction

**Key Concepts:**
- **Object Structure**: Store values with metadata
- **Expiration Index**: Track keys with TTL for efficient scanning
- **Two-Phase Expiry**: Passive + Active
- **Memory Limits**: Track usage, enforce limits
- **Eviction Policies**: Balance between performance and fairness

**Next:** Implement more sophisticated eviction (LRU, LFU)

---

