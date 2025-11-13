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

