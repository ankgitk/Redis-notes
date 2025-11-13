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

# Part 5: Advanced Features

## Chapter 12: Command Pipelining

### What is Pipelining?

**Command Pipelining** is a Redis optimization technique that allows clients to send multiple commands in a single network request, significantly reducing round-trip time (RTT) overhead and improving throughput.

**Key Characteristics:**
- Multiple commands sent together
- Multiple responses returned together
- Commands execute independently (NOT a transaction)
- No atomicity guarantees across pipelined commands

### Why Pipelining Matters

#### The RTT Problem

**Traditional Request-Response Model:**

```
Client                          Server
  |                               |
  |--- SET key1 value1 --------->|
  |                               | (compute)
  |<---------- +OK ---------------|
  |                               |
  |--- SET key2 value2 --------->|
  |                               | (compute)
  |<---------- +OK ---------------|
  |                               |
  |--- GET key1 --------------->|
  |                               | (compute)
  |<-------- "value1" ------------|

Total time = 3 × RTT + 3 × computation_time
```

**With Pipelining:**

```
Client                          Server
  |                               |
  |--- SET key1 value1 --------->|
  |--- SET key2 value2 --------->|
  |--- GET key1 --------------->|
  |                               | (compute all)
  |<---------- +OK ---------------|
  |<---------- +OK ---------------|
  |<-------- "value1" ------------|

Total time = 1 × RTT + 3 × computation_time
```

**Performance Impact:**
- **Localhost**: RTT ≈ 0.1ms (minimal benefit)
- **Same datacenter**: RTT ≈ 1-5ms (moderate benefit)
- **Cross-region**: RTT ≈ 50-200ms (massive benefit)

#### Throughput Improvement

Without pipelining, if each command takes 1ms RTT:
- **Throughput**: 1,000 commands/second

With pipelining (batch of 100 commands):
- **Throughput**: 100,000 commands/second (100× improvement)

---

### How Pipelining Works

#### RESP Protocol for Multiple Commands

Commands are concatenated as a single byte stream using RESP encoding:

**Individual Commands:**
```
PING      → *1\r\n$4\r\nPING\r\n
SET k v   → *3\r\n$3\r\nSET\r\n$1\r\nk\r\n$1\r\nv\r\n
GET k     → *2\r\n$3\r\nGET\r\n$1\r\nk\r\n
```

**Pipelined Request (concatenated):**
```
*1\r\n$4\r\nPING\r\n*3\r\n$3\r\nSET\r\n$1\r\nk\r\n$1\r\nv\r\n*2\r\n$3\r\nGET\r\n$1\r\nk\r\n
```

**Pipelined Response (concatenated):**
```
+PONG\r\n+OK\r\n$1\r\nv\r\n
```

---

### Implementation

#### Architecture Changes

**Before (Single Command):**
```
readCommand()  → returns 1 command
eval()         → processes 1 command
respond()      → writes 1 response directly to socket
```

**After (Pipelining):**
```
readCommands() → returns []commands (slice of commands)
eval()         → processes each command, buffers responses
respond()      → writes all buffered responses in one socket write
```

#### 1. Reading Multiple Commands

**Old: `readCommand` (single command):**
```go
func readCommand(conn net.Conn) (*redisCommand, error) {
    data := make([]byte, 4096)
    n, err := conn.Read(data)
    if err != nil {
        return nil, err
    }

    cmd := decode(data[:n])  // Returns single command
    return cmd, nil
}
```

**New: `readCommands` (multiple commands):**
```go
func readCommands(conn net.Conn) ([]*redisCommand, error) {
    data := make([]byte, 4096)
    n, err := conn.Read(data)
    if err != nil {
        return nil, err
    }

    cmds := decodeMultiple(data[:n])  // Returns slice of commands
    return cmds, nil
}
```

#### 2. Decoding Multiple Commands

**Modified `decode` function:**
```go
// decodeMultiple processes concatenated RESP commands
func decodeMultiple(data []byte) []*redisCommand {
    var commands []*redisCommand
    offset := 0

    for offset < len(data) {
        // Decode one command at a time
        cmd, bytesRead := decodeOne(data[offset:])
        if cmd == nil {
            break  // No more complete commands
        }

        commands = append(commands, cmd)
        offset += bytesRead  // Advance to next command
    }

    return commands
}

// decodeOne decodes a single RESP array and returns bytes consumed
func decodeOne(data []byte) (*redisCommand, int) {
    if len(data) == 0 || data[0] != '*' {
        return nil, 0
    }

    // Parse array length
    i := 1
    for data[i] != '\r' {
        i++
    }
    arrayLen, _ := strconv.Atoi(string(data[1:i]))
    i += 2  // Skip \r\n

    startOffset := 0
    cmd := &redisCommand{
        Args: make([]string, arrayLen),
    }

    // Parse each bulk string element
    for idx := 0; idx < arrayLen; idx++ {
        // Skip '$'
        i++

        // Read bulk string length
        lenStart := i
        for data[i] != '\r' {
            i++
        }
        strLen, _ := strconv.Atoi(string(data[lenStart:i]))
        i += 2  // Skip \r\n

        // Read bulk string content
        cmd.Args[idx] = string(data[i : i+strLen])
        i += strLen + 2  // Skip content + \r\n
    }

    bytesRead := i - startOffset
    return cmd, bytesRead
}
```

#### 3. Buffered Response Writing

**Old: Direct Write:**
```go
func evalAndRespond(cmd *redisCommand, conn net.Conn) {
    switch strings.ToUpper(cmd.Args[0]) {
    case "PING":
        conn.Write([]byte("+PONG\r\n"))  // Direct write
    case "SET":
        // ... process SET
        conn.Write([]byte("+OK\r\n"))    // Direct write
    }
}
```

**New: Buffered Write:**
```go
func evalAndRespond(cmds []*redisCommand, conn net.Conn) {
    var buffer bytes.Buffer  // In-memory buffer

    for _, cmd := range cmds {
        // Evaluate command and get response bytes
        response := evaluateCommand(cmd)
        buffer.Write(response)  // Append to buffer
    }

    // Single write to socket with all responses
    conn.Write(buffer.Bytes())
}

func evaluateCommand(cmd *redisCommand) []byte {
    switch strings.ToUpper(cmd.Args[0]) {
    case "PING":
        return []byte("+PONG\r\n")
    case "SET":
        key, value := cmd.Args[1], cmd.Args[2]
        store[key] = value
        return []byte("+OK\r\n")
    case "GET":
        key := cmd.Args[1]
        if val, exists := store[key]; exists {
            return resp.EncodeBulkString(val)
        }
        return resp.EncodeNullBulkString()
    default:
        return resp.EncodeError("ERR unknown command")
    }
}
```

---

### Performance Benefits

#### Reduced Context Switches

**Without Pipelining (3 commands):**
```
1. Read from socket      (syscall, context switch)
2. Process command 1     (CPU)
3. Write to socket       (syscall, context switch)
4. Read from socket      (syscall, context switch)
5. Process command 2     (CPU)
6. Write to socket       (syscall, context switch)
7. Read from socket      (syscall, context switch)
8. Process command 3     (CPU)
9. Write to socket       (syscall, context switch)

Total: 6 context switches
```

**With Pipelining (3 commands):**
```
1. Read from socket      (syscall, context switch)
2. Process command 1     (CPU)
3. Process command 2     (CPU)
4. Process command 3     (CPU)
5. Write to socket       (syscall, context switch)

Total: 2 context switches
```

**Efficiency Gain:**
- **3× fewer syscalls**
- **3× fewer context switches**
- Continuous CPU execution (better cache locality)

#### Memory Trade-off

**Server-side Buffering Cost:**

For a pipeline with N commands:
```
Memory = N × avg_response_size
```

**Example:**
- 100 commands pipelined
- Average response size: 50 bytes
- **Buffer memory**: 100 × 50 = 5KB

**Practical Limit:**
- Redis handles pipelines of 10,000+ commands
- But large pipelines consume server memory
- Recommended: 100-1,000 commands per pipeline

---

### Testing Pipelining

**Using `netcat` and `printf`:**

```bash
# Build pipelined request
printf "*1\r\n\$4\r\nPING\r\n*3\r\n\$3\r\nSET\r\n\$1\r\nk\r\n\$1\r\nv\r\n*2\r\n\$3\r\nGET\r\n\$1\r\nk\r\n" | nc localhost 6379
```

**Expected output:**
```
+PONG
+OK
$1
v
```

**Using Redis CLI with `--pipe`:**

```bash
# Create commands file
cat > commands.txt << EOF
SET key1 value1
SET key2 value2
GET key1
GET key2
EOF

# Pipe to Redis
cat commands.txt | redis-cli --pipe
```

**Benchmark with `redis-benchmark`:**

```bash
# Without pipelining
redis-benchmark -t set,get -n 100000 -q
# SET: 45000 requests per second
# GET: 48000 requests per second

# With pipelining (16 commands per pipeline)
redis-benchmark -t set,get -n 100000 -P 16 -q
# SET: 580000 requests per second (13× faster)
# GET: 620000 requests per second (13× faster)
```

---

### Key Differences: Pipelining vs Transactions

| Aspect | Pipelining | Transactions (MULTI/EXEC) |
|--------|-----------|--------------------------|
| **Atomicity** | ❌ No | ✅ Yes (all-or-nothing) |
| **Isolation** | ❌ No | ✅ Yes (no interleaving) |
| **Command Order** | ✅ Sequential | ✅ Sequential |
| **Error Handling** | Each command independent | Entire transaction fails |
| **Performance** | High throughput | Moderate (additional overhead) |
| **Use Case** | Bulk operations | Consistent state updates |

**Example showing independence:**

```bash
# Pipeline
SET counter 10
INCR counter
INCR invalid_type  # This fails
INCR counter

# Results:
# +OK
# :11
# -ERR wrong type
# :12   ← Counter still incremented despite previous error
```

---

### Summary

**What Pipelining Provides:**
1. ✅ **Reduced RTT**: Send multiple commands in one network request
2. ✅ **Higher Throughput**: 10-100× improvement for batch operations
3. ✅ **Fewer Context Switches**: Less syscall overhead
4. ✅ **Better CPU Utilization**: Continuous processing

**Implementation Changes:**
1. ✅ `readCommands()`: Decode multiple concatenated RESP commands
2. ✅ `decodeOne()`: Parse single command, return bytes consumed
3. ✅ `evaluateCommand()`: Return response bytes instead of direct write
4. ✅ Buffered response: Accumulate all responses before single write

**Trade-offs:**
- ⚠️ Server memory consumption (buffering responses)
- ⚠️ No atomicity (commands execute independently)
- ⚠️ Client complexity (batch management)

**When to Use:**
- ✅ Bulk inserts
- ✅ Batch reads
- ✅ High-latency networks
- ❌ Avoid for operations requiring immediate feedback

**Next:** AOF persistence for durability

---

## Chapter 13: AOF Persistence (Append-Only File)

### What is Persistence in Redis?

Redis is often described as an **in-memory database**, but it offers multiple **persistence mechanisms** to flush data to disk, preventing data loss on crashes or restarts.

**Two Persistence Modes:**
1. **RDB (Redis Database)**: Point-in-time snapshots
2. **AOF (Append-Only File)**: Command logging (this chapter)

### RDB vs AOF: A Comparison

#### RDB (Snapshot-based)

**What it does:**
- Creates a complete dump of the entire dataset at a specific point in time
- Stored as a single binary file (e.g., `dump.rdb`)

**How it works:**
```
┌─────────────────────┐
│   Redis Memory      │
│  ┌──────────────┐   │      BGSAVE         ┌──────────────┐
│  │ key1: value1 │   │  ================>  │  dump.rdb    │
│  │ key2: value2 │   │   (fork process)    │  (binary)    │
│  │ key3: value3 │   │                     └──────────────┘
│  └──────────────┘   │
└─────────────────────┘
```

**Mechanism (BGSAVE):**
```go
func BGSAVE() {
    // Fork a child process
    pid := fork()

    if pid == 0 {  // Child process
        // Child has copy-on-write access to parent's memory
        rdbFile := createRDBFile("dump.rdb")

        // Iterate entire dataset
        for key, value := range store {
            rdbFile.Write(encodeRDB(key, value))
        }

        rdbFile.Close()
        os.Exit(0)
    }

    // Parent continues serving requests
    return "+Background saving started\r\n"
}
```

**Pros:**
- ✅ **Compact**: Single binary file, highly compressed
- ✅ **Fast recovery**: Loading RDB is faster than replaying commands
- ✅ **Portable**: Easy to copy to S3, backup services

**Cons:**
- ❌ **Data loss**: Crash between snapshots loses all intermediate writes
- ❌ **CPU/Disk intensive**: Full dump is expensive for large datasets
- ❌ **Infrequent**: Typically run every 5-60 minutes

**Data Loss Example:**
```
Time    Event                    RDB State
----    -----                    ---------
00:00   BGSAVE triggered         ✓ dump.rdb (0 keys)
00:01   SET key1 val1
00:02   SET key2 val2
00:03   SET key3 val3
00:04   ⚠️ CRASH                  ✗ Lost 3 keys (last dump was at 00:00)
```

---

#### AOF (Command Logging)

**What it does:**
- Logs every **write command** to a file
- Appends commands to `appendonly.aof`
- Skips read-only commands (GET, SCAN, etc.)

**How it works:**
```
Client Commands              AOF File (appendonly.aof)
---------------              -------------------------
SET key1 val1      ──────>   *3\r\n$3\r\nSET\r\n$4\r\nkey1\r\n$4\r\nval1\r\n
SET key2 val2      ──────>   *3\r\n$3\r\nSET\r\n$4\r\nkey2\r\n$4\r\nval2\r\n
INCR counter       ──────>   *3\r\n$3\r\nSET\r\n$7\r\ncounter\r\n$1\r\n5\r\n
DEL key1           ──────>   *2\r\n$3\r\nDEL\r\n$4\r\nkey1\r\n
```

**Pros:**
- ✅ **High durability**: Minimal data loss (configurable: 1 second)
- ✅ **Append-only**: Simple, sequential writes (fast)
- ✅ **Human-readable**: RESP format, can inspect/edit manually

**Cons:**
- ❌ **Large file size**: Logs all operations, not just final state
- ❌ **Slow recovery**: Must replay all commands on restart
- ❌ **Redundancy**: Multiple updates to same key create bloat

---

### AOF File Format

AOF files store commands in **RESP (REdis Serialization Protocol)** format:

**Example Commands:**
```bash
SET name Alice
SET age 30
INCR age
DEL name
```

**Resulting AOF file:**
```
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$5\r\nAlice\r\n
*3\r\n$3\r\nSET\r\n$3\r\nage\r\n$2\r\n30\r\n
*3\r\n$3\r\nSET\r\n$3\r\nage\r\n$2\r\n31\r\n
*2\r\n$3\r\nDEL\r\n$4\r\nname\r\n
```

**Why RESP format?**
1. **Simple replay**: Feed file contents directly to command parser
2. **Debuggable**: Can read/edit with text editor
3. **Portable**: Standard Redis protocol

---

### Command Normalization

Redis **normalizes** complex commands to simpler equivalents before logging:

**Example: INCR command**

```bash
# Client executes
INCR counter

# If counter was 4, after INCR it becomes 5
# AOF logs the SET command (normalized form):
SET counter 5
```

**Why normalize?**
- **Idempotency**: Replaying `SET counter 5` always gives same result
- **Simplicity**: Don't need special replay logic for INCR
- **Correctness**: Avoids issues with concurrent modifications

**Other normalization examples:**

| Original Command | Logged to AOF |
|-----------------|---------------|
| `INCR counter` | `SET counter 6` |
| `APPEND msg " world"` | `SET msg "hello world"` |
| `SETEX key 10 val` | `SET key val` + `EXPIREAT key 1234567890` |

---

### AOF Rewriting (BGREWRITEAOF)

#### The Bloat Problem

**Scenario:**
```bash
SET user:1:name "Alice"
SET user:1:name "Bob"
SET user:1:name "Charlie"
SET user:1:name "David"
```

**AOF file contains:**
```
*3\r\n$3\r\nSET\r\n$11\r\nuser:1:name\r\n$5\r\nAlice\r\n
*3\r\n$3\r\nSET\r\n$11\r\nuser:1:name\r\n$3\r\nBob\r\n
*3\r\n$3\r\nSET\r\n$11\r\nuser:1:name\r\n$7\r\nCharlie\r\n
*3\r\n$3\r\nSET\r\n$11\r\nuser:1:name\r\n$5\r\nDavid\r\n
```

**But final state only needs:**
```
*3\r\n$3\r\nSET\r\n$11\r\nuser:1:name\r\n$5\r\nDavid\r\n
```

**File size: 300 bytes → 75 bytes (75% reduction)**

---

#### BGREWRITEAOF Implementation

**Process:**

```
┌────────────────────────────────────────────────┐
│ Step 1: Trigger BGREWRITEAOF                   │
│ ────────────────────────────────────────────── │
│ Client sends command or auto-trigger by config │
└────────────────────────────────────────────────┘
                      ↓
┌────────────────────────────────────────────────┐
│ Step 2: Iterate Current Dataset                │
│ ────────────────────────────────────────────── │
│ for key, value := range store {                │
│     writeToTempAOF("SET", key, value)          │
│ }                                              │
└────────────────────────────────────────────────┘
                      ↓
┌────────────────────────────────────────────────┐
│ Step 3: Write to Temporary File                │
│ ────────────────────────────────────────────── │
│ temp file: appendonly.aof.tmp                  │
│ Contains minimal commands for current state    │
└────────────────────────────────────────────────┘
                      ↓
┌────────────────────────────────────────────────┐
│ Step 4: Atomic Rename                          │
│ ────────────────────────────────────────────── │
│ os.Rename("appendonly.aof.tmp",                │
│           "appendonly.aof")                    │
└────────────────────────────────────────────────┘
```

**Go Implementation:**

```go
// evalBGREWRITEAOF handles the BGREWRITEAOF command
func evalBGREWRITEAOF(args []string) []byte {
    // TODO: Should fork() for async execution
    // For now, synchronous implementation

    err := dumpAllAOF()
    if err != nil {
        return resp.EncodeError("ERR " + err.Error())
    }

    return resp.EncodeSimpleString("Background AOF rewrite started")
}

// dumpAllAOF rewrites the entire AOF file
func dumpAllAOF() error {
    log.Println("Rewriting AOF file at", config.AOFPath)

    // Open temp file for writing
    tempPath := config.AOFPath + ".tmp"
    file, err := os.OpenFile(tempPath, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, 0644)
    if err != nil {
        return err
    }
    defer file.Close()

    // Iterate over all keys in memory
    for key, obj := range store {
        // Skip expired keys
        if obj.ExpiresAt > 0 && time.Now().UnixMilli() >= obj.ExpiresAt {
            continue
        }

        // Construct SET command
        cmd := []string{"SET", key, obj.Value.(string)}

        // Encode as RESP array
        respCmd := encodeRESPArray(cmd)

        // Write to file
        _, err := file.Write(respCmd)
        if err != nil {
            return err
        }

        // If key has expiration, write EXPIREAT
        if obj.ExpiresAt > 0 {
            expireCmd := []string{
                "EXPIREAT",
                key,
                strconv.FormatInt(obj.ExpiresAt/1000, 10),
            }
            file.Write(encodeRESPArray(expireCmd))
        }
    }

    // Atomic rename
    err = os.Rename(tempPath, config.AOFPath)
    if err != nil {
        return err
    }

    log.Println("AOF file rewrite complete")
    return nil
}

// encodeRESPArray converts ["SET", "key", "value"] to RESP format
func encodeRESPArray(args []string) []byte {
    var buf bytes.Buffer

    // Array header: *3\r\n
    buf.WriteString(fmt.Sprintf("*%d\r\n", len(args)))

    // Each element as bulk string
    for _, arg := range args {
        buf.WriteString(fmt.Sprintf("$%d\r\n%s\r\n", len(arg), arg))
    }

    return buf.Bytes()
}
```

**Example Execution:**

```bash
# Connect to Redis
$ redis-cli

# Perform multiple operations
127.0.0.1:6379> SET k1 v1
OK
127.0.0.1:6379> SET k2 v2
OK
127.0.0.1:6379> SET k3 v4
OK
127.0.0.1:6379> SET k3 v3
OK

# Trigger rewrite
127.0.0.1:6379> BGREWRITEAOF
Background AOF rewrite started

# Server logs:
# Rewriting AOF file at ./appendonly.aof
# AOF file rewrite complete

# Check file contents
$ cat appendonly.aof
*3
$3
SET
$2
k1
$2
v1
*3
$3
SET
$2
k2
$2
v2
*3
$3
SET
$2
k3
$2
v3
```

**Notice:** Only 3 SET commands (not 4) — the duplicate `k3` was optimized.

---

### AOF Validation and Repair

Redis provides `redis-check-aof` utility to validate and fix corrupted AOF files.

**Check file integrity:**
```bash
$ redis-check-aof appendonly.aof
AOF analyzed
Status: OK
```

**Fix corrupted file:**
```bash
# Corrupt the file (e.g., incomplete write during crash)
$ echo "garbage data" >> appendonly.aof

# Check again
$ redis-check-aof appendonly.aof
AOF analyzed
Status: NOT OK
...

# Repair (removes invalid commands)
$ redis-check-aof --fix appendonly.aof
AOF analyzed
Truncated 15 bytes from file
Status: OK (repaired)
```

---

### Durability Guarantees

AOF supports three **fsync policies** controlling when data is flushed from OS buffer to disk:

| Policy | Description | Durability | Performance |
|--------|-------------|-----------|-------------|
| **always** | fsync after every write | Highest (0 loss) | Slowest |
| **everysec** | fsync once per second | High (≤1s loss) | Fast |
| **no** | OS decides when to fsync | Low (minutes loss) | Fastest |

**Configuration:**
```bash
# redis.conf
appendfsync everysec
```

**Trade-off Example (1M writes/sec):**

- **always**: 500-1000 writes/sec (fsync bottleneck)
- **everysec**: 50,000+ writes/sec (good balance)
- **no**: 100,000+ writes/sec (risky)

---

### Loading AOF on Startup

**Startup Sequence:**

```go
func loadAOF(path string) error {
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()

    log.Println("Loading AOF file:", path)

    scanner := bufio.NewScanner(file)
    var commandBuffer []byte

    for scanner.Scan() {
        commandBuffer = append(commandBuffer, scanner.Bytes()...)
        commandBuffer = append(commandBuffer, '\n')

        // Try to decode complete command
        cmd, bytesRead := decodeOne(commandBuffer)
        if cmd != nil {
            // Execute command
            evaluateCommand(cmd)

            // Remove processed bytes
            commandBuffer = commandBuffer[bytesRead:]
        }
    }

    log.Println("AOF loaded successfully")
    return nil
}
```

**Startup Flow:**
```
1. Server starts
2. Check if appendonly.aof exists
3. If exists:
   a. Read file line by line
   b. Decode RESP commands
   c. Execute commands in memory
4. Server ready to accept connections
```

---

### Summary

**RDB vs AOF Comparison:**

| Feature | RDB | AOF |
|---------|-----|-----|
| **File Size** | Small (binary) | Large (text commands) |
| **Recovery Speed** | Fast | Slow (replay commands) |
| **Data Loss** | Minutes | Seconds |
| **CPU Impact** | High (during fork) | Low (append-only) |
| **Debuggability** | Opaque binary | Human-readable |

**When to Use Each:**

- **RDB only**: Acceptable to lose minutes of data, need fast restarts
- **AOF only**: Need high durability, can tolerate slower restarts
- **Both**: Maximum durability + fast recovery (load RDB, replay AOF since snapshot)

**What We Implemented:**
1. ✅ `BGREWRITEAOF` command
2. ✅ Iterate dataset, write minimal SET commands
3. ✅ RESP encoding for AOF format
4. ✅ Atomic file replacement
5. ✅ Command normalization (INCR → SET)

**Production Enhancements (TODO):**
- ⏳ Async forking (non-blocking BGREWRITEAOF)
- ⏳ Incremental AOF (log commands as they arrive)
- ⏳ Auto-rewrite triggers (size/percentage thresholds)
- ⏳ Mixed persistence (RDB + AOF)

**Next:** Object encodings and the INCR command

---

## Chapter 14: Objects, Encodings, and INCR Command

### The Redis Object Model

Redis stores all values as **`redisObject`** instances, a generic structure that wraps all data types with metadata for efficient memory management and flexible encoding.

#### Why a Generic Object Model?

**Problem:** Different data structures need different metadata:
- Strings need length tracking
- Lists need element count
- Sets need cardinality estimation
- All need expiration tracking
- All need memory management

**Solution:** Single unified structure (`redisObject`) with:
- Type abstraction
- Encoding polymorphism
- Reference counting
- LRU tracking

---

### The redisObject Structure

**C Structure (Redis source):**
```c
typedef struct redisObject {
    unsigned type:4;        // 4 bits: object type
    unsigned encoding:4;    // 4 bits: encoding/implementation
    unsigned lru:24;        // 24 bits: LRU time or LFU data
    int refcount;           // 4 bytes: reference count
    void *ptr;              // 8 bytes: pointer to actual data
} robj;
```

**Memory Layout:**
```
┌─────────────────────────────────────────────┐
│ Byte 0: [type:4][encoding:4]               │  1 byte
│ Bytes 1-3: [lru:24]                        │  3 bytes
│ Bytes 4-7: [refcount:32]                   │  4 bytes
│ Bytes 8-15: [ptr:64]                       │  8 bytes
├─────────────────────────────────────────────┤
│ TOTAL SIZE: 16 bytes (on 64-bit systems)   │
└─────────────────────────────────────────────┘
```

**Note:** Original struct is 12 bytes in Redis (32-bit ptr), but modern 64-bit systems use 16 bytes.

---

### Bitfields Optimization

#### What are Bitfields?

**Bitfields** (C/C++ feature) allow allocating specific numbers of bits to struct members, saving memory:

```c
struct Example {
    unsigned type:4;      // Only 4 bits (values 0-15)
    unsigned encoding:4;  // Only 4 bits (values 0-15)
    unsigned lru:24;      // 24 bits (values 0-16,777,215)
};
```

**Memory Savings:**

| Without Bitfields | With Bitfields |
|------------------|----------------|
| `unsigned type` (4 bytes) | `type:4` (4 bits) |
| `unsigned encoding` (4 bytes) | `encoding:4` (4 bits) |
| `unsigned lru` (4 bytes) | `lru:24` (24 bits) |
| **Total: 12 bytes** | **Total: 4 bytes** |

**Savings: 67% memory reduction (8 bytes saved per object)**

For 1 million objects: **8 MB saved**

---

### Go Implementation (No Native Bitfields)

Go doesn't support bitfields, so we combine `type` and `encoding` into a single `uint8`:

```go
type Object struct {
    typeEncoding uint8       // Combined: [type:4][encoding:4]
    lru          uint32      // 24 bits used (stored as 32-bit for alignment)
    refcount     int32       // Reference count
    value        interface{} // Pointer to actual data
    expiresAt    int64       // Expiration timestamp (ms)
}

// Extract type (upper 4 bits)
func (o *Object) Type() ObjectType {
    return ObjectType(o.typeEncoding >> 4)
}

// Extract encoding (lower 4 bits)
func (o *Object) Encoding() ObjectEncoding {
    return ObjectEncoding(o.typeEncoding & 0x0F)
}

// Set type and encoding
func (o *Object) SetTypeEncoding(objType ObjectType, encoding ObjectEncoding) {
    o.typeEncoding = (uint8(objType) << 4) | uint8(encoding)
}
```

**Bitwise Operations Explained:**

```
typeEncoding = 0b0011_0010
               └┬─┘ └┬─┘
                │    └─ encoding = 2 (lower 4 bits)
                └────── type = 3 (upper 4 bits)

// Extract type
type = typeEncoding >> 4
     = 0b0011_0010 >> 4
     = 0b0000_0011
     = 3

// Extract encoding
encoding = typeEncoding & 0x0F
         = 0b0011_0010 & 0b0000_1111
         = 0b0000_0010
         = 2

// Set typeEncoding
typeEncoding = (type << 4) | encoding
             = (3 << 4) | 2
             = 0b0011_0000 | 0b0000_0010
             = 0b0011_0010
```

---

### Object Types

Redis supports a **limited set** of abstract types (4 bits = max 16 types):

```go
type ObjectType uint8

const (
    OBJ_STRING ObjectType = iota  // 0
    OBJ_LIST                      // 1
    OBJ_SET                       // 2
    OBJ_ZSET                      // 3 (sorted set)
    OBJ_HASH                      // 4
    OBJ_MODULE                    // 5 (custom modules)
    // 6-15 reserved for future types
)
```

**Key Insight:** Integers are **NOT** a distinct type in Redis. They're stored as `OBJ_STRING` with `OBJ_ENCODING_INT`.

---

### Object Encodings

Each type can have **multiple implementations** (encodings), optimized for different scenarios:

```go
type ObjectEncoding uint8

const (
    OBJ_ENCODING_RAW      ObjectEncoding = iota  // 0: Raw bytes
    OBJ_ENCODING_INT                             // 1: Integer
    OBJ_ENCODING_HT                              // 2: Hash table
    OBJ_ENCODING_ZIPLIST                         // 3: Ziplist
    OBJ_ENCODING_INTSET                          // 4: Integer set
    OBJ_ENCODING_SKIPLIST                        // 5: Skip list
    OBJ_ENCODING_EMBSTR                          // 6: Embedded string
    OBJ_ENCODING_QUICKLIST                       // 7: Quicklist
    OBJ_ENCODING_LISTPACK                        // 8: Listpack
)
```

#### Encodings by Type

**OBJ_STRING:**
- `OBJ_ENCODING_RAW`: General strings, binary data
- `OBJ_ENCODING_INT`: Strings representing integers
- `OBJ_ENCODING_EMBSTR`: Small strings (≤44 bytes)

**OBJ_LIST:**
- `OBJ_ENCODING_ZIPLIST`: Small lists (<512 elements)
- `OBJ_ENCODING_LINKEDLIST`: Large lists
- `OBJ_ENCODING_QUICKLIST`: Hybrid (default in modern Redis)

**OBJ_SET:**
- `OBJ_ENCODING_INTSET`: All elements are small integers
- `OBJ_ENCODING_HT`: General sets (hash table)

**OBJ_ZSET:**
- `OBJ_ENCODING_ZIPLIST`: Small sorted sets
- `OBJ_ENCODING_SKIPLIST`: Large sorted sets

**OBJ_HASH:**
- `OBJ_ENCODING_ZIPLIST`: Small hashes
- `OBJ_ENCODING_HT`: Large hashes

---

### Dynamic Encoding Conversion

Redis **automatically converts** encodings when thresholds are crossed:

**Example: List Encoding Transition**

```
Small list (< 512 elements):
OBJ_LIST + OBJ_ENCODING_ZIPLIST
  ↓ (insert 513th element)
Large list (>= 512 elements):
OBJ_LIST + OBJ_ENCODING_LINKEDLIST
```

**Why?**
- **Ziplist**: Memory-efficient, slower access
- **Linked list**: Memory-intensive, faster access
- **Trade-off**: Use ziplist until performance degrades

**Threshold Configuration:**
```bash
# redis.conf
list-max-ziplist-entries 512
list-max-ziplist-value 64
```

---

### String Encodings Deep Dive

#### OBJ_ENCODING_INT

**Used when:** String value is a valid 64-bit signed integer

```go
func deduceTypeEncoding(value string) (ObjectType, ObjectEncoding) {
    // Try to parse as integer
    if _, err := strconv.ParseInt(value, 10, 64); err == nil {
        return OBJ_STRING, OBJ_ENCODING_INT
    }

    // Otherwise, check length
    if len(value) <= 44 {
        return OBJ_STRING, OBJ_ENCODING_EMBSTR
    }

    return OBJ_STRING, OBJ_ENCODING_RAW
}
```

**Example:**
```bash
SET counter "42"
# Stored as: type=OBJ_STRING, encoding=OBJ_ENCODING_INT
# value pointer stores "42" (can be optimized to store int64 directly)
```

---

#### OBJ_ENCODING_EMBSTR

**Used when:** String length ≤ 44 bytes

**Why 44 bytes?**

```
Redis object overhead:
  - typeEncoding: 1 byte
  - lru:          3 bytes
  - refcount:     4 bytes
  - ptr:          8 bytes
  - TOTAL:        16 bytes

Typical allocation size: 64 bytes

Available for string:
  64 bytes (allocation)
  - 16 bytes (redisObject)
  - 3 bytes (SDS header for length/capacity)
  - 1 byte (null terminator)
  = 44 bytes
```

**Memory Layout:**

```
┌────────────────────────────────────────────┐
│ redisObject (16 bytes)                     │
├────────────────────────────────────────────┤
│ SDS Header (3 bytes)                       │
├────────────────────────────────────────────┤
│ String Data (up to 44 bytes)               │
├────────────────────────────────────────────┤
│ Null Terminator (1 byte)                   │
└────────────────────────────────────────────┘
  Total: 64 bytes (fits in single allocation)
```

**Benefits:**
- ✅ Single malloc (not two: object + string)
- ✅ Better cache locality
- ✅ Reduced memory fragmentation

---

#### OBJ_ENCODING_RAW

**Used when:** String length > 44 bytes OR binary data

**Memory Layout:**

```
┌────────────────────┐
│ redisObject        │
│   ptr ────────┐    │
└───────────────│────┘
                │
                ↓
        ┌──────────────────┐
        │ SDS Structure    │
        │ - len            │
        │ - capacity       │
        │ - data[]         │
        └──────────────────┘
```

**Example:**
```bash
SET longkey "This is a very long string that exceeds 44 bytes..."
# Stored as: type=OBJ_STRING, encoding=OBJ_ENCODING_RAW
# ptr points to separate SDS allocation
```

---

### INCR Command Implementation

#### Behavior Specification

```bash
# Key doesn't exist → initialize to 0, then increment
INCR counter
# Result: 1

# Key exists with integer-encoded string
SET counter "5"
INCR counter
# Result: 6

# Key exists with non-integer string
SET counter "hello"
INCR counter
# Error: "ERR value is not an integer or out of range"
```

---

#### Implementation

```go
func evalINCR(args []string) []byte {
    if len(args) != 1 {
        return resp.EncodeError("ERR wrong number of arguments for 'INCR' command")
    }

    key := args[0]

    // Get existing object or create new one
    obj, exists := store[key]

    var currentValue int64

    if !exists {
        // Key doesn't exist → initialize to 0
        currentValue = 0
    } else {
        // Validate type
        if obj.Type() != OBJ_STRING {
            return resp.EncodeError("WRONGTYPE Operation against a key holding the wrong kind of value")
        }

        // Validate encoding
        if obj.Encoding() != OBJ_ENCODING_INT {
            return resp.EncodeError("ERR value is not an integer or out of range")
        }

        // Parse current value
        var err error
        currentValue, err = strconv.ParseInt(obj.value.(string), 10, 64)
        if err != nil {
            return resp.EncodeError("ERR value is not an integer or out of range")
        }
    }

    // Increment
    currentValue++

    // Store back as string with INT encoding
    newObj := &Object{
        value:     strconv.FormatInt(currentValue, 10),
        expiresAt: 0,
    }
    newObj.SetTypeEncoding(OBJ_STRING, OBJ_ENCODING_INT)

    store[key] = newObj

    // Return integer (not string)
    return resp.EncodeInteger(currentValue)
}
```

---

#### Performance Considerations

**String ↔ Integer Conversion Cost:**

```go
// Every INCR operation requires:
1. ParseInt(string → int64)     // ~50-100 ns
2. Increment (int64++)          // ~1 ns
3. FormatInt(int64 → string)    // ~50-100 ns

Total: ~100-200 ns per INCR
```

**Alternative: Store int64 directly in ptr**

Redis creator (Antirez) acknowledged this overhead but chose simplicity:
- **Pro**: Unified type system (all strings)
- **Con**: Repeated conversions for numeric operations

**Potential optimization** (GitHub issue):
- Store integers directly as `int64` in `ptr` field
- Avoid string conversions
- **Estimated speedup**: 2-5× for numeric operations

---

### SET Command with Type Deduction

```go
func evalSET(args []string) []byte {
    if len(args) < 2 {
        return resp.EncodeError("ERR wrong number of arguments for 'SET' command")
    }

    key, value := args[0], args[1]

    // Deduce optimal encoding
    objType, encoding := deduceTypeEncoding(value)

    obj := &Object{
        value:     value,
        expiresAt: 0,
    }
    obj.SetTypeEncoding(objType, encoding)

    // Parse expiration options (EX, PX, etc.)
    for i := 2; i < len(args); i++ {
        option := strings.ToUpper(args[i])
        switch option {
        case "EX":  // Seconds
            seconds, _ := strconv.ParseInt(args[i+1], 10, 64)
            obj.expiresAt = time.Now().UnixMilli() + (seconds * 1000)
            i++
        case "PX":  // Milliseconds
            ms, _ := strconv.ParseInt(args[i+1], 10, 64)
            obj.expiresAt = time.Now().UnixMilli() + ms
            i++
        }
    }

    store[key] = obj
    return resp.EncodeSimpleString("OK")
}

func deduceTypeEncoding(value string) (ObjectType, ObjectEncoding) {
    // Try integer
    if _, err := strconv.ParseInt(value, 10, 64); err == nil {
        return OBJ_STRING, OBJ_ENCODING_INT
    }

    // Try embedded string
    if len(value) <= 44 {
        return OBJ_STRING, OBJ_ENCODING_EMBSTR
    }

    // Default to raw
    return OBJ_STRING, OBJ_ENCODING_RAW
}
```

**Example:**
```bash
SET name "Alice"          # → OBJ_STRING + OBJ_ENCODING_EMBSTR (≤44 bytes)
SET age "25"              # → OBJ_STRING + OBJ_ENCODING_INT (valid integer)
SET bio "Long text..."    # → OBJ_STRING + OBJ_ENCODING_RAW (>44 bytes)
```

---

### Extensibility: Adding New Data Structures

The type/encoding split makes Redis highly extensible:

**Example: Adding Bloom Filter**

```go
// Use existing type
const OBJ_ENCODING_BLOOMFILTER ObjectEncoding = 9

// Implement as byte array (similar to OBJ_ENCODING_RAW)
func evalBFADD(args []string) []byte {
    key, item := args[0], args[1]

    obj, exists := store[key]
    if !exists {
        // Create new Bloom filter
        bf := bloomfilter.New(10000, 5)  // 10k items, 5 hash functions

        obj = &Object{
            value: bf,
        }
        obj.SetTypeEncoding(OBJ_STRING, OBJ_ENCODING_BLOOMFILTER)
        store[key] = obj
    }

    // Type check
    if obj.Encoding() != OBJ_ENCODING_BLOOMFILTER {
        return resp.EncodeError("WRONGTYPE Operation against a key holding the wrong kind of value")
    }

    // Add to Bloom filter
    bf := obj.value.(*bloomfilter.BloomFilter)
    bf.Add(item)

    return resp.EncodeInteger(1)
}
```

**Benefits:**
- No changes to core `redisObject` structure
- Reuses existing infrastructure (expiration, LRU, refcount)
- Easy to contribute new optimized encodings

---

### Summary

**redisObject Design Principles:**
1. ✅ **Unified Structure**: All values use same object wrapper
2. ✅ **Type Abstraction**: Separate logical type from implementation
3. ✅ **Encoding Polymorphism**: Multiple implementations per type
4. ✅ **Memory Optimization**: Bitfields save 8 bytes per object
5. ✅ **Dynamic Conversion**: Automatic encoding upgrades

**Encoding Strategies:**

| Encoding | Use Case | Memory | Performance |
|----------|----------|--------|-------------|
| `OBJ_ENCODING_INT` | Integer strings | Low | Fast (numeric ops) |
| `OBJ_ENCODING_EMBSTR` | Small strings (≤44B) | Low | Fast (single alloc) |
| `OBJ_ENCODING_RAW` | Large strings/binary | Medium | Moderate |
| `OBJ_ENCODING_ZIPLIST` | Small collections | Very Low | Slow (linear scan) |
| `OBJ_ENCODING_LINKEDLIST` | Large collections | High | Fast (O(1) ops) |

**INCR Implementation:**
1. ✅ Check key existence
2. ✅ Validate type = OBJ_STRING
3. ✅ Validate encoding = OBJ_ENCODING_INT
4. ✅ Parse string to int64
5. ✅ Increment
6. ✅ Convert back to string
7. ✅ Store with INT encoding

**Trade-offs:**
- ⚠️ String ↔ Integer conversion overhead (100-200ns)
- ⚠️ Bitfields not available in Go (use manual bit manipulation)
- ✅ Simplified type system (no dedicated int type)
- ✅ Extensibility (easy to add new encodings)

**Next:** INFO command and advanced eviction strategies

---

## Chapter 15: INFO Command and Monitoring

### What is the INFO Command?

The **INFO command** is Redis's built-in monitoring and observability tool that returns real-time statistics about the server's state, memory usage, connected clients, and more.

**Purpose:**
- Monitor database health
- Set up alerts and dashboards
- Debug performance issues
- Track key metrics over time

**Basic usage:**
```bash
redis-cli INFO
```

---

### INFO Command Format

**Response Structure:**

```
# Keyspace
db0:keys=1000,expires=50,avg_ttl=3600000
db1:keys=0,expires=0,avg_ttl=0

# Server
redis_version:7.0.0
redis_mode:standalone
os:Linux 5.10.0-x86_64
uptime_in_seconds:86400

# Memory
used_memory:1048576
used_memory_human:1.00M
maxmemory:2097152
maxmemory_human:2.00M
```

**Format Specification:**
- Each section starts with `# Section_Name`
- Followed by `key:value` pairs
- Each line terminated with `\r\n`
- Values can be comma-separated (e.g., `keys=1000,expires=50`)

---

### Key Sections and Metrics

#### 1. Keyspace Section

**Most Important Metrics:**

| Metric | Description | Example |
|--------|-------------|---------|
| `keys` | Total number of keys in database | `keys=10000` |
| `expires` | Number of keys with expiration set | `expires=250` |
| `avg_ttl` | Average time-to-live in milliseconds | `avg_ttl=3600000` |

**Example:**
```bash
127.0.0.1:6379> INFO keyspace
# Keyspace
db0:keys=15,expires=3,avg_ttl=120000
```

#### 2. Memory Section

```bash
used_memory:5242880          # 5 MB in bytes
used_memory_human:5.00M      # Human-readable format
used_memory_rss:6291456      # Resident set size (OS perspective)
maxmemory:10485760           # Memory limit (10 MB)
maxmemory_policy:allkeys-lru # Eviction policy
mem_fragmentation_ratio:1.20 # Memory fragmentation
```

#### 3. Stats Section

```bash
total_connections_received:1000
total_commands_processed:50000
instantaneous_ops_per_sec:100
keyspace_hits:8000
keyspace_misses:2000
evicted_keys:150
```

**Key Derived Metrics:**
```
Hit Rate = keyspace_hits / (keyspace_hits + keyspace_misses)
        = 8000 / (8000 + 2000)
        = 80%
```

---

### Implementation

#### Go Implementation

```go
// Global stats tracker
var keyspaceStats = make(map[int]map[string]int64)

func init() {
    // Initialize stats for db0-db3
    for i := 0; i < 4; i++ {
        keyspaceStats[i] = map[string]int64{
            "keys":    0,
            "expires": 0,
            "avg_ttl": 0,
        }
    }
}

// evalINFO handles the INFO command
func evalINFO(args []string) []byte {
    var buf bytes.Buffer

    // Keyspace section
    buf.WriteString("# Keyspace\r\n")

    for dbID := 0; dbID < 4; dbID++ {
        stats := keyspaceStats[dbID]
        buf.WriteString(fmt.Sprintf(
            "db%d:keys=%d,expires=%d,avg_ttl=%d\r\n",
            dbID,
            stats["keys"],
            stats["expires"],
            stats["avg_ttl"],
        ))
    }

    // Return as bulk string
    return resp.EncodeBulkString(buf.String())
}

// Update stats on key operations
func updateKeyspaceStats(dbID int, metric string, delta int64) {
    keyspaceStats[dbID][metric] += delta
}
```

#### Tracking Key Count

**On Key Insert (PUT):**
```go
func Put(key string, obj *Object) {
    store[key] = obj

    // Increment key count
    updateKeyspaceStats(0, "keys", 1)

    // Track expiration
    if obj.ExpiresAt > 0 {
        updateKeyspaceStats(0, "expires", 1)
    }
}
```

**On Key Delete:**
```go
func Delete(key string) bool {
    obj, exists := store[key]
    if !exists {
        return false
    }

    // Decrement key count
    updateKeyspaceStats(0, "keys", -1)

    // Track expiration removal
    if obj.ExpiresAt > 0 {
        updateKeyspaceStats(0, "expires", -1)
    }

    delete(store, key)
    return true
}
```

---

### Monitoring Stack Integration

#### Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Redis     │────>│   Exporter   │────>│  Prometheus  │
│   (DiceDB)   │     │              │     │  (TSDB)      │
└──────────────┘     └──────────────┘     └──────────────┘
   INFO command      Scrapes metrics      Stores time-series
                     every 15s                    │
                                                  │
                                                  ↓
                                          ┌──────────────┐
                                          │   Grafana    │
                                          │  (Visualize) │
                                          └──────────────┘
```

**Components:**

1. **Redis Exporter**: Prometheus exporter that periodically executes `INFO` and exposes metrics
2. **Prometheus**: Time-series database that scrapes and stores metrics
3. **Grafana**: Visualization platform for creating dashboards

---

### Observing Eviction Patterns

#### The Sawtooth Pattern

When running a continuous write workload against a memory-limited Redis instance with eviction enabled, you'll observe a **sawtooth pattern** in the key count graph:

```
Keys
 │
100├───╱╲───╱╲───╱╲───╱╲
 80│  ╱  ╲ ╱  ╲ ╱  ╲ ╱  ╲
 60│ ╱    ╳    ╳    ╳    ╲
 40│╱    ╱ ╲  ╱ ╲  ╱ ╲
   └────────────────────────> Time
```

**Why Sawtooth?**

1. **Fill Phase**: Keys inserted until limit (e.g., 100 keys)
2. **Eviction Trigger**: Memory/key limit exceeded
3. **Eviction Phase**: 40% of keys evicted (40 keys removed)
4. **Repeat**: Back to 60 keys, fill again

---

### Summary

**What INFO Provides:**
1. ✅ Real-time database statistics
2. ✅ Key count, memory usage, hit rates
3. ✅ Server metadata (version, uptime)
4. ✅ Foundation for monitoring stack

**Implementation Details:**
1. ✅ Global stats tracker (`keyspaceStats`)
2. ✅ Update on PUT/DELETE operations
3. ✅ Format as `# Section\r\nkey:value\r\n`
4. ✅ Return as RESP bulk string

**Monitoring Stack:**
1. ✅ Redis Exporter → scrapes INFO every 15s
2. ✅ Prometheus → stores time-series data
3. ✅ Grafana → visualizes metrics

**Next:** Approximated LRU algorithm theory

---
## Chapter 16: The Approximated LRU Algorithm

### What is LRU?

**Least Recently Used (LRU)** is a cache eviction algorithm that removes the least recently accessed items first, based on the principle of **temporal locality**: recently accessed data is more likely to be accessed again soon.

**Ideal LRU Behavior:**

```
Time    Access    Cache State
----    ------    -----------
t=0     SET A     [A]
t=1     SET B     [A, B]
t=2     SET C     [A, B, C]
t=3     GET A     [B, C, A]    ← A moved to front
t=4     SET D     [C, A, D]    ← B evicted (least recent)
```

---

### Why Not Exact LRU?

#### Naive Exact LRU Implementation

**Approach 1: Timestamp + Sorting**

```go
type Object struct {
    Value         interface{}
    LastAccessedAt time.Time
}

func evictLRU() {
    // Sort all keys by LastAccessedAt
    keys := getAllKeys()
    sort.Slice(keys, func(i, j int) bool {
        return keys[i].LastAccessedAt.Before(keys[j].LastAccessedAt)
    })

    // Evict oldest
    delete(store, keys[0].Key)
}
```

**Problems:**
- ❌ **O(N log N) sorting** on every eviction (catastrophic at scale)
- ❌ **32-bit timestamp** per object (4 bytes × 1M keys = 4MB overhead)
- ❌ **No real-time ordering**

---

**Approach 2: Doubly Linked List**

```go
type LRUCache struct {
    capacity int
    cache    map[string]*Node
    head     *Node
    tail     *Node
}

type Node struct {
    Key   string
    Value interface{}
    Prev  *Node
    Next  *Node
}

func (c *LRUCache) Get(key string) interface{} {
    if node, exists := c.cache[key]; exists {
        c.moveToHead(node)  // Move to front
        return node.Value
    }
    return nil
}

func (c *LRUCache) Put(key string, value interface{}) {
    if len(c.cache) >= c.capacity {
        c.removeTail()  // Evict LRU
    }
    // Insert at head
}
```

**Exact LRU Guarantees:**
- ✅ O(1) access and eviction
- ✅ Perfect recency ordering

**Problems for Redis:**
- ❌ **Memory overhead**: 16 bytes per object (2 pointers × 8 bytes)
  - 1M keys → 16MB wasted on pointers alone
- ❌ **CPU overhead**: Constant pointer manipulation on every access
- ❌ **Cache unfriendly**: Random memory access for linked nodes
- ❌ **Fragmentation**: Nodes scattered across memory

**For 1 million keys:**
```
Exact LRU overhead:
  - Timestamp: 4 bytes × 1M = 4MB
  - Pointers: 16 bytes × 1M = 16MB
  - Total: 20MB of metadata (just for LRU!)
```

---

### Redis's Solution: Approximated LRU

#### Core Innovations

1. **24-bit Clock**: Store only 24 bits of timestamp (not 32 or 64)
2. **Sampling**: Don't scan all keys, sample N random keys (typically 5)
3. **Eviction Pool**: Maintain small pool (16 items) of candidates
4. **Simple Array**: Use array instead of doubly linked list

**Benefits:**
- ✅ **3 bytes per object** (not 20 bytes)
- ✅ **No pointer overhead**
- ✅ **No sorting overhead**
- ✅ **Cache-friendly** (array access, not pointer chasing)

---

### The 24-bit LRU Clock

#### Why 24 Bits?

**Memory Layout in `redisObject`:**

```c
struct redisObject {
    unsigned type:4;       // 4 bits
    unsigned encoding:4;   // 4 bits
    unsigned lru:24;       // 24 bits ← LRU clock stored here
    int refcount;          // 32 bits
    void *ptr;             // 64 bits (on 64-bit systems)
};
```

**Total first 32 bits:**
- type (4) + encoding (4) + lru (24) = 32 bits (fits in 1 CPU word)

**If we used 32 bits for LRU:**
- Would need extra 8 bits → breaks alignment
- Wastes 1 byte per object
- 1M objects → 1MB wasted

---

#### Computing the 24-bit Clock

```go
const LRU_CLOCK_MAX = (1 << 24) - 1  // 0xFFFFFF = 16,777,215

func GetLRUClock() uint32 {
    nowSeconds := uint32(time.Now().Unix())
    return nowSeconds & LRU_CLOCK_MAX  // Mask to 24 bits
}
```

**Behavior:**

```
Unix Time (seconds)     24-bit Clock
--------------------    ------------
1,700,000,000          →  11,046,144  (0xA8A900)
1,700,000,001          →  11,046,145  (0xA8A901)
...
1,716,777,215          →  16,777,215  (0xFFFFFF) ← MAX
1,716,777,216          →  0           (0x000000) ← WRAP!
1,716,777,217          →  1           (0x000001)
```

**Clock Wraparound:**
- 24 bits → max value: 16,777,215 seconds
- **= 194 days**
- After 194 days, clock wraps back to 0

---

### Calculating Idle Time with Wraparound

**Challenge:** How to compute idle time when clock wraps around?

**Solution:**

```go
func EstimateObjectIdleTime(obj *Object) uint32 {
    currentClock := GetLRUClock()
    lastAccessed := obj.LRU

    var idleTime uint32

    if currentClock >= lastAccessed {
        // Normal case
        idleTime = currentClock - lastAccessed
    } else {
        // Wraparound case
        idleTime = (LRU_CLOCK_MAX - lastAccessed) + currentClock
    }

    return idleTime
}
```

**Example (Wraparound):**

```
Scenario:
- Key inserted at: LRU = 16,777,200 (near max)
- Current time:    Clock = 100 (wrapped around)

Calculation:
idleTime = (LRU_CLOCK_MAX - lastAccessed) + currentClock
         = (16,777,215 - 16,777,200) + 100
         = 15 + 100
         = 115 seconds
```

**Visual:**

```
Clock Timeline:
  ...16,777,200 ──────> 16,777,215 (MAX) ──> 0 ──> 100
      ▲                                          ▲
    Inserted                                  Now
      └──────────────────────────────────────┘
                    115 seconds
```

---

#### Edge Case: 194-Day Unaccessed Key

**Problem:**

If a key remains unaccessed for 194+ days:
- Clock completes full cycle
- Idle time calculation becomes ambiguous

**Example:**

```
Key inserted:  Clock = 100
194 days later: Clock = 100 (wrapped exactly once)

Calculated idle time = 100 - 100 = 0 seconds ❌
Actual idle time = 194 days ✓
```

**Redis's Stance:**

> "If a key in a cache remains unaccessed for 194 days, it's effectively dead. This edge case is acceptable."

**Practical Impact:**
- ⚠️ Extremely rare in production caches
- ✅ Trade-off for 8 bytes/object memory savings
- ✅ 1M keys → 8MB saved

---

### Sampling-Based Eviction

Instead of scanning **all keys** to find the true LRU, Redis:

1. **Samples** N random keys (default: 5)
2. Computes idle time for each
3. Adds worst candidates to **eviction pool**
4. Evicts from pool

**Algorithm:**

```go
const SAMPLE_SIZE = 5
const POOL_SIZE = 16

func SampleAndEvict() {
    pool := NewEvictionPool(POOL_SIZE)

    // Sample random keys
    samples := sampleRandomKeys(SAMPLE_SIZE)

    for _, key := range samples {
        obj := store[key]
        idleTime := EstimateObjectIdleTime(obj)

        // Add to pool if worse than current pool members
        pool.Push(key, idleTime)
    }

    // Evict worst candidate (highest idle time)
    victimKey := pool.PopWorst()
    delete(store, victimKey)
}
```

---

### The Eviction Pool

**Structure:**

```go
type EvictionPool struct {
    Items  []PoolItem
    KeySet map[string]struct{}  // Fast membership check
}

type PoolItem struct {
    Key      string
    IdleTime uint32
}

const POOL_SIZE = 16
```

**Why a Pool?**

Instead of evicting immediately after sampling, Redis maintains a pool to:
1. **Accumulate** best candidates across multiple sampling rounds
2. **Always evict** the globally worst candidate seen so far
3. **Avoid** premature eviction of recently sampled keys

---

### Why Array Over Doubly Linked List?

**For small pool (16 elements):**

| Operation | Array (unsorted) | Array (sorted) | Doubly Linked List |
|-----------|-----------------|----------------|-------------------|
| **Insert** | O(1) | O(N) for sort | O(1) |
| **Find Min** | O(N) | O(1) | O(N) without min pointer |
| **Remove** | O(N) | O(N) with shift | O(1) |
| **Memory** | 16 items | 16 items | 16 items + 32 pointers |

**Reality Check:**

For N=16:
- Array operations: ~16 comparisons (cache-friendly, sequential)
- List operations: ~16 pointer dereferences (cache-unfriendly, random access)

**Winner: Array**
- ✅ **Cache locality**: All items in contiguous memory
- ✅ **Fewer allocations**: Single array allocation
- ✅ **No pointer overhead**: 16 × 2 × 8 = 256 bytes saved

---

### Summary

**Approximated LRU Design:**
1. ✅ **24-bit clock**: 3 bytes per object (vs 20 bytes)
2. ✅ **Clock wraparound**: 194-day cycle (acceptable edge case)
3. ✅ **Sampling**: Check 5 random keys (not all keys)
4. ✅ **Eviction pool**: Size 16 array (not doubly linked list)
5. ✅ **Cache-friendly**: Sequential array access

**Key Insights:**
- **Memory efficiency** over theoretical perfection
- **Approximation** is acceptable for cache workloads
- **Simple data structures** (arrays) beat complex ones at small scale
- **Trade-offs** are explicit and well-justified

**Accuracy:**
- 5 samples: ~95% match to exact LRU
- 10 samples: ~98% match to exact LRU
- Good enough for real-world caching

**Next:** Implementing approximated LRU in practice

---

## Chapter 17: Implementing the Approximated LRU Algorithm

### Redis Source Code Structure

The LRU implementation in Redis is located in `evict.c`, containing:
- Clock management functions
- Idle time calculation
- Eviction pool logic
- Main eviction loop

**Key Constants:**

```c
#define EVPOOL_SIZE 16
#define EVPOOL_CACHED_SIZE 5  // Sample size
#define LRU_CLOCK_MAX ((1<<24)-1)  // 0xFFFFFF
#define LRU_CLOCK_RESOLUTION 1000   // 1 second
```

---

### Core Functions

#### 1. Getting the LRU Clock

```c
unsigned int getLRUClock(void) {
    return (mstime() / LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX;
}
```

**Go Implementation:**

```go
const (
    LRU_CLOCK_MAX        = (1 << 24) - 1  // 16,777,215
    LRU_CLOCK_RESOLUTION = 1              // 1 second
)

func GetLRUClock() uint32 {
    nowSeconds := uint32(time.Now().Unix())
    return nowSeconds & LRU_CLOCK_MAX
}
```

**Breakdown:**
```
time.Now().Unix()  → 1,700,000,000 (epoch seconds)
& 0xFFFFFF         → 11,046,144    (last 24 bits)
```

---

#### 2. Estimating Object Idle Time

```c
unsigned long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRUClock();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        return (LRU_CLOCK_MAX - o->lru + lruclock) * LRU_CLOCK_RESOLUTION;
    }
}
```

**Go Implementation:**

```go
func EstimateObjectIdleTime(obj *Object) uint32 {
    currentClock := GetLRUClock()
    lastAccessed := obj.LastAccessedAt

    var idleTime uint32

    if currentClock >= lastAccessed {
        // Normal case: no wraparound
        idleTime = currentClock - lastAccessed
    } else {
        // Wraparound case
        idleTime = (LRU_CLOCK_MAX - lastAccessed) + currentClock
    }

    return idleTime * LRU_CLOCK_RESOLUTION
}
```

---

### Two Dictionary Architecture

Redis uses **two hash tables** internally:

#### 1. Main Store (All Keys)

```go
// stores all key-value pairs
var store = make(map[string]*Object)
```

**Purpose:** Primary key-value storage

---

#### 2. Expires Index (Keys with TTL)

```go
// stores only keys with expiration
var expires = make(map[*Object]int64)
```

**Purpose:**
- Quick iteration over keys with expiration
- Faster sampling for `volatile-*` eviction policies
- Efficient expiry scanning

---

### Object Structure with LRU

**Go Implementation:**

```go
type Object struct {
    Value          interface{}
    Type           ObjectType
    Encoding       ObjectEncoding
    LastAccessedAt uint32   // 24-bit LRU clock (stored as uint32)
    ExpiresAt      int64    // Unix milliseconds
}
```

**Note:** Go doesn't support bitfields, so we use `uint32` but only use lower 24 bits.

---

### Eviction Pool Implementation

#### Pool Structure

```go
const POOL_SIZE = 16

type EvictionPool struct {
    Items  []PoolItem
    KeySet map[string]struct{}
}

type PoolItem struct {
    Key            string
    LastAccessedAt uint32
}
```

**Pool Invariants:**
1. Max size = 16
2. Sorted by idle time (ascending: best → worst)
3. No duplicate keys

---

#### Push to Pool

```go
func (p *EvictionPool) Push(key string, lastAccessedAt uint32) {
    // Skip duplicates
    if _, exists := p.KeySet[key]; exists {
        return
    }

    newItem := PoolItem{
        Key:            key,
        LastAccessedAt: lastAccessedAt,
    }

    if len(p.Items) < POOL_SIZE {
        // Pool not full
        p.Items = append(p.Items, newItem)
        p.KeySet[key] = struct{}{}
        p.sortByIdleTime()
    } else {
        // Pool full: only add if worse than best
        bestIdleTime := GetIdleTime(p.Items[0].LastAccessedAt)
        newIdleTime := GetIdleTime(lastAccessedAt)

        if newIdleTime > bestIdleTime {
            // Remove best (lowest idle time)
            delete(p.KeySet, p.Items[0].Key)

            // Replace with new item
            p.Items[0] = newItem
            p.KeySet[key] = struct{}{}
            p.sortByIdleTime()
        }
    }
}

func (p *EvictionPool) sortByIdleTime() {
    sort.Slice(p.Items, func(i, j int) bool {
        idleI := GetIdleTime(p.Items[i].LastAccessedAt)
        idleJ := GetIdleTime(p.Items[j].LastAccessedAt)
        return idleI < idleJ  // Ascending order
    })
}

func GetIdleTime(lastAccessedAt uint32) uint32 {
    currentClock := GetLRUClock()
    if currentClock >= lastAccessedAt {
        return currentClock - lastAccessedAt
    }
    return (LRU_CLOCK_MAX - lastAccessedAt) + currentClock
}
```

**Note:** Redis uses `memmove()` for efficient array shifting instead of re-sorting. Our Go implementation uses `sort.Slice()` for simplicity.

---

#### Pop from Pool

```go
func (p *EvictionPool) Pop() string {
    if len(p.Items) == 0 {
        return ""
    }

    // Remove worst item (highest idle time, at end)
    worst := p.Items[len(p.Items)-1]
    p.Items = p.Items[:len(p.Items)-1]
    delete(p.KeySet, worst.Key)

    return worst.Key
}
```

---

### Populating the Eviction Pool

```go
func PopulateEvictionPool() {
    const SAMPLE_SIZE = 5

    // Sample random keys
    samples := sampleRandomKeys(SAMPLE_SIZE)

    for _, key := range samples {
        obj, exists := store[key]
        if !exists {
            continue
        }

        // Skip expired keys
        if HasExpired(obj) {
            continue
        }

        // Add to pool
        evictionPool.Push(key, obj.LastAccessedAt)
    }
}

func sampleRandomKeys(count int) []string {
    keys := make([]string, 0, count)

    for key := range store {
        keys = append(keys, key)
        if len(keys) >= count {
            break
        }
    }

    return keys
}
```

---

### Main Eviction Loop

```go
func EvictKeysLRU(numToEvict int) int {
    evicted := 0

    for evicted < numToEvict {
        // Ensure pool has candidates
        if len(evictionPool.Items) == 0 {
            PopulateEvictionPool()
        }

        // No more keys to evict
        if len(evictionPool.Items) == 0 {
            break
        }

        // Pop worst candidate
        victimKey := evictionPool.Pop()

        // Delete from store
        obj := store[victimKey]
        delete(store, victimKey)

        // Delete from expires index
        delete(expires, obj)

        evicted++

        // Update stats
        updateKeyspaceStats(0, "keys", -1)
        if obj.ExpiresAt > 0 {
            updateKeyspaceStats(0, "expires", -1)
        }
    }

    return evicted
}
```

---

### Integration with Store Operations

#### On Key Access (GET)

```go
func Get(key string) (*Object, bool) {
    obj, exists := store[key]
    if !exists {
        return nil, false
    }

    // Check expiration
    if HasExpired(obj) {
        Delete(key)
        return nil, false
    }

    // Update LRU clock
    obj.LastAccessedAt = GetLRUClock()

    return obj, true
}
```

#### On Key Insert/Update (SET)

```go
func Put(key string, obj *Object) {
    // Update LRU clock
    obj.LastAccessedAt = GetLRUClock()

    // Store object
    store[key] = obj

    // Add to expires index if TTL set
    if obj.ExpiresAt > 0 {
        expires[obj] = obj.ExpiresAt
    }

    // Update stats
    updateKeyspaceStats(0, "keys", 1)
    if obj.ExpiresAt > 0 {
        updateKeyspaceStats(0, "expires", 1)
    }

    // Trigger eviction if needed
    if len(store) > config.MaxKeys {
        EvictKeysLRU(int(float64(len(store)) * 0.1))  // Evict 10%
    }
}
```

#### On Key Delete

```go
func Delete(key string) bool {
    obj, exists := store[key]
    if !exists {
        return false
    }

    // Remove from both dictionaries
    delete(store, key)
    delete(expires, obj)

    // Update stats
    updateKeyspaceStats(0, "keys", -1)
    if obj.ExpiresAt > 0 {
        updateKeyspaceStats(0, "expires", -1)
    }

    return true
}
```

---

### Testing LRU Eviction

**Test Scenario:**

```go
func TestLRU() {
    config.MaxKeys = 100

    // Insert 100 keys
    for i := 0; i < 100; i++ {
        key := fmt.Sprintf("key_%d", i)
        Put(key, &Object{Value: fmt.Sprintf("value_%d", i)})
        time.Sleep(1 * time.Millisecond)
    }

    fmt.Println("Keys:", len(store))  // 100

    // Access first 50 keys (make them recently used)
    for i := 0; i < 50; i++ {
        key := fmt.Sprintf("key_%d", i)
        Get(key)
    }

    // Insert 20 more keys (trigger eviction)
    for i := 100; i < 120; i++ {
        key := fmt.Sprintf("key_%d", i)
        Put(key, &Object{Value: fmt.Sprintf("value_%d", i)})
    }

    fmt.Println("Keys after eviction:", len(store))  // ~100

    // Expected: key_50 to key_99 evicted (least recently used)
    // key_0 to key_49 retained (recently accessed)
    // key_100 to key_119 retained (newly inserted)
}
```

---

### Performance Characteristics

**Time Complexity:**

| Operation | Complexity |
|-----------|-----------|
| Get/Set (LRU update) | O(1) |
| Sample keys | O(SAMPLE_SIZE) = O(5) = O(1) |
| Pool push | O(POOL_SIZE log POOL_SIZE) = O(16 log 16) ≈ O(1) |
| Pool pop | O(1) |
| Evict one key | O(SAMPLE_SIZE + POOL_SIZE) ≈ O(1) |

**Space Complexity:**
- Per object: 4 bytes (`LastAccessedAt` as uint32, using 24 bits)
- Eviction pool: 16 × (8 bytes key pointer + 4 bytes LRU) ≈ 192 bytes
- Expires map: 8 bytes per key with TTL

---

### Summary

**Implementation Highlights:**
1. ✅ **24-bit clock**: `GetLRUClock()` masks to 0xFFFFFF
2. ✅ **Wraparound handling**: Check `current >= last` for idle time
3. ✅ **Two dictionaries**: `store` (all keys) + `expires` (TTL keys)
4. ✅ **Eviction pool**: Fixed array of 16 items
5. ✅ **Sampling**: Check 5 random keys per round
6. ✅ **LRU update**: Set `LastAccessedAt` on every access

**Go vs Redis Differences:**
- **Go**: No bitfields → use uint32 for 24-bit value
- **Go**: `sort.Slice()` for pool → Redis uses `memmove()`
- **Go**: Simple map iteration → Redis uses `dictGetSomeKeys()`
- **Go**: Key count limit → Redis uses memory limit

**Production Enhancements (Redis):**
- Memory-based eviction (not key count)
- ZMalloc wrapper for memory tracking
- Async eviction in background
- Multiple eviction policies (allkeys-lru, volatile-lru, etc.)

**Next:** Custom memory allocators and persistence strategies

---
## Chapter 18: Memory Capping with zmalloc

### What is zmalloc?

**zmalloc** is Redis's custom memory allocator wrapper that tracks total memory usage across the entire application. Unlike standard `malloc`, which only allocates memory, zmalloc provides **memory accounting** — knowing exactly how much memory Redis is consuming at any moment.

**Key Purpose:**
- Track cumulative memory usage
- Enable memory-based eviction policies
- Provide foundation for `maxmemory` enforcement

---

### The Problem: Memory-Based Eviction

**Scenario:**

```
Goal: Evict keys when memory exceeds 1GB

Challenge:
- How does Redis know when it's using 1GB?
- Standard malloc() doesn't track total usage
- System calls are expensive
```

**Naive Approach (Too Slow):**

```c
// Check OS memory usage
struct rusage usage;
getrusage(RUSAGE_SELF, &usage);
long memory = usage.ru_maxrss;  // Resident set size

// Problem: System call on every allocation
// Cost: ~500ns per call
// Redis allocates millions of times per second
```

---

### zmalloc Design

#### Core Concept

**Wrap `malloc` to maintain a running total:**

```
┌──────────────────────────────────────────┐
│   Application (Redis)                    │
│                                          │
│   zmalloc(size) ───┐                    │
│                    │                     │
└────────────────────┼─────────────────────┘
                     │
                     ↓
       ┌─────────────────────────────┐
       │   zmalloc.c                 │
       │                             │
       │   used_memory += size  ←────┤ Track
       │   ptr = malloc(size)        │
       │   return ptr                │
       └─────────────────────────────┘
                     │
                     ↓
              ┌──────────────┐
              │  OS malloc   │
              └──────────────┘
```

**Benefits:**
- ✅ O(1) memory usage lookup: `zmalloc_used_memory()`
- ✅ No expensive system calls
- ✅ Exact tracking (byte-level precision)

---

### Implementation

#### Global Counter

```c
// Global atomic variable
static size_t used_memory = 0;
```

**Why atomic?**
- Redis is single-threaded for command execution
- But background threads (AOF rewrite, RDB save) also allocate
- Atomic operations prevent race conditions

---

#### Core Functions

**1. zmalloc() - Allocation with Tracking**

```c
void *zmalloc(size_t size) {
    void *ptr = ztrymalloc_usable(size);
    if (!ptr) {
        zmalloc_default_oom();  // Out of memory handler
    }
    return ptr;
}

void *ztrymalloc_usable(size_t size) {
    void *ptr;

    // Allocate extra space for size prefix
    size_t usable_size = size + PREFIX_SIZE;

    ptr = malloc(usable_size);
    if (!ptr) return NULL;

    // Store allocation size in prefix
    *((size_t*)ptr) = size;

    // Update global counter (atomically)
    update_zmalloc_stat_alloc(size);

    // Return pointer after prefix
    return (char*)ptr + PREFIX_SIZE;
}
```

**Memory Layout:**

```
Allocated Block:
┌──────────────┬───────────────────────────┐
│ PREFIX_SIZE  │  User Data (size bytes)   │
│ (size info)  │                           │
└──────────────┴───────────────────────────┘
▲              ▲
│              └─ Returned pointer
└─ Actual malloc'd address
```

**Why store size?**
- Needed for `free` to know how much to subtract from `used_memory`

---

**2. update_zmalloc_stat_alloc() - Increment Counter**

```c
#define update_zmalloc_stat_alloc(__n) do {         \
    size_t _n = (__n);                              \
    atomicIncr(used_memory, _n);                    \
} while(0)
```

**Atomic Increment:**

```c
// Using GCC atomic builtins
#define atomicIncr(var, count) __sync_add_and_fetch(&var, (count))
```

---

**3. zfree() - Deallocation with Tracking**

```c
void zfree(void *ptr) {
    if (ptr == NULL) return;

    // Get actual allocated pointer (before prefix)
    void *realptr = (char*)ptr - PREFIX_SIZE;

    // Retrieve size from prefix
    size_t size = *((size_t*)realptr);

    // Update counter
    update_zmalloc_stat_free(size);

    // Free memory
    free(realptr);
}
```

**4. update_zmalloc_stat_free() - Decrement Counter**

```c
#define update_zmalloc_stat_free(__n) do {          \
    size_t _n = (__n);                              \
    atomicDecr(used_memory, _n);                    \
} while(0)

#define atomicDecr(var, count) __sync_sub_and_fetch(&var, (count))
```

---

**5. zmalloc_used_memory() - Query Usage**

```c
size_t zmalloc_used_memory(void) {
    size_t um;
    atomicGet(used_memory, um);
    return um;
}
```

**Cost: O(1)** — simple atomic read

---

### Integration with Eviction

#### Memory Limit Check

**In evict.c:**

```c
int overMemoryAfterAllocation(void) {
    // No limit configured
    if (server.maxmemory == 0) return 0;

    // Check if over limit
    size_t mem_used = zmalloc_used_memory();
    if (mem_used > server.maxmemory) {
        return 1;  // Over limit
    }

    return 0;
}
```

#### Eviction Loop

**Triggered on every write command:**

```c
int freeMemoryIfNeeded(void) {
    size_t mem_used = zmalloc_used_memory();
    size_t mem_tofree = mem_used - server.maxmemory;

    while (mem_tofree > 0) {
        // Sample keys and populate eviction pool
        evictionPoolPopulate(/* ... */);

        // Evict worst candidate
        sds bestkey = evictionPoolPopBest();
        if (bestkey == NULL) break;

        // Delete key
        dbDelete(db, bestkey);

        // Recalculate freed memory
        size_t new_mem = zmalloc_used_memory();
        mem_tofree = new_mem - server.maxmemory;
    }

    return C_OK;
}
```

**Flow:**

```
1. Write command arrives (SET key value)
2. Before executing:
   - Check: zmalloc_used_memory() > maxmemory?
   - If yes: freeMemoryIfNeeded()
3. Eviction loop:
   - Evict keys until: zmalloc_used_memory() <= maxmemory
4. Execute SET command
```

---

### zmalloc Everywhere

**zmalloc is used for ALL allocations:**

```c
// Keys and values
robj *createStringObject(char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr, len));
}

sds sdsnewlen(const void *init, size_t initlen) {
    void *sh = zmalloc(/* size */);  // ← zmalloc
    // ...
}

// Data structures
dictEntry *dictAddRaw(dict *d, void *key) {
    dictEntry *entry = zmalloc(sizeof(*entry));  // ← zmalloc
    // ...
}

// Eviction pool
struct evictionPoolEntry *EvictionPoolLRU;
EvictionPoolLRU = zmalloc(sizeof(struct evictionPoolEntry) * EVPOOL_SIZE);
```

**Result:** `zmalloc_used_memory()` reflects **total** Redis memory usage.

---

### Out of Memory Handling

**When malloc fails:**

```c
void zmalloc_default_oom(void) {
    fprintf(stderr, "zmalloc: Out of memory trying to allocate %zu bytes\n", size);
    fflush(stderr);
    abort();  // Crash Redis
}
```

**Why abort?**
- Redis cannot continue without memory
- Better to crash than corrupt data
- Restart triggers persistence recovery (RDB/AOF)

**Production Handling:**
- Monitor memory usage with `INFO memory`
- Set appropriate `maxmemory` (e.g., 80% of available RAM)
- Configure eviction policy

---

### Go Implementation (Simplified)

Go doesn't allow malloc wrapping easily, so we track manually:

```go
var (
    usedMemory     int64
    maxMemory      int64 = 1024 * 1024 * 1024  // 1GB
)

func allocateObject(value string) *Object {
    obj := &Object{Value: value}

    // Estimate size
    size := int64(unsafe.Sizeof(*obj)) + int64(len(value))

    // Track memory
    atomic.AddInt64(&usedMemory, size)

    return obj
}

func freeObject(obj *Object) {
    // Estimate size
    size := int64(unsafe.Sizeof(*obj)) + int64(len(obj.Value.(string)))

    // Track memory
    atomic.AddInt64(&usedMemory, -size)
}

func getUsedMemory() int64 {
    return atomic.LoadInt64(&usedMemory)
}

func isMemoryFull() bool {
    return getUsedMemory() > maxMemory
}
```

**Limitations:**
- Manual tracking (error-prone)
- Doesn't account for Go runtime overhead
- Approximation, not exact

**Alternative:** Use cgroups or OS limits

---

### Design Principles

**1. Observation, Not Control**

zmalloc **tracks** memory, doesn't **prevent** allocations:

```c
void *zmalloc(size_t size) {
    void *ptr = malloc(size);  // Always tries to allocate
    if (ptr) {
        used_memory += size;   // Track after allocation
    }
    return ptr;
}
```

**Policy enforcement** happens at higher level (evict.c).

---

**2. Separation of Concerns**

```
┌─────────────────────────────────────────┐
│  Application Logic (evict.c)           │
│  - Check: used_memory > maxmemory?     │
│  - Trigger eviction                     │
└────────────────┬────────────────────────┘
                 │
                 ↓ Query
      ┌──────────────────────┐
      │  zmalloc.c           │
      │  - Track allocations │
      │  - Provide usage     │
      └──────────┬───────────┘
                 │
                 ↓ Allocate
         ┌──────────────┐
         │  OS malloc   │
         └──────────────┘
```

**Benefits:**
- zmalloc: Simple, focused
- evict.c: Flexible policies
- Easy to test and maintain

---

### Performance Impact

**Cost per allocation:**

| Operation | Time |
|-----------|------|
| Standard malloc | ~100ns |
| zmalloc prefix write | ~5ns |
| Atomic increment | ~10ns |
| **Total** | **~115ns** |

**Overhead: ~15%**

**Trade-off:**
- ⚠️ 15% slower allocations
- ✅ Zero-cost memory usage queries
- ✅ Enables memory-based eviction

**For Redis:**
- Allocations: Thousands/sec
- Memory checks: Millions/sec
- **Net win:** Amortized cost justified

---

### Summary

**What zmalloc Provides:**
1. ✅ **Exact memory tracking**: Byte-level precision
2. ✅ **O(1) queries**: `zmalloc_used_memory()` is instant
3. ✅ **Atomic operations**: Thread-safe for background tasks
4. ✅ **Universal usage**: All Redis allocations tracked

**Implementation Details:**
1. ✅ Global `used_memory` counter
2. ✅ Prefix stores allocation size
3. ✅ `zmalloc()` increments, `zfree()` decrements
4. ✅ Atomic operations for concurrency

**Integration with Eviction:**
1. ✅ `overMemoryAfterAllocation()` checks limit
2. ✅ `freeMemoryIfNeeded()` evicts until under limit
3. ✅ Runs on every write command

**Design Philosophy:**
- **Observation** over control
- **Separation** of tracking and policy
- **Simplicity** for reliability

**Next:** Overriding malloc with jemalloc/tcmalloc

---

## Chapter 19: Overriding malloc for Better Performance

### The Problem with Standard malloc

**Standard malloc (glibc)** is a general-purpose allocator designed for diverse workloads. For specialized applications like Redis, it has limitations:

#### 1. Memory Fragmentation

**Scenario:**

```
Time    Action              Memory Layout
----    ------              -------------
t=0     Allocate A (1KB)    [A____________________]
t=1     Allocate B (2KB)    [A][B_________________]
t=2     Allocate C (1KB)    [A][B][C______________]
t=3     Free B              [A][  ][C______________]
t=4     Allocate D (3KB)    [A][  ][C____________] ← Can't fit!
                             └─2KB gap (wasted)
```

**Problem:**
- Free memory exists (2KB gap)
- But too fragmented to satisfy allocation (3KB needed)
- **External fragmentation**: Total free memory sufficient, but not contiguous

---

**Internal Fragmentation:**

```
Request: 1000 bytes
OS page size: 4096 bytes (4KB)

malloc allocates: 4096 bytes
Wasted: 3096 bytes (75% waste)
```

---

#### 2. Poor Cache Locality

**Sequential allocations may not be contiguous:**

```
malloc(100);  // Address: 0x1000
malloc(100);  // Address: 0x5000  ← 16KB gap!
malloc(100);  // Address: 0x2000
```

**Impact on Redis:**
- Iterating hash table: Random memory access
- CPU cache misses: ~100× slower than cache hits
- Throughput degradation

---

#### 3. Concurrency Bottlenecks

**Standard malloc uses global locks:**

```c
void *malloc(size_t size) {
    pthread_mutex_lock(&global_lock);  // ← Bottleneck
    void *ptr = allocate(size);
    pthread_mutex_unlock(&global_lock);
    return ptr;
}
```

**Problem:**
- Multi-threaded apps (Redis background tasks) serialize on lock
- Contention reduces parallelism

---

### Alternative Allocators

Redis supports multiple allocators through compile-time configuration:

```c
// zmalloc.h
#ifdef USE_TCMALLOC
#define malloc(size) tc_malloc(size)
#define free(ptr) tc_free(ptr)
#elif defined(USE_JEMALLOC)
#define malloc(size) je_malloc(size)
#define free(ptr) je_free(ptr)
#else
// Use system malloc
#endif
```

---

### jemalloc (Facebook)

**jemalloc** is a memory allocator originally developed for FreeBSD, later adopted by Facebook for its high-performance applications.

#### Key Features

**1. Size Classes**

jemalloc groups allocations into **size classes** to reduce fragmentation:

```
Size Classes:
8, 16, 32, 48, 64, 80, 96, 112, 128, ...
256, 512, 1024, 2048, 4096, ...
```

**Example:**

```
Request: 100 bytes
Size class: 128 bytes (next power-of-2-ish bucket)
Internal waste: 28 bytes (22%)
```

**vs Standard malloc:**

```
Request: 100 bytes
Page allocation: 4096 bytes
Internal waste: 3996 bytes (97%)
```

**Benefit:** Predictable waste, better than page-level allocation

---

**2. Arena-Based Allocation**

jemalloc uses **multiple arenas** (memory regions) for parallel allocation:

```
┌─────────────────────────────────────────┐
│         jemalloc                        │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐│
│  │ Arena 0 │  │ Arena 1 │  │ Arena 2 ││
│  └─────────┘  └─────────┘  └─────────┘│
│       ▲            ▲            ▲       │
└───────┼────────────┼────────────┼───────┘
        │            │            │
   Thread 1     Thread 2     Thread 3
```

**Benefit:**
- Each thread allocates from its own arena
- No lock contention
- Parallelism preserved

---

**3. Transparent Huge Pages (THP) Support**

Normal page size: 4KB
Huge page size: 2MB or 1GB

**Benefit:**
- Fewer page table entries
- Reduced TLB (Translation Lookaside Buffer) misses
- Faster address translation

---

**4. Detailed Statistics**

```bash
$ redis-cli INFO memory
# Memory
used_memory:5242880
used_memory_human:5.00M
used_memory_rss:6291456
mem_fragmentation_ratio:1.20
mem_allocator:jemalloc-5.2.1
```

jemalloc provides:
- Allocation size distribution
- Fragmentation metrics
- Per-thread statistics

---

#### Configuring jemalloc in Redis

**Compile Redis with jemalloc:**

```bash
# Redis detects jemalloc automatically if installed
sudo apt-get install libjemalloc-dev
cd redis
make MALLOC=jemalloc
```

**Verify:**

```bash
$ redis-server --version
Redis server v=7.0.0 malloc=jemalloc-5.2.1
```

**Runtime Configuration:**

```bash
# Set jemalloc background thread (improves fragmentation)
redis-cli CONFIG SET jemalloc-bg-thread yes
```

---

### tcmalloc (Google)

**tcmalloc** (Thread-Caching Malloc) is Google's high-performance allocator used in Chrome, Google Search, and other services.

#### Key Features

**1. Thread-Local Caches**

Each thread has a cache of free objects:

```
┌──────────────────────────────────────┐
│  tcmalloc                            │
│                                      │
│  Thread 1 Cache:                    │
│  ┌──────────────┐                   │
│  │ 8B:  [●●●●]  │                   │
│  │ 16B: [●●●]   │                   │
│  │ 32B: [●●●●●] │                   │
│  └──────────────┘                   │
│                                      │
│  Thread 2 Cache:                    │
│  ┌──────────────┐                   │
│  │ 8B:  [●●]    │                   │
│  │ 16B: [●●●●]  │                   │
│  └──────────────┘                   │
│         ▲                            │
└─────────┼────────────────────────────┘
          │
     Central Heap
     (lock required)
```

**Allocation Path:**

```
1. Check thread cache (fast, no lock)
2. If cache empty: fetch from central heap (lock)
3. Populate thread cache
4. Return object
```

**Benefit:** Most allocations avoid locks

---

**2. Size Classes (Similar to jemalloc)**

```
Small objects (≤256KB): 88 size classes
Large objects (>256KB): Page-level allocation
```

---

**3. Span Management**

tcmalloc groups pages into **spans**:

```
Span: [Page][Page][Page][Page]
       ├─ 64B objects ──┤
       [●][●][●][●][●][●][●][●]
```

**Benefit:** Efficient bulk allocation/deallocation

---

#### Configuring tcmalloc in Redis

**Compile Redis with tcmalloc:**

```bash
sudo apt-get install libgoogle-perftools-dev
cd redis
make MALLOC=tcmalloc
```

**Verify:**

```bash
$ redis-server --version
Redis server v=7.0.0 malloc=tcmalloc-2.9.1
```

---

### Performance Comparison

**Benchmark: 1M allocations/deallocations**

| Allocator | Time (ms) | Fragmentation | Peak RSS |
|-----------|----------|---------------|----------|
| **glibc malloc** | 2500 | 35% | 150 MB |
| **jemalloc** | 1200 | 12% | 110 MB |
| **tcmalloc** | 1000 | 10% | 105 MB |

**Redis Throughput (SET commands/sec):**

| Allocator | Throughput | Notes |
|-----------|-----------|-------|
| **glibc** | 85,000 | Baseline |
| **jemalloc** | 110,000 | +29% |
| **tcmalloc** | 115,000 | +35% |

**Winner depends on workload:**
- **jemalloc**: Better for long-running servers, great statistics
- **tcmalloc**: Faster for high-concurrency, simpler profiling

---

### Redis Default: jemalloc

**Why jemalloc?**

1. ✅ **Excellent fragmentation control**
2. ✅ **Detailed memory statistics** (critical for debugging)
3. ✅ **Arena-based parallelism**
4. ✅ **Transparent huge page support**
5. ✅ **Active development** and community support

**Redis Configuration:**

```bash
# redis.conf
# (jemalloc is default if available at compile time)

# Enable background thread for defragmentation
jemalloc-bg-thread yes
```

---

### Abstraction Layer: zmalloc.h

**Redis's abstraction allows easy switching:**

```c
// zmalloc.h
#if defined(USE_TCMALLOC)
#define zmalloc_size(p) tc_malloc_size(p)
#elif defined(USE_JEMALLOC)
#define zmalloc_size(p) je_malloc_usable_size(p)
#else
#define zmalloc_size(p) malloc_usable_size(p)
#endif
```

**Benefit:**
- Switch allocators without code changes
- Compile-time decision
- Benchmark to choose best

---

### Monitoring Memory Health

**Check fragmentation ratio:**

```bash
redis-cli INFO memory | grep fragmentation
mem_fragmentation_ratio:1.20
```

**Interpretation:**

| Ratio | Meaning | Action |
|-------|---------|--------|
| < 1.0 | Redis using swap | ⚠️ Increase RAM or enable eviction |
| 1.0-1.5 | Healthy | ✅ No action |
| 1.5-2.0 | Moderate fragmentation | ⚠️ Monitor, consider restart |
| > 2.0 | High fragmentation | ❌ Restart Redis or defragment |

**Active defragmentation (Redis 4.0+):**

```bash
redis-cli CONFIG SET activedefrag yes
```

---

### Summary

**Standard malloc Limitations:**
1. ❌ High fragmentation (internal + external)
2. ❌ Poor cache locality
3. ❌ Global lock contention

**Alternative Allocators:**

| Feature | jemalloc | tcmalloc | glibc |
|---------|----------|----------|-------|
| **Fragmentation** | Low | Very Low | High |
| **Concurrency** | Arena-based | Thread cache | Global lock |
| **Statistics** | Excellent | Good | Basic |
| **Redis Default** | ✅ Yes | ❌ No | ❌ No |

**Redis Integration:**
1. ✅ Compile-time selection (`make MALLOC=jemalloc`)
2. ✅ Abstraction layer (`zmalloc.h`)
3. ✅ Runtime monitoring (`INFO memory`)

**Key Takeaway:**
- For high-performance, memory-intensive applications
- Custom allocators (jemalloc/tcmalloc) are **essential**
- 20-30% throughput improvement
- 50-70% reduction in fragmentation

**Next:** Graceful shutdown and signal handling

---

## Chapter 20: Implementing Graceful Shutdown

### What is Graceful Shutdown?

**Graceful shutdown** ensures a process terminates cleanly by:
1. Completing ongoing operations
2. Saving state to disk
3. Releasing resources (sockets, file handles)
4. Avoiding data corruption

**Contrast with Abrupt Shutdown:**

| Type | Trigger | Behavior | Data Safety |
|------|---------|----------|-------------|
| **Graceful** | SIGTERM, SIGINT (Ctrl+C) | Complete work, save state, exit | ✅ Safe |
| **Abrupt** | SIGKILL, power loss | Immediate termination | ❌ Risk of corruption |

---

### Operating System Signals

**Signals** are notifications sent by the OS kernel to processes:

```
┌──────────────┐         Signal          ┌──────────────┐
│   Kernel     │ ───────────────────────>│   Process    │
│              │      (SIGTERM, SIGINT)  │              │
└──────────────┘                         └──────────────┘
```

**Common Signals:**

| Signal | Number | Source | Default Behavior |
|--------|--------|--------|------------------|
| `SIGINT` | 2 | Ctrl+C | Terminate |
| `SIGTERM` | 15 | `kill <pid>` | Terminate |
| `SIGKILL` | 9 | `kill -9 <pid>` | Force kill (cannot be caught) |
| `SIGHUP` | 1 | Terminal closed | Terminate |
| `SIGPIPE` | 13 | Write to closed pipe | Terminate |
| `SIGSEGV` | 11 | Segmentation fault | Core dump + terminate |
| `SIGBUS` | 7 | Bus error | Core dump + terminate |

---

### Redis Graceful Shutdown

**Example:**

```bash
$ redis-server &
[1] 12345

$ redis-cli SET mykey "value"
OK

$ kill -SIGTERM 12345
# or Ctrl+C

# Redis output:
# User requested shutdown...
# Saving the final RDB snapshot before exiting.
# DB saved on disk
# Redis is now ready to exit, bye bye...
```

**Steps:**
1. Receive SIGTERM/SIGINT
2. Set shutdown flag
3. Complete in-flight commands
4. Save RDB snapshot (if persistence enabled)
5. Flush AOF buffer
6. Close client connections
7. Exit cleanly

---

### Signal Handling in Redis

#### Setup Signal Handlers

**In `server.c` (initialization):**

```c
void setupSignalHandlers(void) {
    struct sigaction act;

    // SIGTERM handler
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    act.sa_handler = sigtermHandler;
    sigaction(SIGTERM, &act, NULL);

    // SIGINT handler (Ctrl+C)
    act.sa_handler = sigtermHandler;
    sigaction(SIGINT, &act, NULL);

    // SIGHUP - ignore (don't terminate on terminal close)
    signal(SIGHUP, SIG_IGN);

    // SIGPIPE - ignore (don't crash on broken pipe)
    signal(SIGPIPE, SIG_IGN);
}
```

---

#### Signal Handler Implementation

```c
static void sigtermHandler(int sig) {
    char *msg;

    switch (sig) {
    case SIGTERM:
        msg = "Received SIGTERM, scheduling shutdown...";
        break;
    case SIGINT:
        msg = "Received SIGINT, scheduling shutdown...";
        break;
    default:
        msg = "Received shutdown signal, scheduling shutdown...";
    }

    // Log message
    serverLogFromHandler(LL_WARNING, msg);

    // Set shutdown flag (read by main loop)
    server.shutdown_asap = 1;
}
```

**Key Point:** Handler sets flag, doesn't perform shutdown directly
- Signal handlers run in async context
- Cannot safely call most functions
- Main loop checks flag and triggers shutdown

---

#### Main Loop Integration

```c
int main(int argc, char **argv) {
    // ... initialization ...

    aeSetBeforeSleepProc(server.el, beforeSleep);

    // Main event loop
    aeMain(server.el);

    // Cleanup after event loop exits
    aeDeleteEventLoop(server.el);
    return 0;
}

void beforeSleep(struct aeEventLoop *eventLoop) {
    // Check shutdown flag on every loop iteration
    if (server.shutdown_asap) {
        if (prepareForShutdown(SHUTDOWN_NOFLAGS) == C_OK) {
            exit(0);
        }
        // If preparation failed, keep running
        server.shutdown_asap = 0;
    }

    // ... other pre-sleep tasks ...
}
```

---

### Graceful Shutdown Implementation

#### prepareForShutdown() Function

```c
int prepareForShutdown(int flags) {
    // 1. Log shutdown
    serverLog(LL_WARNING, "User requested shutdown...");

    // 2. Kill background children (RDB/AOF rewrite)
    if (server.aof_child_pid != -1) {
        serverLog(LL_WARNING, "There is a child rewriting the AOF. Killing it!");
        kill(server.aof_child_pid, SIGKILL);
    }
    if (server.rdb_child_pid != -1) {
        serverLog(LL_WARNING, "There is a child saving an .rdb. Killing it!");
        kill(server.rdb_child_pid, SIGKILL);
    }

    // 3. Flush AOF buffer
    if (server.aof_state != AOF_OFF) {
        flushAppendOnlyFile(1);  // Force flush
        aof_fsync(server.aof_fd);
    }

    // 4. Save final RDB snapshot (if persistence enabled)
    if (server.saveparamslen > 0 && !(flags & SHUTDOWN_NOSAVE)) {
        serverLog(LL_NOTICE, "Saving the final RDB snapshot before exiting.");
        if (rdbSave(server.rdb_filename) == C_OK) {
            serverLog(LL_NOTICE, "DB saved on disk");
        }
    }

    // 5. Remove PID file
    if (server.daemonize || server.pidfile) {
        unlink(server.pidfile);
    }

    // 6. Close listening sockets
    closeListeningSockets(1);

    serverLog(LL_WARNING, "Redis is now ready to exit, bye bye...");
    return C_OK;
}
```

---

### Wait for Command Completion

**Challenge:** Don't interrupt commands mid-execution

**Solution: Engine Status State Machine**

```go
type EngineStatus int

const (
    ENGINE_WAITING      EngineStatus = 0  // Idle, waiting for events
    ENGINE_BUSY         EngineStatus = 1  // Executing command
    ENGINE_SHUTTING_DOWN EngineStatus = 2  // Shutdown initiated
)

var engineStatus int32 = ENGINE_WAITING  // Atomic variable
```

---

#### Event Loop with Status Tracking

```go
func runAsyncTCPServer() {
    for {
        // Wait for events
        events, err := epoll.Wait(-1)
        if err != nil {
            if engineStatus == ENGINE_SHUTTING_DOWN {
                break  // Exit loop
            }
            continue
        }

        for _, event := range events {
            // Attempt to transition to BUSY
            if !compareAndSwap(&engineStatus, ENGINE_WAITING, ENGINE_BUSY) {
                // Already shutting down, skip this event
                continue
            }

            // Execute command
            processClientCommand(event.Fd)

            // Return to WAITING
            atomic.StoreInt32(&engineStatus, ENGINE_WAITING)
        }
    }

    serverLog("Server shutdown complete")
}
```

---

#### Signal Handler with Wait Logic

```go
func waitForSignal(wg *sync.WaitGroup, sigs chan os.Signal) {
    defer wg.Done()

    // Block until signal received
    sig := <-sigs
    serverLog("Received signal: %v", sig)

    // Set shutdown status
    atomic.StoreInt32(&engineStatus, ENGINE_SHUTTING_DOWN)

    // Wait for current command to finish
    for {
        status := atomic.LoadInt32(&engineStatus)
        if status != ENGINE_BUSY {
            break  // No command in progress
        }
        time.Sleep(10 * time.Millisecond)  // Poll
    }

    // Perform shutdown tasks
    shutdown()

    os.Exit(0)
}
```

---

#### Critical Race Condition Fix

**Problem:**

```
Time    Main Loop                Signal Handler
----    ---------                --------------
t=0     status = WAITING
t=1                              status = SHUTTING_DOWN
t=2     status = BUSY            ← TOO LATE!
```

Main loop might enter BUSY after shutdown initiated.

**Solution: Atomic Compare-and-Swap**

```go
func compareAndSwap(addr *int32, old, new int32) bool {
    return atomic.CompareAndSwapInt32(addr, old, new)
}

// In event loop:
if !compareAndSwap(&engineStatus, ENGINE_WAITING, ENGINE_BUSY) {
    // Transition failed (already shutting down)
    continue
}
```

**Behavior:**

```
Scenario 1: Normal transition
- Current: WAITING
- CAS(WAITING → BUSY): Success
- Execute command

Scenario 2: Shutdown initiated
- Current: SHUTTING_DOWN
- CAS(WAITING → BUSY): Fail (current ≠ WAITING)
- Skip command, check shutdown flag
```

---

### Shutdown Function

```go
func shutdown() {
    serverLog("Starting graceful shutdown...")

    // 1. Rewrite AOF
    if config.AOFEnabled {
        serverLog("Rewriting AOF file...")
        if err := dumpAllAOF(); err != nil {
            serverLog("Error rewriting AOF: %v", err)
        }
    }

    // 2. Close listening socket
    if listener != nil {
        listener.Close()
        serverLog("Listening socket closed")
    }

    // 3. Close client connections
    for _, conn := range clients {
        conn.Close()
    }
    serverLog("All client connections closed")

    // 4. Flush and close AOF file
    if aofFile != nil {
        aofFile.Sync()
        aofFile.Close()
        serverLog("AOF file flushed and closed")
    }

    // 5. Clean up temporary files
    os.Remove("temp.aof")

    serverLog("Shutdown complete, goodbye!")
}
```

---

### Testing Graceful Shutdown

#### Test 1: Basic Shutdown

```bash
$ go run main.go &
[1] 12345
Server listening on :7379

$ redis-cli -p 7379 SET key1 value1
OK

$ kill -SIGTERM 12345
# Server output:
# Received signal: terminated
# Starting graceful shutdown...
# Rewriting AOF file...
# AOF file rewrite complete
# Listening socket closed
# All client connections closed
# Shutdown complete, goodbye!
```

---

#### Test 2: Shutdown During Long Command

**Implement test command:**

```go
func evalSLEEP(args []string) []byte {
    if len(args) != 1 {
        return resp.EncodeError("ERR wrong number of arguments")
    }

    seconds, _ := strconv.Atoi(args[0])
    time.Sleep(time.Duration(seconds) * time.Second)

    return resp.EncodeSimpleString("OK")
}
```

**Test:**

```bash
$ go run main.go &

$ redis-cli -p 7379 SLEEP 10 &
# Sleeping for 10 seconds...

$ kill -SIGTERM <pid>
# Server output:
# Received signal: terminated
# Waiting for command to complete...
# (after 10 seconds)
# Starting graceful shutdown...
# Shutdown complete, goodbye!
```

**Verification:** Shutdown waits for SLEEP to finish ✅

---

### Error Signal Handling

For crashes (SIGSEGV, SIGBUS, etc.), Redis logs diagnostic info:

```c
void setupSignalHandlers(void) {
    // ... SIGTERM/SIGINT handlers ...

    // Crash handlers
    act.sa_flags = SA_NODEFER | SA_RESETHAND | SA_SIGINFO;
    act.sa_sigaction = sigsegvHandler;
    sigaction(SIGSEGV, &act, NULL);
    sigaction(SIGBUS, &act, NULL);
    sigaction(SIGFPE, &act, NULL);
    sigaction(SIGILL, &act, NULL);
}

void sigsegvHandler(int sig, siginfo_t *info, void *secret) {
    serverLog(LL_WARNING, "===== REDIS BUG REPORT START =====");
    serverLog(LL_WARNING, "    Redis %s crashed by signal: %d", REDIS_VERSION, sig);

    // Stack trace
    logStackTrace(secret);

    // Client info
    logCurrentClient();

    // Memory info
    logMemoryInfo();

    serverLog(LL_WARNING, "===== REDIS BUG REPORT END =====");

    // Re-raise signal to trigger core dump
    raise(sig);
}
```

---

### Summary

**Graceful Shutdown Flow:**
1. ✅ Receive signal (SIGTERM/SIGINT)
2. ✅ Set shutdown flag
3. ✅ Wait for in-flight commands
4. ✅ Save persistence data (RDB/AOF)
5. ✅ Close sockets and connections
6. ✅ Clean up resources
7. ✅ Exit

**Implementation Highlights:**
1. ✅ **Signal handlers**: Set flag, don't execute logic directly
2. ✅ **Engine status**: State machine (WAITING/BUSY/SHUTTING_DOWN)
3. ✅ **Atomic CAS**: Prevent race between command start and shutdown
4. ✅ **Wait loop**: Poll until BUSY → WAITING
5. ✅ **Shutdown function**: Persistence, cleanup, exit

**Go Implementation:**

```go
// Main function
func main() {
    var wg sync.WaitGroup

    // Goroutine 1: TCP server
    go runAsyncTCPServer()

    // Goroutine 2: Signal handler
    wg.Add(1)
    sigs := make(chan os.Signal, 1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
    go waitForSignal(&wg, sigs)

    wg.Wait()  // Wait for signal handler to complete
}
```

**Key Principles:**
- **Data safety**: Never corrupt persistence files
- **Client courtesy**: Finish commands before disconnect
- **Resource cleanup**: Release sockets, file handles
- **Crash resilience**: Log diagnostic info for debugging

**Next:** Transactions and MULTI/EXEC implementation

---
