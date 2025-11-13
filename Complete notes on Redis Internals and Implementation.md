# Complete Notes on Redis Internals and Implementation

> A comprehensive guide to understanding Redis from the ground up - covering architecture, implementation details, data structures, algorithms, and advanced concepts.

---

## Table of Contents

### Parts 1-7: Redis Internals and Implementation (How Redis is Built)

1. [Introduction and Overview](#introduction-and-overview)
2. [Building Redis from Scratch](#building-redis-from-scratch)
3. [Core Commands Implementation](#core-commands-implementation)
4. [Memory Management & Eviction](#memory-management--eviction)
5. [Advanced Features](#advanced-features)
6. [Internal Data Structures](#internal-data-structures)
7. [Advanced Algorithms](#advanced-algorithms)

### Part 8: User Guide and Practical Patterns (How to Use Redis)

8. [Redis Data Structures: User Guide and Practical Patterns](#part-8-redis-data-structures---user-guide-and-practical-patterns)
   - [Chapter 28: Strings - Commands, Use Cases, Examples](#chapter-28-strings---the-foundation-of-redis-data-storage)
   - [Chapter 29: Lists - Commands, Use Cases, Examples](#chapter-29-lists---ordered-collections-for-queues-and-stacks)
   - [Chapter 30: Sets - Commands, Use Cases, Examples](#chapter-30-sets---unique-collections-for-membership-and-relationships)
   - [Chapter 31: Sorted Sets - Commands, Use Cases, Examples](#chapter-31-sorted-sets---ranked-collections-with-scores)
   - [Chapter 32: Hashes - Commands, Use Cases, Examples](#chapter-32-hashes---field-value-pairs-within-keys)
   - [Chapter 33: Bitmaps - Commands, Use Cases, Examples](#chapter-33-bitmaps---efficient-binary-operations)
   - [Chapter 34: HyperLogLogs - Commands, Use Cases, Examples](#chapter-34-hyperloglogs---cardinality-estimation-at-scale)
   - [Chapter 35: Streams - Commands, Use Cases, Examples](#chapter-35-streams---event-logs-and-message-queues)
   - [Chapter 36: Pub/Sub - Messaging Patterns](#chapter-36-pubsub---messaging-patterns)
   - [Chapter 37: Lua Scripting - Programmability](#chapter-37-lua-scripting-and-programmability)
   - [Chapter 38: Geospatial - Location-Based Queries](#chapter-38-geospatial-commands---location-based-queries)
   - [Chapter 39: Performance and Best Practices](#chapter-39-performance-considerations-and-best-practices)
   - [Chapter 40: Probabilistic Data Structures at Scale](#chapter-40-probabilistic-data-structures-at-scale)
   - [Chapter 41: Java Integration with Jedis - Production Patterns](#chapter-41-java-integration-with-jedis---production-patterns)

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
## Chapter 21: Implementing Transactions (MULTI/EXEC)

### What are Redis Transactions?

**Redis transactions** allow grouping multiple commands into a single atomic unit that executes sequentially without interruption.

**Key Characteristics:**
- **Atomicity**: All commands execute together
- **Isolation**: No other client commands interleave
- **No rollback**: Failed commands don't revert transaction
- **Simplicity**: Command queuing, not complex 2PC

**Commands:**
- `MULTI`: Begin transaction
- `EXEC`: Commit and execute all queued commands
- `DISCARD`: Abort transaction, clear queue

---

### Basic Transaction Flow

**Example:**

```bash
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET user:1:name "Alice"
QUEUED
127.0.0.1:6379(TX)> SET user:1:age "30"
QUEUED
127.0.0.1:6379(TX)> INCR user:1:visits
QUEUED
127.0.0.1:6379(TX)> EXEC
1) OK
2) OK
3) (integer) 1
```

**Visual Flow:**

```
┌─────────────────────────────────────────────┐
│ Client                                      │
│                                             │
│ MULTI ──────────────────────────> Server   │
│         <─────────────────────── OK         │
│                                             │
│ SET user:1:name "Alice" ─────────> Server  │
│         <─────────────────────── QUEUED    │
│                                             │
│ SET user:1:age "30" ──────────────> Server │
│         <─────────────────────── QUEUED    │
│                                             │
│ INCR user:1:visits ───────────────> Server │
│         <─────────────────────── QUEUED    │
│                                             │
│ EXEC ──────────────────────────────> Server│
│         (execute all commands)              │
│         <─────────────────────────┐        │
│         Array of 3 responses:     │        │
│         1) OK                     │        │
│         2) OK                     │        │
│         3) (integer) 1            │        │
└─────────────────────────────────────────────┘
```

---

### Transactions vs Pipelining

| Feature | Transactions (MULTI/EXEC) | Pipelining |
|---------|--------------------------|------------|
| **Atomicity** | ✅ Yes (all or nothing visibility) | ❌ No |
| **Isolation** | ✅ Yes (no interleaving) | ❌ No |
| **Rollback** | ❌ No | ❌ No |
| **Performance** | Moderate (queuing overhead) | High (no queuing) |
| **Use Case** | Consistent state updates | Bulk operations |

**Example showing difference:**

```bash
# Transaction: Atomic transfer
MULTI
DECRBY account:A 100
INCRBY account:B 100
EXEC
# Both execute together, or neither (if connection drops)

# Pipeline: Independent commands
SET key1 value1
SET key2 value2
GET key1
# Execute independently, no atomicity guarantee
```

---

### State Management for Transactions

**Challenge:** Clients need to maintain transaction state

**Solution:** Per-client state tracking

#### Client Object Structure

```go
type Client struct {
    FD             int              // File descriptor (socket)
    CommandQueue   []*RedisCommand  // Queued commands during transaction
    IsTransaction  bool             // Transaction mode flag
    Buffer         []byte           // Response buffer
}
```

#### Global Client Registry

```go
var connectedClients = make(map[int]*Client)

func onClientConnect(fd int) {
    client := &Client{
        FD:            fd,
        CommandQueue:  make([]*RedisCommand, 0),
        IsTransaction: false,
        Buffer:        make([]byte, 0, 4096),
    }
    connectedClients[fd] = client
}

func onClientDisconnect(fd int) {
    delete(connectedClients, fd)
    close(fd)
}
```

**Why File Descriptor as Key?**
- Unique per connection
- Already available from epoll events
- No need for session IDs

---

### Implementation

#### 1. MULTI Command

**Purpose:** Enter transaction mode

```go
func evalMULTI(client *Client, args []string) []byte {
    if client.IsTransaction {
        return resp.EncodeError("ERR MULTI calls can not be nested")
    }

    client.IsTransaction = true
    client.CommandQueue = make([]*RedisCommand, 0)

    return resp.EncodeSimpleString("OK")
}
```

**Client State Change:**

```
Before:
┌────────────────────┐
│ Client             │
│ IsTransaction: ❌  │
│ CommandQueue: []   │
└────────────────────┘

After MULTI:
┌────────────────────┐
│ Client             │
│ IsTransaction: ✅  │
│ CommandQueue: []   │
└────────────────────┘
```

---

#### 2. Command Queuing

**For clients in transaction mode, queue commands instead of executing:**

```go
func processCommand(client *Client, cmd *RedisCommand) []byte {
    cmdName := strings.ToUpper(cmd.Args[0])

    // Transaction control commands execute immediately
    switch cmdName {
    case "MULTI":
        return evalMULTI(client, cmd.Args[1:])
    case "EXEC":
        return evalEXEC(client, cmd.Args[1:])
    case "DISCARD":
        return evalDISCARD(client, cmd.Args[1:])
    }

    // If in transaction mode, queue command
    if client.IsTransaction {
        client.CommandQueue = append(client.CommandQueue, cmd)
        return resp.EncodeSimpleString("QUEUED")
    }

    // Normal execution
    return executeCommand(cmd)
}
```

**Queue Growth:**

```
MULTI
┌────────────────────┐
│ CommandQueue: []   │
└────────────────────┘

SET key1 val1
┌────────────────────────────────┐
│ CommandQueue: [SET key1 val1] │
└────────────────────────────────┘

SET key2 val2
┌──────────────────────────────────────────────────┐
│ CommandQueue: [SET key1 val1, SET key2 val2]    │
└──────────────────────────────────────────────────┘

GET key1
┌─────────────────────────────────────────────────────────────────┐
│ CommandQueue: [SET key1 val1, SET key2 val2, GET key1]         │
└─────────────────────────────────────────────────────────────────┘
```

---

#### 3. EXEC Command

**Purpose:** Execute all queued commands atomically

```go
func evalEXEC(client *Client, args []string) []byte {
    if !client.IsTransaction {
        return resp.EncodeError("ERR EXEC without MULTI")
    }

    // Execute all queued commands
    results := make([][]byte, 0, len(client.CommandQueue))

    for _, cmd := range client.CommandQueue {
        result := executeCommand(cmd)
        results = append(results, result)
    }

    // Clear transaction state
    client.IsTransaction = false
    client.CommandQueue = nil

    // Return array of results
    return encodeArray(results)
}

func encodeArray(items [][]byte) []byte {
    var buf bytes.Buffer

    // Array header: *<count>\r\n
    buf.WriteString(fmt.Sprintf("*%d\r\n", len(items)))

    // Each item (already RESP encoded)
    for _, item := range items {
        buf.Write(item)
    }

    return buf.Bytes()
}
```

**Execution Flow:**

```
CommandQueue: [SET key1 val1, SET key2 val2, GET key1]

Execute SET key1 val1 → +OK\r\n
Execute SET key2 val2 → +OK\r\n
Execute GET key1      → $4\r\nval1\r\n

Encode as array:
*3\r\n
+OK\r\n
+OK\r\n
$4\r\nval1\r\n
```

---

#### 4. DISCARD Command

**Purpose:** Abort transaction without executing

```go
func evalDISCARD(client *Client, args []string) []byte {
    if !client.IsTransaction {
        return resp.EncodeError("ERR DISCARD without MULTI")
    }

    // Clear transaction state
    client.IsTransaction = false
    client.CommandQueue = nil

    return resp.EncodeSimpleString("OK")
}
```

**Example:**

```bash
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET key1 value1
QUEUED
127.0.0.1:6379(TX)> SET key2 value2
QUEUED
127.0.0.1:6379(TX)> DISCARD
OK
127.0.0.1:6379> GET key1
(nil)  # Commands were not executed
```

---

### Isolation and Concurrency

#### Single-Threaded Execution

**Redis's single-threaded event loop ensures isolation:**

```
Time    Client A              Client B              Redis
----    --------              --------              -----
t=0     MULTI                                       Set A: IsTransaction=true
t=1     SET key1 val1                               Queue in A
t=2                           MULTI                  Set B: IsTransaction=true
t=3     SET key2 val2                               Queue in A
t=4                           SET key3 val3          Queue in B
t=5     EXEC                                        Execute A's queue
t=6                                                 ├─ SET key1 val1
t=7                                                 ├─ SET key2 val2
t=8                                                 └─ Return array
t=9                           EXEC                  Execute B's queue
t=10                                                ├─ SET key3 val3
t=11                                                └─ Return array
```

**Key Point:** Client B's commands can't execute until Client A's transaction completes.

---

### No Rollback

**Redis transactions do NOT support rollback:**

```bash
127.0.0.1:6379> SET balance 100
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> DECRBY balance 50
QUEUED
127.0.0.1:6379(TX)> INCR invalid_key
QUEUED
127.0.0.1:6379(TX)> EXEC
1) (integer) 50
2) (integer) 1
127.0.0.1:6379> GET balance
"50"  # First command succeeded despite second command error
```

**Why No Rollback?**

Redis philosophy:
1. **Simplicity**: No undo log, no compensation logic
2. **Performance**: No overhead for rollback tracking
3. **Responsibility**: Application should validate before EXEC
4. **Reality**: Errors in well-tested code are rare

---

### Error Handling

#### Syntax Errors (Before EXEC)

```bash
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET key value
QUEUED
127.0.0.1:6379(TX)> SET key  # Missing argument
(error) ERR wrong number of arguments for 'SET' command
127.0.0.1:6379(TX)> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
```

**Implementation:**

```go
type Client struct {
    FD             int
    CommandQueue   []*RedisCommand
    IsTransaction  bool
    HasError       bool  // Track syntax errors
}

func processCommand(client *Client, cmd *RedisCommand) []byte {
    if client.IsTransaction {
        // Validate command syntax
        if err := validateCommand(cmd); err != nil {
            client.HasError = true
            return resp.EncodeError("ERR " + err.Error())
        }

        client.CommandQueue = append(client.CommandQueue, cmd)
        return resp.EncodeSimpleString("QUEUED")
    }
    // ...
}

func evalEXEC(client *Client, args []string) []byte {
    if !client.IsTransaction {
        return resp.EncodeError("ERR EXEC without MULTI")
    }

    if client.HasError {
        client.IsTransaction = false
        client.CommandQueue = nil
        client.HasError = false
        return resp.EncodeError("EXECABORT Transaction discarded because of previous errors")
    }

    // Execute commands...
}
```

---

#### Runtime Errors (During EXEC)

```bash
127.0.0.1:6379> SET mykey "string_value"
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> INCR mykey  # Will fail at runtime
QUEUED
127.0.0.1:6379(TX)> SET anotherkey "value"
QUEUED
127.0.0.1:6379(TX)> EXEC
1) (error) ERR value is not an integer or out of range
2) OK  # This command still executes!
```

**Key Point:** Runtime errors don't abort transaction

---

### WATCH for Optimistic Locking

**Problem:** Transaction values may change between MULTI and EXEC

```bash
# Client A                      # Client B
SET balance 100
MULTI
DECRBY balance 50
                                SET balance 0  # Concurrent modification!
EXEC
# Balance is now -50 (incorrect!)
```

**Solution: WATCH command**

```bash
# Client A
WATCH balance
GET balance  # Read: 100
MULTI
DECRBY balance 50
EXEC  # Aborts if balance changed since WATCH
```

**Implementation:**

```go
type Client struct {
    FD            int
    CommandQueue  []*RedisCommand
    IsTransaction bool
    WatchedKeys   map[string]uint64  // key -> version
}

var keyVersions = make(map[string]uint64)

func evalWATCH(client *Client, args []string) []byte {
    if client.IsTransaction {
        return resp.EncodeError("ERR WATCH inside MULTI is not allowed")
    }

    for _, key := range args {
        version, exists := keyVersions[key]
        if !exists {
            version = 0
        }
        client.WatchedKeys[key] = version
    }

    return resp.EncodeSimpleString("OK")
}

func onKeyModified(key string) {
    keyVersions[key]++
}

func evalEXEC(client *Client, args []string) []byte {
    if !client.IsTransaction {
        return resp.EncodeError("ERR EXEC without MULTI")
    }

    // Check if watched keys changed
    for key, watchedVersion := range client.WatchedKeys {
        currentVersion := keyVersions[key]
        if currentVersion != watchedVersion {
            // Transaction aborted
            client.IsTransaction = false
            client.CommandQueue = nil
            client.WatchedKeys = nil
            return resp.EncodeNullArray()  // (nil)
        }
    }

    // Execute commands...
    results := make([][]byte, 0, len(client.CommandQueue))
    for _, cmd := range client.CommandQueue {
        result := executeCommand(cmd)
        results = append(results, result)

        // Update key versions for modified keys
        for _, key := range cmd.ModifiedKeys {
            onKeyModified(key)
        }
    }

    // Clear state
    client.IsTransaction = false
    client.CommandQueue = nil
    client.WatchedKeys = nil

    return encodeArray(results)
}
```

**WATCH Example:**

```bash
# Optimistic locking pattern
WATCH balance
balance = GET balance
MULTI
SET balance (balance - 50)
result = EXEC

if result is nil:
    # Transaction aborted, retry
else:
    # Success
```

---

### Testing Transactions

**Test 1: Basic Transaction**

```go
func TestBasicTransaction(t *testing.T) {
    conn := connectRedis()

    conn.Send("MULTI")
    conn.Send("SET", "key1", "value1")
    conn.Send("SET", "key2", "value2")
    conn.Send("GET", "key1")

    results, _ := conn.Do("EXEC")

    // results: [OK, OK, "value1"]
    assert.Equal(t, 3, len(results))
    assert.Equal(t, "OK", results[0])
    assert.Equal(t, "value1", results[2])
}
```

---

**Test 2: Discard**

```go
func TestDiscard(t *testing.T) {
    conn := connectRedis()

    conn.Do("SET", "key", "original")

    conn.Send("MULTI")
    conn.Send("SET", "key", "modified")
    conn.Do("DISCARD")

    value, _ := conn.Do("GET", "key")
    assert.Equal(t, "original", value)  // Not modified
}
```

---

**Test 3: Optimistic Locking with WATCH**

```go
func TestWatchSuccess(t *testing.T) {
    conn := connectRedis()
    conn.Do("SET", "counter", "0")

    conn.Do("WATCH", "counter")
    conn.Send("MULTI")
    conn.Send("INCR", "counter")
    results, _ := conn.Do("EXEC")

    assert.NotNil(t, results)  // Transaction succeeded
}

func TestWatchAbort(t *testing.T) {
    conn1 := connectRedis()
    conn2 := connectRedis()

    conn1.Do("SET", "counter", "0")
    conn1.Do("WATCH", "counter")

    // Concurrent modification
    conn2.Do("INCR", "counter")

    conn1.Send("MULTI")
    conn1.Send("INCR", "counter")
    results, _ := conn1.Do("EXEC")

    assert.Nil(t, results)  // Transaction aborted
}
```

---

### Performance Considerations

**Transaction Overhead:**

| Operation | Time (ns) |
|-----------|----------|
| Normal SET | 50,000 |
| MULTI | 5,000 |
| Queue command | 1,000 |
| EXEC (3 commands) | 160,000 |

**Cost Breakdown:**
- Queuing: ~1µs per command
- Execution: Same as normal
- Array encoding: ~5µs

**Total: ~10-20% overhead for transactions**

---

### Use Cases

**1. Atomic Counters**

```bash
MULTI
INCR page:views
INCR user:actions
EXEC
```

---

**2. Conditional Updates**

```bash
WATCH config:version
version = GET config:version
if version == expected_version:
    MULTI
    SET config:value new_value
    INCR config:version
    EXEC
```

---

**3. Consistent Multi-Key Updates**

```bash
MULTI
DECRBY inventory:item123 1
INCRBY cart:user456 1
EXEC
```

---

### Summary

**What Transactions Provide:**
1. ✅ **Atomicity**: All commands execute together
2. ✅ **Isolation**: No interleaving from other clients
3. ✅ **Consistency**: Single-threaded guarantees order
4. ❌ **Durability**: In-memory, not durable (use AOF/RDB)

**Implementation Highlights:**
1. ✅ Per-client state (`IsTransaction`, `CommandQueue`)
2. ✅ Command queuing during MULTI
3. ✅ Array response encoding for EXEC
4. ✅ WATCH for optimistic locking
5. ✅ Syntax error detection before EXEC

**Key Differences from Traditional Databases:**
- **No BEGIN/COMMIT**: Use MULTI/EXEC
- **No ROLLBACK**: Failed commands don't revert
- **No 2PC**: Simple command queuing
- **No ACID guarantees**: Single-node, in-memory

**Redis Transaction Philosophy:**
- **Simplicity** over complexity
- **Performance** over features
- **Pragmatism** over theory

**Next:** Internal data structures (ziplist, quicklist, intset)

---
# Part 6: Internal Data Structures

## Chapter 22: List Internals - Ziplist and Quicklist

### Redis List Implementation

Redis lists support operations like `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, and `LRANGE`. The internal implementation has evolved over time for optimal performance and memory efficiency.

**Current Encoding (Redis 7.0+):**
```bash
127.0.0.1:6379> LPUSH mylist "element1"
(integer) 1
127.0.0.1:6379> DEBUG OBJECT mylist
Value at:0x7f8a... encoding:quicklist serializedlength:20 lru:12345678
```

**Encoding:** `quicklist` (hybrid structure)

---

### Why Not Standard Doubly Linked List?

**Traditional Doubly Linked List:**

```c
struct ListNode {
    struct ListNode *prev;  // 8 bytes (64-bit)
    struct ListNode *next;  // 8 bytes
    void *value;            // 8 bytes
};
// Total: 24 bytes per node
```

**Problems:**

#### 1. Memory Overhead

```
For a 5-character string ("hello"):
- String data: 5 bytes
- Node overhead: 24 bytes
- redisObject: 16 bytes
- Total: 45 bytes (89% overhead!)
```

**For 1 million small strings:**
```
Data: 5MB
Overhead: 40MB
Total: 45MB (800% waste)
```

---

#### 2. Cache Inefficiency

**Memory Layout:**

```
Node 1 @ 0x1000 ──> prev: 0x5000
                    next: 0x3000
                    value: "hello"

Node 2 @ 0x3000 ──> prev: 0x1000
                    next: 0x7000
                    value: "world"

Node 3 @ 0x7000 ──> prev: 0x3000
                    next: NULL
                    value: "redis"
```

**Traversal Pattern:**
- Jump from 0x1000 → 0x3000 (12KB gap)
- Jump from 0x3000 → 0x7000 (16KB gap)
- **CPU cache misses** on every access

---

### Solution 1: Ziplist

**Ziplist** is a memory-efficient, contiguous data structure that stores elements sequentially in a single memory block.

#### Memory Layout

```
┌─────────────────────────────────────────────────────────────┐
│ Total Bytes │ Tail Offset │ Num Elements │ Entries... │ END │
│   (4 bytes) │  (4 bytes)  │  (2 bytes)   │            │ 0xFF│
└─────────────────────────────────────────────────────────────┘
```

**Example:**

```
Ziplist containing ["hello", "world", "redis"]:

┌────┬────┬────┬─────────────────────────────────────────────┬────┐
│ 50 │ 35 │ 3  │ Entry1 │ Entry2 │ Entry3                   │0xFF│
└────┴────┴────┴─────────────────────────────────────────────┴────┘
  │    │    │
  │    │    └─ Number of elements (3)
  │    └────── Offset to last entry (35 bytes)
  └─────────── Total size (50 bytes)
```

---

#### Entry Structure

Each entry contains:
1. **Previous Entry Length**: Variable (1 or 5 bytes)
2. **Encoding**: Variable (describes data type)
3. **Data**: Actual content

**Entry Layout:**

```
┌──────────────────┬──────────┬────────────┐
│ Prev Entry Len   │ Encoding │ Data       │
│ (1 or 5 bytes)   │ (varies) │ (varies)   │
└──────────────────┴──────────┴────────────┘
```

---

#### Previous Entry Length Encoding

**Optimization for small lengths:**

```c
if (prev_len <= 253) {
    // Store in 1 byte
    *p = prev_len;
} else {
    // Store 0xFE + 4 bytes
    *p = 0xFE;
    *(uint32_t*)(p+1) = prev_len;
}
```

**Example:**

```
Entry after 10-byte entry:
┌────┬──────┬──────────┐
│ 10 │ enc  │ "hello"  │
└────┴──────┴──────────┘
  │
  └─ Previous entry was 10 bytes (fits in 1 byte)

Entry after 300-byte entry:
┌────┬────┬────┬────┬────┬────┬──────┬──────────┐
│0xFE│0x2C│0x01│0x00│0x00│enc │ data │          │
└────┴────┴────┴────┴────┴────┴──────┴──────────┘
  │    └──────────┬─────────┘
  │               └─ 300 (0x012C) in 4 bytes
  └─ Flag indicating 4-byte length follows
```

---

#### Encoding Field

**First 2 bits determine type:**

| Bits | Type | Description |
|------|------|-------------|
| `00` | String | Length ≤ 63 bytes (6 bits for length) |
| `01` | String | Length ≤ 16,383 bytes (14 bits for length) |
| `10` | String | Length > 16,383 bytes (32 bits for length) |
| `11` | Integer | Next 2 bits specify integer size |

**String Encoding:**

```
Type 00 (small string):
┌──┬──────┐
│00│LLLLLL│ + data
└──┴──────┘
   └─ 6-bit length (max 63)

Type 01 (medium string):
┌──┬──────┬────────┐
│01│LLLLLL│LLLLLLLL│ + data
└──┴──────┴────────┘
   └────┬────────┘
        └─ 14-bit length (max 16,383)

Type 10 (large string):
┌──┬──────┬────┬────┬────┬────┐
│10│000000│LL  │LL  │LL  │LL  │ + data
└──┴──────┴────┴────┴────┴────┘
           └────────┬─────────┘
                    └─ 32-bit length
```

**Integer Encoding:**

```
Type 11 (integer):
┌──┬──┬────────┐
│11│XX│ data   │
└──┴──┴────────┘
    │
    └─ 00: 16-bit int
       01: 32-bit int
       10: 64-bit int
       11: 24-bit or 8-bit int
```

---

#### Complete Example

**Ziplist with 3 elements: ["redis", 42, "world"]**

```
┌────────────────────────────────────────────────────────────────┐
│ Total: 45 │ Tail: 32 │ Count: 3 │                               │
├────────────────────────────────────────────────────────────────┤
│ Entry 1: "redis"                                               │
│ ┌────┬────────┬─────┬─────┬─────┬─────┬─────┐                 │
│ │ 0  │00000101│ 'r' │ 'e' │ 'd' │ 'i' │ 's' │                 │
│ └────┴────────┴─────┴─────┴─────┴─────┴─────┘                 │
│   │      │                                                      │
│   │      └─ String, length 5                                   │
│   └─ No previous entry                                         │
├────────────────────────────────────────────────────────────────┤
│ Entry 2: 42                                                    │
│ ┌────┬────────┬────┬────┐                                      │
│ │ 7  │11000000│0x2A│0x00│                                      │
│ └────┴────────┴────┴────┘                                      │
│   │      │                                                      │
│   │      └─ 16-bit integer                                     │
│   └─ Previous entry: 7 bytes                                   │
├────────────────────────────────────────────────────────────────┤
│ Entry 3: "world"                                               │
│ ┌────┬────────┬─────┬─────┬─────┬─────┬─────┐                 │
│ │ 4  │00000101│ 'w' │ 'o' │ 'r' │ 'l' │ 'd' │                 │
│ └────┴────────┴─────┴─────┴─────┬─────┴─────┘                 │
│   │      │                       ▲                              │
│   │      └─ String, length 5     │                              │
│   └─ Previous entry: 4 bytes     └─ Tail offset points here    │
├────────────────────────────────────────────────────────────────┤
│ End: 0xFF                                                      │
└────────────────────────────────────────────────────────────────┘
```

---

#### Ziplist Operations

**Push (tail):**

```c
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    // Calculate new size
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl));
    size_t reqlen = /* encoded size of new entry */;

    // Realloc ziplist
    zl = ziplistResize(zl, curlen + reqlen);

    // Insert entry at tail
    // Update tail offset
    // Update count

    return zl;
}
```

**Pop (head/tail):**

```c
unsigned char *ziplistPop(unsigned char *zl, int where) {
    // Read entry at position
    // Calculate entry size
    // memmove to remove entry
    // Realloc to shrink
    // Update metadata

    return zl;
}
```

**Find:**

```c
unsigned char *ziplistFind(unsigned char *p, unsigned char *vstr, unsigned int vlen) {
    while (p[0] != ZIP_END) {
        // Decode entry
        // Compare with target
        if (match) return p;

        // Skip to next entry
        p += entry_size;
    }
    return NULL;
}
```

---

#### Ziplist Advantages

1. ✅ **Sequential memory**: Cache-friendly traversal
2. ✅ **No pointers**: Zero per-element overhead
3. ✅ **O(1) head/tail access**: Via tail offset
4. ✅ **Memory efficient**: Compact encoding

**Memory Comparison:**

```
Linked List (3 elements):
- Nodes: 3 × 24 = 72 bytes
- Data: 15 bytes
- Total: 87 bytes

Ziplist (3 elements):
- Header: 10 bytes
- Entries: ~25 bytes (with encoding)
- Total: 35 bytes

Savings: 60%
```

---

#### Ziplist Disadvantages

1. ❌ **Insert/Delete middle**: O(N) - requires memmove
2. ❌ **Reallocation**: Growth requires new allocation + copy
3. ❌ **Cascade updates**: Changing entry size may cascade

**Cascade Update Problem:**

```
Before:
Entry1 (252 bytes) | Entry2 (prev_len: 252, 1 byte) | Entry3...

After inserting 2 bytes in Entry1:
Entry1 (254 bytes) | Entry2 (prev_len: 254, 5 bytes!) | Entry3 (prev_len changed!)...
                                   └─ Needs 0xFE prefix

Result: Chain reaction of updates
```

---

### Solution 2: Quicklist

**Quicklist** = Doubly linked list of ziplists

#### Structure

```
┌──────────────────────────────────────────────────────────┐
│                      Quicklist                           │
├──────────────────────────────────────────────────────────┤
│  head ──┐                                      tail ──┐  │
│         │                                             │  │
│         ▼                                             ▼  │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  │ Quicklist    │ ───> │ Quicklist    │ ───> │ Quicklist    │
│  │ Node 1       │ <─── │ Node 2       │ <─── │ Node 3       │
│  ├──────────────┤      ├──────────────┤      ├──────────────┤
│  │ *prev = NULL │      │ *prev        │      │ *prev        │
│  │ *next ────────────> │ *next ────────────> │ *next = NULL │
│  │ *ziplist     │      │ *ziplist     │      │ *ziplist     │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘
│         │                     │                     │
│         ▼                     ▼                     ▼
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  │  Ziplist 1  │       │  Ziplist 2  │       │  Ziplist 3  │
│  │  [a,b,c,d]  │       │  [e,f,g,h]  │       │  [i,j,k,l]  │
│  └─────────────┘       └─────────────┘       └─────────────┘
└──────────────────────────────────────────────────────────────┘
```

---

#### Quicklist Node

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;           // Pointer to ziplist
    unsigned int sz;             // Ziplist size in bytes
    unsigned int count : 16;     // Number of items in ziplist
    unsigned int encoding : 2;   // RAW=1 or LZF=2 (compressed)
    unsigned int container : 2;  // ZIPLIST=2
    unsigned int recompress : 1; // Was compressed?
    unsigned int attempted_compress : 1;
    unsigned int extra : 10;     // Reserved
} quicklistNode;
```

---

#### Configuration

```bash
# redis.conf
list-max-ziplist-size -2

# Meaning of negative values:
# -1: max 4KB per ziplist
# -2: max 8KB per ziplist (default)
# -3: max 16KB per ziplist
# -4: max 32KB per ziplist
# -5: max 64KB per ziplist

# Positive values = max entries per ziplist
list-max-ziplist-size 512
```

---

#### Quicklist Operations

**Push:**

```c
void quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *node = quicklist->head;

    if (node && ziplistCanFit(node->zl, sz)) {
        // Add to existing ziplist
        node->zl = ziplistPush(node->zl, value, sz, ZIPLIST_HEAD);
        node->count++;
    } else {
        // Create new node
        quicklistNode *new_node = quicklistCreateNode();
        new_node->zl = ziplistNew();
        new_node->zl = ziplistPush(new_node->zl, value, sz, ZIPLIST_HEAD);
        _quicklistInsertNodeBefore(quicklist, node, new_node);
    }
}
```

**Split:**

```c
void quicklistSplitNode(quicklistNode *node) {
    // If ziplist too large, split in half
    size_t sz = ziplistBlobLen(node->zl);

    if (sz < list_max_ziplist_size) return;

    // Create new node
    quicklistNode *new_node = quicklistCreateNode();

    // Split ziplist at midpoint
    unsigned char *zl2 = ziplistSplit(node->zl, /* midpoint */);
    new_node->zl = zl2;

    // Insert new node after current
    _quicklistInsertNodeAfter(quicklist, node, new_node);
}
```

---

#### Performance Characteristics

| Operation | Ziplist | Quicklist |
|-----------|---------|-----------|
| **LPUSH/RPUSH** | O(1)* | O(1) |
| **LPOP/RPOP** | O(1)* | O(1) |
| **LINDEX (middle)** | O(N) | O(N) |
| **LINSERT (middle)** | O(N) | O(N) |
| **Memory** | Excellent | Very Good |
| **Cache locality** | Excellent | Good (per-ziplist) |

*Amortized O(1), may require reallocation

---

### Comparison Summary

**Doubly Linked List:**
- ❌ 24 bytes overhead per element
- ❌ Poor cache locality
- ✅ O(1) insert/delete anywhere
- ✅ No reallocation

**Ziplist:**
- ✅ 1-5 bytes overhead per element
- ✅ Excellent cache locality
- ❌ O(N) insert/delete middle
- ❌ Cascade updates possible
- ❌ Best for small lists (<512 elements)

**Quicklist:**
- ✅ ~2-3 bytes overhead per element
- ✅ Good cache locality (per ziplist)
- ✅ Dynamic sizing
- ✅ Best of both worlds
- ✅ Production default

**Memory Usage (1000 elements):**

```
Doubly Linked List: ~40KB
Ziplist: ~15KB (if fits)
Quicklist (10 ziplists): ~17KB

Savings: 57%
```

---

## Chapter 23: Set Internals - Intset

### Redis Set Implementation

Redis sets store unique elements with fast membership testing.

```bash
127.0.0.1:6379> SADD myset 1 2 3 4 5
(integer) 5
127.0.0.1:6379> DEBUG OBJECT myset
Value at:0x7f8a... encoding:intset serializedlength:50
```

**Encoding:** `intset` (for integer-only sets)

---

### Set Encoding Strategies

Redis dynamically chooses encoding based on contents:

#### 1. Intset Encoding

**Used when:** All members are integers

```bash
127.0.0.1:6379> SADD numbers 10 20 30
(integer) 3
127.0.0.1:6379> OBJECT ENCODING numbers
"intset"
```

---

#### 2. Hash Table Encoding

**Used when:** Contains non-integers or too many integers

```bash
127.0.0.1:6379> SADD mixed 10 "hello" 30
(integer) 3
127.0.0.1:6379> OBJECT ENCODING mixed
"hashtable"
```

---

### Intset Structure

**Intset** is a sorted array of integers stored in a single contiguous memory block.

#### Memory Layout

```c
typedef struct intset {
    uint32_t encoding;  // 4 bytes: INT16, INT32, or INT64
    uint32_t length;    // 4 bytes: number of elements
    int8_t contents[];  // Variable: actual integers
} intset;
```

**Example:**

```
Intset with [10, 20, 30] (16-bit encoding):

┌──────────┬──────────┬────┬────┬────┬────┬────┬────┐
│ encoding │ length   │ 10 │ 00 │ 20 │ 00 │ 30 │ 00 │
│  (INT16) │   (3)    │    │    │    │    │    │    │
└──────────┴──────────┴────┴────┴────┴────┴────┴────┘
  4 bytes    4 bytes     2 bytes  2 bytes  2 bytes

Total: 8 + 6 = 14 bytes
```

---

#### Encoding Types

```c
#define INTSET_ENC_INT16 (sizeof(int16_t))  // 2 bytes
#define INTSET_ENC_INT32 (sizeof(int32_t))  // 4 bytes
#define INTSET_ENC_INT64 (sizeof(int64_t))  // 8 bytes
```

**Selection:**

```
Value Range              Encoding
-----------              --------
-32,768 to 32,767       → INT16
-2^31 to 2^31-1         → INT32
-2^63 to 2^63-1         → INT64
```

---

### Intset Operations

#### 1. Creation

```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);  // Start with smallest
    is->length = 0;
    return is;
}
```

---

#### 2. Adding Elements

**Algorithm:**

```
1. Check if value fits current encoding
2. If not, upgrade encoding
3. Binary search for insert position
4. If value exists, return (sets are unique)
5. If new, shift elements and insert
```

**Implementation:**

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;

    *success = 1;

    // Upgrade if necessary
    if (valenc > intrev32ifbe(is->encoding)) {
        return intsetUpgradeAndAdd(is, value);
    }

    // Binary search
    if (intsetSearch(is, value, &pos)) {
        *success = 0;  // Already exists
        return is;
    }

    // Resize
    is = intsetResize(is, intrev32ifbe(is->length) + 1);

    // Shift elements
    if (pos < intrev32ifbe(is->length)) {
        intsetMoveTail(is, pos, pos + 1);
    }

    // Insert
    _intsetSet(is, pos, value);
    is->length = intrev32ifbe(intrev32ifbe(is->length) + 1);

    return is;
}
```

---

#### 3. Upgrading Encoding

**When adding 40000 to INT16 intset:**

```
Before (INT16):
┌──────┬────┬────┬────┬────┬────┬────┐
│ enc  │len │ 10 │ 00 │ 20 │ 00 │ 30 │ 00 │
│INT16 │ 3  │        │        │        │
└──────┴────┴────┴────┴────┴────┴────┴────┘

After (INT32) - 40000 added:
┌──────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ enc  │len │ 10 │ 00 │ 00 │ 00 │ 20 │ 00 │ 00 │ 00 │ 30 │ 00 │ 00 │ 00 │9C40│ 00 │ 00 │ 00 │
│INT32 │ 4  │          10            │          20            │          30            │40000│
└──────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
```

**Upgrade Implementation:**

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);

    // Determine prepend (value < all) or append (value > all)
    int prepend = value < 0 ? 1 : 0;

    // Update encoding
    is->encoding = intrev32ifbe(newenc);

    // Resize for new encoding
    is = intsetResize(is, length + 1);

    // Move existing elements (from back to avoid overwrites)
    while (length--) {
        _intsetSet(is, length + prepend, _intsetGetEncoded(is, length, curenc));
    }

    // Insert new value at prepend (0) or end (length+1)
    if (prepend) {
        _intsetSet(is, 0, value);
    } else {
        _intsetSet(is, intrev32ifbe(is->length), value);
    }

    is->length = intrev32ifbe(intrev32ifbe(is->length) + 1);
    return is;
}
```

---

#### 4. Binary Search

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length) - 1, mid = -1;
    int64_t cur = -1;

    // Empty set
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    }

    // Value out of range
    if (value > _intsetGet(is, max)) {
        if (pos) *pos = intrev32ifbe(is->length);
        return 0;
    } else if (value < _intsetGet(is, 0)) {
        if (pos) *pos = 0;
        return 0;
    }

    // Binary search
    while (max >= min) {
        mid = ((unsigned)min + (unsigned)max) >> 1;
        cur = _intsetGet(is, mid);

        if (value > cur) {
            min = mid + 1;
        } else if (value < cur) {
            max = mid - 1;
        } else {
            break;  // Found
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;  // Found
    } else {
        if (pos) *pos = min;
        return 0;  // Not found
    }
}
```

---

#### 5. Removing Elements

```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;

    *success = 0;

    // Check if value could be in intset
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is, value, &pos)) {
        uint32_t len = intrev32ifbe(is->length);
        *success = 1;

        // Shift elements left
        if (pos < (len - 1)) {
            intsetMoveTail(is, pos + 1, pos);
        }

        // Resize
        is = intsetResize(is, len - 1);
        is->length = intrev32ifbe(len - 1);
    }

    return is;
}
```

---

### Configuration Threshold

```bash
# redis.conf
set-max-intset-entries 512

# If elements > 512, convert to hashtable
```

**Example:**

```bash
127.0.0.1:6379> CONFIG GET set-max-intset-entries
1) "set-max-intset-entries"
2) "512"

127.0.0.1:6379> SADD bigset <513 integers>
127.0.0.1:6379> OBJECT ENCODING bigset
"hashtable"  # Converted
```

---

### Performance Analysis

**Time Complexity:**

| Operation | Intset | Hashtable |
|-----------|--------|-----------|
| **SADD** | O(N) | O(1) |
| **SREM** | O(N) | O(1) |
| **SISMEMBER** | O(log N) | O(1) |
| **SINTER** | O(N·M) | O(N·M) |
| **Memory** | Excellent | Good |

**Why O(log N) for SISMEMBER?**
- Binary search on sorted array
- Much faster than O(N) linear scan
- Acceptable for small sets (<512 elements)

---

**Space Comparison:**

```
Hashtable (512 integers):
- Buckets: 512 × 16 = 8KB
- Overhead: ~4KB
- Total: ~12KB

Intset (512 int32):
- Header: 8 bytes
- Data: 512 × 4 = 2048 bytes
- Total: ~2KB

Savings: 83%
```

---

### When Intset is Used

**Conditions:**
1. ✅ All members are integers
2. ✅ Count ≤ `set-max-intset-entries` (default 512)

**Conversion to Hashtable:**
- Adding non-integer member
- Exceeding entry limit

**Example:**

```bash
127.0.0.1:6379> SADD numbers 1 2 3
127.0.0.1:6379> OBJECT ENCODING numbers
"intset"

127.0.0.1:6379> SADD numbers "hello"
127.0.0.1:6379> OBJECT ENCODING numbers
"hashtable"  # Permanent conversion
```

---

## Chapter 24: Geospatial Queries and Geohash

### Redis Geospatial Support

Redis provides commands for storing and querying geographic coordinates.

**Commands:**
- `GEOADD`: Add locations
- `GEODIST`: Distance between points
- `GEORADIUS`: Find locations within radius
- `GEOSEARCH`: Advanced proximity search

---

### Basic Usage

```bash
# Add locations
127.0.0.1:6379> GEOADD cities 13.361389 38.115556 "Palermo"
(integer) 1
127.0.0.1:6379> GEOADD cities 15.087269 37.502669 "Catania"
(integer) 1

# Find distance
127.0.0.1:6379> GEODIST cities Palermo Catania km
"166.2742"

# Find nearby (within 100km of Palermo)
127.0.0.1:6379> GEORADIUS cities 13.361389 38.115556 100 km
1) "Palermo"
```

---

### The Challenge

**Naive Approach:**

```
For each query:
  For each location:
    distance = sqrt((x2-x1)² + (y2-y1)²)
    if distance <= radius:
      add to results

Complexity: O(N) per query
```

**Problem:** N-dimensional distance calculations are expensive
- Square root computation
- Floating-point arithmetic
- No index possible

---

### GeoHash Algorithm

**GeoHash** encodes 2D coordinates (lat, lon) into a single integer, enabling fast proximity searches.

#### Core Concept

**Divide and Conquer:**

```
1. Split world in half
2. Assign bit based on which half
3. Repeat recursively
4. Concatenate bits
```

---

#### Step-by-Step Example

**Goal:** Encode location (Longitude: 13.361, Latitude: 38.115)

**Step 1: Longitude (-180 to 180)**

```
Iteration 1: [-180, 180]
  Midpoint: 0
  13.361 > 0 → Right half → Bit: 1
  Range: [0, 180]

Iteration 2: [0, 180]
  Midpoint: 90
  13.361 < 90 → Left half → Bit: 0
  Range: [0, 90]

Iteration 3: [0, 90]
  Midpoint: 45
  13.361 < 45 → Left half → Bit: 0
  Range: [0, 45]

Continue for 26 iterations...
Final: 01001010110001... (26 bits)
```

**Step 2: Latitude (-90 to 90)**

```
Iteration 1: [-90, 90]
  Midpoint: 0
  38.115 > 0 → Right half → Bit: 1
  Range: [0, 90]

Iteration 2: [0, 90]
  Midpoint: 45
  38.115 < 45 → Left half → Bit: 0
  Range: [0, 45]

Continue for 26 iterations...
Final: 10011101001110... (26 bits)
```

---

#### Interleaving Bits

**Why interleave?**

Concatenation would only allow zooming in one dimension:
```
Concat: [lat bits][lon bits]
Removing bits from right only affects longitude
```

**Interleaving solution:**
```
Odd bits  = Longitude
Even bits = Latitude

Interleaved: 1 0 1 0 0 1 0 1 1 0 0 1 ...
             │ │ │ │ │ │ │ │ │ │ │ │
             │ └─│─└─│─└─│─└─│─└─│─└─ Latitude
             └───┴───┴───┴───┴───┴─── Longitude
```

---

#### Efficient Interleaving

**Naive approach: Loop (O(N))**

```c
uint64_t interleave_naive(uint32_t x, uint32_t y) {
    uint64_t result = 0;
    for (int i = 0; i < 32; i++) {
        result |= ((uint64_t)(x & (1 << i)) << i);
        result |= ((uint64_t)(y & (1 << i)) << (i + 1));
    }
    return result;
}
```

---

**Optimized: Bit manipulation (O(1))**

```c
uint64_t interleave64(uint32_t xlo, uint32_t ylo) {
    static const uint64_t B[] = {
        0x5555555555555555ULL,
        0x3333333333333333ULL,
        0x0F0F0F0F0F0F0F0FULL,
        0x00FF00FF00FF00FFULL,
        0x0000FFFF0000FFFFULL
    };
    static const unsigned S[] = {1, 2, 4, 8, 16};

    uint64_t x = xlo;
    uint64_t y = ylo;

    x = (x | (x << S[4])) & B[4];
    y = (y | (y << S[4])) & B[4];

    x = (x | (x << S[3])) & B[3];
    y = (y | (y << S[3])) & B[3];

    x = (x | (x << S[2])) & B[2];
    y = (y | (y << S[2])) & B[2];

    x = (x | (x << S[1])) & B[1];
    y = (y | (y << S[1])) & B[1];

    x = (x | (x << S[0])) & B[0];
    y = (y | (y << S[0])) & B[0];

    return x | (y << 1);
}
```

**Magic numbers:**
- `0x5555...` = `0101010101...` (odd bits)
- `0x3333...` = `0011001100...` (pairs)
- `0x0F0F...` = `0000111100...` (nibbles)

---

### Computing GeoHash

**Redis Implementation:**

```c
void geohashEncode(const GeoHashRange *lat_range,
                   const GeoHashRange *lon_range,
                   double latitude, double longitude,
                   uint8_t step, GeoHashBits *hash) {
    // Calculate relative offsets (0 to 1)
    double lat_offset =
        (latitude - lat_range->min) / (lat_range->max - lat_range->min);
    double lon_offset =
        (longitude - lon_range->min) / (lon_range->max - lon_range->min);

    // Scale to integer (26-bit precision)
    lat_offset *= (1ULL << step);
    lon_offset *= (1ULL << step);

    // Interleave
    hash->bits = interleave64((uint32_t)lat_offset, (uint32_t)lon_offset);
    hash->step = step * 2;  // 26 bits each = 52 bits total
}
```

---

### Proximity via Prefix Matching

**Key Insight:** Locations with same GeoHash prefix are nearby

```
Location A: 11010110... (GeoHash)
Location B: 11010101... (Same 5-bit prefix)
Location C: 10110010... (Different prefix)

A and B are closer than A and C
```

**Zoom Levels:**

```
52 bits: ~0.6m precision
50 bits: ~2.4m precision
48 bits: ~9.6m precision
...
26 bits: ~10m precision
20 bits: ~600m precision
```

---

### Finding Neighbors

**Algorithm:**

```
1. Calculate GeoHash of query point
2. Determine search radius in bits
3. Generate neighbor GeoHashes
4. Search for points with matching prefixes
```

**Neighbor GeoHashes:**

```
Current: 110101...

Neighbors (move 1 grid cell):
North:     110111...
South:     110001...
East:      110110...
West:      110100...
NorthEast: 110111...
...
```

---

### Redis Internal Storage

**Sorted Set Encoding:**

```bash
127.0.0.1:6379> GEOADD locations 13.361 38.115 "Palermo"
127.0.0.1:6379> TYPE locations
zset

127.0.0.1:6379> ZSCORE locations Palermo
"3479099956230698"
         └─ This is the GeoHash!
```

**Under the hood:**

```c
void geoaddCommand(client *c) {
    // For each location:
    double lat = /* parse latitude */;
    double lon = /* parse longitude */;

    // Encode as GeoHash
    GeoHashBits hash;
    geohashEncode(&lat_range, &lon_range, lat, lon, 26, &hash);

    // Store in sorted set (score = GeoHash)
    zsetAdd(key, hash.bits, member);
}
```

---

### GEORADIUS Implementation

**High-level algorithm:**

```c
void georadiusGeneric(client *c, int flags) {
    // 1. Compute GeoHash of center point
    GeoHashBits hash;
    geohashEncode(/* center lat/lon */);

    // 2. Get GeoHash ranges for radius
    GeoHashRadius radius_area = geohashGetAreasByRadius(lat, lon, radius_m);

    // 3. For each range, query sorted set
    for (int i = 0; i < 9; i++) {  // 9 neighbor cells
        GeoHashBits min = radius_area.hash[i].min;
        GeoHashBits max = radius_area.hash[i].max;

        // ZRANGEBYSCORE on GeoHash range
        membersInRange(zset, min.bits, max.bits);
    }

    // 4. Filter by exact distance
    for (each candidate) {
        double distance = geohashGetDistance(center, candidate);
        if (distance <= radius) {
            addReply(candidate);
        }
    }
}
```

---

### Performance Analysis

**Naive Distance Calculation:**
```
For 1M locations:
- Time per query: O(N) = 1M distance calculations
- With 1000 queries/sec: 1B calculations/sec
```

**GeoHash Approach:**
```
For 1M locations:
- GeoHash lookup: O(log N) = ~20 comparisons
- Neighbor cells: 9 × O(log N) = ~180 comparisons
- Exact distance: O(K) where K = candidates (~100)
- Total: ~300 operations

Speedup: 3,333× faster
```

---

### Precision vs Performance

**Bits vs Accuracy:**

| Bits | Grid Size | Use Case |
|------|-----------|----------|
| 52 | 0.6m | Indoor navigation |
| 48 | 9.6m | Building-level |
| 40 | 153m | City block |
| 32 | 2.4km | Neighborhood |
| 26 | 10m | Redis default |

**Trade-off:**
- More bits = Higher precision, more memory
- Fewer bits = Coarser grid, less memory

---

### Summary

**GeoHash Algorithm:**
1. ✅ Encode (lat, lon) as single integer
2. ✅ Recursive binary division
3. ✅ Bit interleaving for 2D zooming
4. ✅ Prefix matching for proximity

**Implementation:**
1. ✅ 26-bit precision per dimension (52 bits total)
2. ✅ Stored as sorted set scores
3. ✅ Neighbor cell generation
4. ✅ Two-phase filter (GeoHash + exact distance)

**Performance:**
- **GEOADD**: O(log N)
- **GEORADIUS**: O(log N + K) where K = results
- **Memory**: 8 bytes per location (vs 16 bytes for lat+lon)

**Next:** String internals with Simple Dynamic Strings (SDS)

---
## Chapter 25: String Internals - Simple Dynamic Strings (SDS)

### Why Not C Strings?

Redis doesn't use standard C strings (`char*`) due to several limitations:

#### Problem 1: O(N) Length Calculation

```c
// C string
char *str = "hello";

// Get length
size_t len = strlen(str);  // O(N) - scans until '\0'
```

**For Redis:**
- Millions of string operations per second
- Repeated `strlen()` calls = wasted CPU cycles

---

#### Problem 2: Inefficient Appends

```c
// Append to C string
char *str = "hello";
char *new_str = malloc(strlen(str) + 6);  // Realloc every time
strcpy(new_str, str);
strcat(new_str, "world");
free(str);
str = new_str;
```

**Problems:**
- `malloc()` + `free()` on every append
- No pre-allocation strategy
- Memory fragmentation

---

#### Problem 3: Not Binary-Safe

```c
// C string
char str[] = "hello\0world";

printf("%s", str);  // Output: "hello"
                    // Lost "world" after null byte!
```

**Redis needs:**
- Store arbitrary bytes (serialized objects, images)
- Support embedded `\0` characters

---

### Solution: Simple Dynamic Strings (SDS)

**SDS** is Redis's custom string implementation with O(1) length, efficient appends, and binary safety.

#### Basic Structure

```c
struct sdshdr8 {
    uint8_t len;        // Current string length
    uint8_t alloc;      // Allocated capacity
    unsigned char flags; // Type of header
    char buf[];         // String data (flexible array member)
};
```

---

### SDS Header Types

Redis uses **different header sizes** based on string length to save memory:

```c
// For strings < 32 bytes
struct __attribute__((__packed__)) sdshdr5 {
    unsigned char flags; // 3 bits type, 5 bits length
    char buf[];
};

// For strings < 256 bytes
struct __attribute__((__packed__)) sdshdr8 {
    uint8_t len;        // 1 byte
    uint8_t alloc;      // 1 byte
    unsigned char flags; // 1 byte
    char buf[];
};

// For strings < 64KB
struct __attribute__((__packed__)) sdshdr16 {
    uint16_t len;       // 2 bytes
    uint16_t alloc;     // 2 bytes
    unsigned char flags; // 1 byte
    char buf[];
};

// For strings < 4GB
struct __attribute__((__packed__)) sdshdr32 {
    uint32_t len;       // 4 bytes
    uint32_t alloc;     // 4 bytes
    unsigned char flags; // 1 byte
    char buf[];
};

// For huge strings
struct __attribute__((__packed__)) sdshdr64 {
    uint64_t len;       // 8 bytes
    uint64_t alloc;     // 8 bytes
    unsigned char flags; // 1 byte
    char buf[];
};
```

**Header Type Selection:**

| String Length | Header Type | Overhead |
|--------------|-------------|----------|
| < 32 bytes | sdshdr5 | 1 byte |
| < 256 bytes | sdshdr8 | 3 bytes |
| < 64KB | sdshdr16 | 5 bytes |
| < 4GB | sdshdr32 | 9 bytes |
| ≥ 4GB | sdshdr64 | 17 bytes |

---

### SDS Memory Layout

**Example: "hello" (5 bytes)**

```
┌──────────────────────────────────────┐
│ sdshdr8 (using 8-bit header)         │
├──────────────────────────────────────┤
│ len:   5          (1 byte)           │
│ alloc: 10         (1 byte)           │
│ flags: SDS_TYPE_8 (1 byte)           │
│ buf:   ['h','e','l','l','o','\0',...] │
│        └───────┬───────┘               │
│                └─ Null-terminated     │
│                   for C compatibility │
└──────────────────────────────────────┘

Total: 3 bytes (header) + 6 bytes (data) = 9 bytes
       (vs 6 bytes for C string, but with O(1) length)
```

---

### SDS Operations

#### 1. Create New SDS

```c
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);  // Determine header type

    // Allocate header + string + null terminator
    int hdrlen = sdsHdrSize(type);
    sh = zmalloc(hdrlen + initlen + 1);

    // Initialize header
    s = (char*)sh + hdrlen;
    switch (type) {
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8, s);
            sh->len = initlen;
            sh->alloc = initlen;
            break;
        }
        // ... other types
    }

    // Copy data
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';  // Null terminate

    return s;
}
```

**Pointer Trick:**

```
Allocated Memory:
┌──────────┬────────────┐
│ Header   │ String buf │
└──────────┴────────────┘
           ▲
           └─ s points here (not to header!)
```

**Why?** SDS can be passed to C functions expecting `char*`

---

#### 2. Get Length (O(1))

```c
size_t sdslen(const sds s) {
    unsigned char flags = s[-1];  // Read flags byte before s
    switch (flags & SDS_TYPE_MASK) {
        case SDS_TYPE_8:
            return SDS_HDR(8, s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16, s)->len;
        // ... other types
    }
}
```

**Memory Access:**

```
s points to buf:
          s[-1]    s[0]
            │       │
            ▼       ▼
┌──┬──┬──┬──┬───┬───┬───┬───┬───┬───┐
│..│..│..│fl│'h'│'e'│'l'│'l'│'o'│'\0'│
└──┴──┴──┴──┴───┴───┴───┴───┴───┴───┘
          ▲
          └─ flags (header type encoded)
```

**Comparison:**

```
C string:    strlen(s)  → O(N)
SDS:         sdslen(s)  → O(1)
```

---

#### 3. Append (Efficient)

```c
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    // Make room (may realloc)
    s = sdsMakeRoomFor(s, len);

    // Copy new data
    memcpy(s + curlen, t, len);

    // Update length
    sdssetlen(s, curlen + len);
    s[curlen + len] = '\0';

    return s;
}
```

**Smart Reallocation:**

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    size_t len = sdslen(s);
    size_t newlen = len + addlen;

    // Pre-allocate extra space
    if (newlen < SDS_MAX_PREALLOC) {
        newlen *= 2;  // Double capacity
    } else {
        newlen += SDS_MAX_PREALLOC;  // Add 1MB
    }

    // Realloc if needed
    if (sdsavail(s) < addlen) {
        s = sdsRealloc(s, newlen);
    }

    return s;
}
```

**Growth Strategy:**

```
Initial: "hello" (len=5, alloc=5)
Append "world": Need 5 more bytes
  → Allocate 20 bytes (5+5)*2
  → Actual usage: 10 bytes
  → Free space: 10 bytes

Next append: If ≤10 bytes, no realloc needed!
```

---

### Redis Object Encodings for Strings

Redis chooses encoding based on value:

```c
robj *tryObjectEncoding(robj *o) {
    sds s = o->ptr;

    // 1. Try integer encoding
    long value;
    if (string2l(s, sdslen(s), &value)) {
        // Value is integer
        if (value >= 0 && value < OBJ_SHARED_INTEGERS) {
            // Use shared integer object
            decrRefCount(o);
            return shared.integers[value];
        } else {
            // Create integer-encoded object
            o->encoding = OBJ_ENCODING_INT;
            o->ptr = (void*)value;
            sdsfree(s);
        }
        return o;
    }

    // 2. Try EMBSTR encoding
    if (sdslen(s) <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        robj *emb = createEmbeddedStringObject(s, sdslen(s));
        decrRefCount(o);
        return emb;
    }

    // 3. Default: RAW encoding (already using SDS)
    trimStringObjectIfNeeded(o);
    return o;
}
```

---

#### OBJ_ENCODING_INT

**Store integer as pointer:**

```c
robj *createStringObjectFromLongLong(long long value) {
    robj *o = createObject(OBJ_STRING, NULL);
    o->encoding = OBJ_ENCODING_INT;
    o->ptr = (void*)value;  // No SDS allocated!
    return o;
}
```

**Memory:**

```
For "42":
┌──────────────────┐
│ redisObject      │
│ - type: STRING   │
│ - encoding: INT  │
│ - ptr: 42        │  ← Integer stored directly
└──────────────────┘

Total: 16 bytes (just the object)
```

---

#### OBJ_ENCODING_EMBSTR

**Embed SDS in object:**

```c
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    // Allocate object + SDS header + data in one block
    robj *o = zmalloc(sizeof(robj) + sizeof(struct sdshdr8) + len + 1);

    // Setup SDS
    struct sdshdr8 *sh = (void*)(o + 1);
    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;

    // Setup object
    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh->buf;

    // Copy data
    memcpy(sh->buf, ptr, len);
    sh->buf[len] = '\0';

    return o;
}
```

**Memory Layout:**

```
For "hello" (5 bytes):
┌─────────────────────────────────────────┐
│ Single allocation (64 bytes)            │
├─────────────────────────────────────────┤
│ redisObject (16 bytes)                  │
│ ├─ type: STRING                         │
│ ├─ encoding: EMBSTR                     │
│ └─ ptr ──┐                              │
├──────────┼──────────────────────────────┤
│ sdshdr8  │(3 bytes)                     │
│ ├─ len: 5│                              │
│ ├─ alloc:│5                             │
│ └─ flags:│SDS_TYPE_8                    │
├──────────┼──────────────────────────────┤
│ buf <────┘                              │
│ ['h','e','l','l','o','\0']              │
└─────────────────────────────────────────┘

Total: 16 + 3 + 6 = 25 bytes
Fits in 64-byte allocation (Redis minimum)
```

**Why 44 bytes limit?**

```
64 bytes (allocation)
- 16 bytes (redisObject)
- 3 bytes (sdshdr8)
- 1 byte (null terminator)
= 44 bytes available for string
```

---

#### OBJ_ENCODING_RAW

**Separate SDS allocation:**

```c
robj *createStringObject(const char *ptr, size_t len) {
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
        return createEmbeddedStringObject(ptr, len);
    }
    return createRawStringObject(ptr, len);
}

robj *createRawStringObject(const char *ptr, size_t len) {
    return createObject(OBJ_STRING, sdsnewlen(ptr, len));
}
```

**Memory Layout:**

```
For "Long string..." (100 bytes):

┌──────────────────┐
│ redisObject      │
│ - type: STRING   │
│ - encoding: RAW  │
│ - ptr ────────┐  │
└───────────────┼──┘
                │
                ↓
        ┌───────────────────┐
        │ SDS (separate)    │
        │ - len: 100        │
        │ - alloc: 200      │
        │ - buf: [data...]  │
        └───────────────────┘

Total: 16 + (3 + 101) = 120 bytes
```

---

### Performance Comparison

**Append Benchmark:**

```c
// C string (naive)
char *str = malloc(6);
strcpy(str, "hello");

for (int i = 0; i < 10000; i++) {
    size_t len = strlen(str);
    str = realloc(str, len + 6);
    strcat(str, "world");
}
// Time: ~50ms (10,000 reallocs)

// SDS (smart)
sds s = sdsnew("hello");

for (int i = 0; i < 10000; i++) {
    s = sdscat(s, "world");
}
// Time: ~2ms (~20 reallocs due to pre-allocation)

Speedup: 25×
```

---

### Binary Safety Example

```c
// C string (fails)
char binary[] = "hello\0world";
printf("%zu", strlen(binary));  // Output: 5 (stops at \0)

// SDS (works)
sds s = sdsnewlen("hello\0world", 11);
printf("%zu", sdslen(s));  // Output: 11 (correct)

// Can store any bytes
unsigned char image_data[] = {0xFF, 0xD8, 0xFF, 0xE0, ...};
sds image_sds = sdsnewlen(image_data, sizeof(image_data));
```

---

### Summary

**SDS vs C Strings:**

| Feature | C String | SDS |
|---------|----------|-----|
| **Length lookup** | O(N) | O(1) |
| **Append** | O(N) reallocations | O(1) amortized |
| **Binary safe** | ❌ No (stops at \0) | ✅ Yes |
| **Memory overhead** | 0 bytes | 1-17 bytes (header) |
| **Pre-allocation** | ❌ No | ✅ Yes |
| **C compatible** | ✅ Native | ✅ buf is null-terminated |

**Encoding Strategy:**

```
Value Type       Encoding       Memory
----------       --------       ------
Integer          INT            16 bytes
String ≤44B      EMBSTR         ~25-60 bytes
String >44B      RAW            16 + SDS size
```

**Design Philosophy:**
- **Optimize common case** (small strings, appends)
- **Pay for what you use** (different header sizes)
- **Backward compatible** (works with C APIs)

**Next:** HyperLogLog for cardinality estimation

---

# Part 7: Advanced Algorithms

## Chapter 26: HyperLogLog and Cardinality Estimation

### The Cardinality Problem

**Problem:** Count unique elements in a massive stream

**Example Use Cases:**
- Unique visitors to a website (millions/day)
- Unique IP addresses in network traffic
- Distinct words in documents
- Unique users clicking an ad

---

### Naive Solutions

#### 1. Hash Set

```go
uniqueUsers := make(map[string]bool)

for event := range events {
    uniqueUsers[event.UserID] = true
}

count := len(uniqueUsers)  // Exact count
```

**Memory:**
- 1M unique users × 36 bytes/entry = 36 MB
- 1B unique users × 36 bytes/entry = 36 GB ❌

---

#### 2. Bitmap

```go
bitmap := make([]byte, 1_000_000_000/8)  // 125 MB

for event := range events {
    hash := hashUserID(event.UserID)
    setBit(bitmap, hash % 1_000_000_000)
}

count := countSetBits(bitmap)  // Approximate count
```

**Memory:** Fixed (125 MB for 1B possible IDs)
**Accuracy:** Limited (hash collisions)

---

### HyperLogLog Solution

**HyperLogLog (HLL)** provides approximate cardinality with:
- **Fixed memory**: 12 KB (regardless of cardinality)
- **Standard error**: ±0.81%
- **Mergeable**: Combine multiple HLLs

**Redis Commands:**

```bash
# Add elements
127.0.0.1:6379> PFADD unique_visitors user1 user2 user3
(integer) 1

# Get count
127.0.0.1:6379> PFCOUNT unique_visitors
(integer) 3

# Merge multiple HLLs
127.0.0.1:6379> PFMERGE total hll1 hll2 hll3
OK
127.0.0.1:6379> PFCOUNT total
(integer) 15234
```

---

### Flajolet-Martin Algorithm

**Core idea:** Use hash distribution properties to estimate cardinality

#### Observation

For uniformly distributed hash values (binary):

```
Probability of rightmost 1-bit at position k:

P(position 0) = 1/2     (50%)     e.g., ...0001
P(position 1) = 1/4     (25%)     e.g., ...0010
P(position 2) = 1/8     (12.5%)   e.g., ...0100
P(position k) = 1/2^(k+1)
```

**Intuition:** If we see rightmost bit at position 5, we've likely seen ~2^5 = 32 different hashes.

---

#### Algorithm

**Step 1: Hash each element**

```
Element     Hash (binary)         Rightmost 1-bit position
-------     -------------         ------------------------
user1    →  ...10110100          →  2
user2    →  ...01001000          →  3
user3    →  ...10000001          →  0
user4    →  ...00100000          →  5
```

**Step 2: Track maximum position**

```
max_position = max(2, 3, 0, 5) = 5
```

**Step 3: Estimate cardinality**

```
Estimate = 2^max_position = 2^5 = 32
```

---

#### Example Walkthrough

**Processing 8 elements:**

```
Elements: [A, B, C, D, E, F, G, H]

Hash results (showing last 8 bits):
A → 10110100  (rightmost 1 at position 2)
B → 01001000  (rightmost 1 at position 3)
C → 10010001  (rightmost 1 at position 0)
D → 00100000  (rightmost 1 at position 5)
E → 11000010  (rightmost 1 at position 1)
F → 01110100  (rightmost 1 at position 2)
G → 10001000  (rightmost 1 at position 3)
H → 00010000  (rightmost 1 at position 4)

Max position: 5
Estimate: 2^5 = 32
Actual: 8

Error: 300% (very high for small samples!)
```

---

### Improving Accuracy

#### Problem 1: High Variance

Single observation → high error

**Solution: Stochastic Averaging**

Divide hash space into **m buckets**, track max for each:

```
Hash → [bucket_index | remaining_bits]
        (use first k bits for bucket selection)
```

**Algorithm:**

```
buckets[m]  // m = 2^k buckets

for element in stream:
    hash = hash(element)
    bucket_index = hash >> (64 - k)  // First k bits
    remaining = hash & ((1 << (64-k)) - 1)  // Last 64-k bits

    position = rightmost_1_bit(remaining)
    buckets[bucket_index] = max(buckets[bucket_index], position)

// Harmonic mean of estimates
estimate = m * 2^(average(buckets))
```

---

#### Harmonic Mean

**Why not arithmetic mean?**

Arithmetic mean is skewed by outliers:
```
Buckets: [1, 1, 1, 10]
Arithmetic mean: (2^1 + 2^1 + 2^1 + 2^10) / 4 = 258
Actual: ~4

Harmonic mean: 4 / (1/2 + 1/2 + 1/2 + 1/1024) ≈ 2.67
Actual: ~4
```

**Formula:**

```
HarmonicMean(x1, x2, ..., xm) = m / (1/x1 + 1/x2 + ... + 1/xm)

For HLL:
Estimate = α_m * m * 2^(HarmonicMean(buckets))

where α_m is a bias correction constant
```

---

### HyperLogLog Refinements

#### 1. Bias Correction

```c
double alpha_m(unsigned m) {
    switch (m) {
        case 16:  return 0.673;
        case 32:  return 0.697;
        case 64:  return 0.709;
        default:  return 0.7213 / (1.0 + 1.079 / m);
    }
}
```

---

#### 2. Small/Large Range Corrections

```c
double hllCount(uint8_t *registers, int m) {
    double estimate = /* harmonic mean calculation */;

    // Small range correction
    if (estimate <= 2.5 * m) {
        int zeros = count_zeros(registers, m);
        if (zeros != 0) {
            estimate = m * log((double)m / zeros);
        }
    }

    // Large range correction (for 64-bit hashes)
    if (estimate > (1.0/30.0) * (1ULL << 32)) {
        estimate = -1 * (1ULL << 32) * log(1.0 - estimate / (1ULL << 32));
    }

    return estimate;
}
```

---

### Redis Implementation

#### Storage Formats

**Dense Representation:**

```
For m=16384 buckets, 6 bits per bucket:
Memory = 16384 * 6 / 8 = 12 KB
```

**Sparse Representation (for small cardinalities):**

```
Encode runs of zeros and individual values:
- Run of n zeros: encoded compactly
- Non-zero value: encode position + value

Switches to dense when sparse > 3KB
```

---

#### PFADD Implementation

```c
int pfadd(robj *o, robj **argv, int argc) {
    struct hllhdr *hdr = o->ptr;
    int updated = 0;

    for (int i = 0; i < argc; i++) {
        uint64_t hash = MurmurHash64A(argv[i]->ptr, sdslen(argv[i]->ptr), 0);

        // Extract bucket index (first 14 bits for 16384 buckets)
        int index = hash & 0x3fff;

        // Extract remaining 50 bits, count leading zeros + 1
        hash >>= 14;
        int count = __builtin_clzll(hash | (1ULL << 63)) + 1;

        // Update bucket if new max
        if (hllPatSet(hdr, index, count)) {
            updated = 1;
        }
    }

    return updated;
}
```

---

#### PFCOUNT Implementation

```c
uint64_t pfcount(robj *o) {
    struct hllhdr *hdr = o->ptr;
    double m = 16384.0;
    double E = 0.0;

    // Compute harmonic mean
    for (int i = 0; i < 16384; i++) {
        int val = hllPatGet(hdr, i);
        E += 1.0 / pow(2.0, val);
    }

    // Calculate estimate
    E = (1.0 / E) * m * m * 0.7213 / (1.0 + 1.079 / m);

    // Apply corrections
    if (E <= 2.5 * m) {
        int zeros = countZeros(hdr);
        if (zeros != 0) {
            E = m * log(m / (double)zeros);
        }
    } else if (E > (1.0/30.0) * pow(2, 32)) {
        E = -pow(2, 32) * log(1.0 - E / pow(2, 32));
    }

    return (uint64_t)E;
}
```

---

#### PFMERGE Implementation

```c
void pfmerge(robj *dest, robj **src, int count) {
    struct hllhdr *dest_hdr = dest->ptr;

    for (int i = 0; i < count; i++) {
        struct hllhdr *src_hdr = src[i]->ptr;

        // Merge: take maximum register value for each bucket
        for (int j = 0; j < 16384; j++) {
            int src_val = hllPatGet(src_hdr, j);
            int dest_val = hllPatGet(dest_hdr, j);

            if (src_val > dest_val) {
                hllPatSet(dest_hdr, j, src_val);
            }
        }
    }
}
```

---

### Accuracy Analysis

**Standard Error:**

```
σ = 1.04 / sqrt(m)

For m=16384:
σ = 1.04 / sqrt(16384) = 1.04 / 128 ≈ 0.81%
```

**Practical Results:**

```bash
# Test with known cardinality
for i in {1..1000000}; do
    echo "user_$i"
done | redis-cli --pipe PFADD unique_users

# Check estimate
PFCOUNT unique_users
# 1001234  (actual: 1000000, error: 0.12%)
```

**Error Distribution:**

```
Test 1000 trials:
Mean error: 0.81%
Std dev: 0.81%
99th percentile: 2.5%
```

---

### Use Cases

**1. Unique Visitors (per page)**

```bash
# Track daily visitors
PFADD visitors:homepage:2024-01-15 user1 user2 ...
PFADD visitors:about:2024-01-15 user3 user4 ...

# Get counts
PFCOUNT visitors:homepage:2024-01-15
PFCOUNT visitors:about:2024-01-15

# Total unique across pages
PFMERGE visitors:all:2024-01-15 \
    visitors:homepage:2024-01-15 \
    visitors:about:2024-01-15

PFCOUNT visitors:all:2024-01-15
```

**Memory:** 12 KB per page per day (vs GB for exact sets)

---

**2. IP Address Deduplication**

```bash
# Track unique IPs per hour
PFADD ips:2024-01-15:00 192.168.1.1 192.168.1.2 ...
PFADD ips:2024-01-15:01 192.168.1.3 192.168.1.1 ...

# Daily unique IPs
PFMERGE ips:2024-01-15 ips:2024-01-15:00 ips:2024-01-15:01 ... ips:2024-01-15:23
PFCOUNT ips:2024-01-15
```

---

### Summary

**HyperLogLog:**
- ✅ **Memory**: Fixed 12 KB (up to ~10^9 elements)
- ✅ **Accuracy**: ±0.81% standard error
- ✅ **Mergeable**: Union of multiple HLLs
- ✅ **Fast**: O(1) add, O(m) count (m=16384)

**Trade-offs:**
- ⚠️ Approximate (not exact)
- ⚠️ Insert-only (can't remove elements)
- ⚠️ Can't list elements

**When to Use:**
- ✅ Counting unique items at scale
- ✅ Memory is constrained
- ✅ Approximate counts acceptable
- ❌ Need exact counts or element listing

**Next:** LFU eviction with approximate counting

---

## Chapter 27: LFU and Approximate Counting

### LFU Eviction Strategy

**LFU (Least Frequently Used)** evicts keys accessed the fewest times, regardless of recency.

**LRU vs LFU:**

```
Scenario: Keys A, B, C

LRU Timeline:
t=0: Access A
t=1: Access B
t=2: Access C
t=3: Evict? → A (least recent)

LFU Timeline:
t=0-10: Access A (100 times)
t=11: Access B (1 time)
t=12: Access C (1 time)
t=13: Evict? → B or C (least frequent)
```

**When LFU is Better:**
- Hot keys exist (repeatedly accessed)
- Burst access patterns (LRU would evict hot key after burst)

---

### The Frequency Counting Problem

**Naive Approach:**

```go
type Object struct {
    Value       interface{}
    AccessCount int64  // 8 bytes per object!
}

func onAccess(obj *Object) {
    obj.AccessCount++  // No upper bound
}
```

**Problems:**
1. **8 bytes overhead** per object (millions of keys = MB wasted)
2. **No decay**: Old hot keys stay hot forever
3. **Overflow risk**: Counters grow indefinitely

---

### Redis LFU Design

**Uses same 24 bits as LRU:**

```c
struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:24;  // ← Reused for LFU!
    // ...
};
```

**LFU Bit Layout:**

```
24 bits total:
┌──────────────────┬──────────────┐
│ Last Decay Time  │ Log Counter  │
│    (16 bits)     │   (8 bits)   │
└──────────────────┴──────────────┘
```

**Fields:**
- **Last Decay Time (16 bits)**: Unix minutes (mod 2^16)
- **Log Counter (8 bits)**: Logarithmic access frequency

---

### Morris Counter

**Problem:** Store large counts in small space (8 bits)

**Solution:** Store log of count, not count itself

#### Algorithm

```
Instead of storing N, store V where:
    N ≈ 2^V - 1

Example:
    N=1     → V=1   (2^1-1 = 1)
    N=3     → V=2   (2^2-1 = 3)
    N=7     → V=3   (2^3-1 = 7)
    N=255   → V=8   (2^8-1 = 255)
    N=65535 → V=16  (but we only have 8 bits!)
```

---

#### Probabilistic Increment

**Challenge:** Can't increment V on every access (saturates too quickly)

**Solution:** Increment with probability inversely proportional to current count

```
Current V → Estimated N
Probability of incrementing V = 1 / (N(V+1) - N(V))

where N(V) = 2^V - 1
```

**Example:**

```
V=3, N≈7:
- N(3) = 7
- N(4) = 15
- Increment probability = 1/(15-7) = 1/8 = 12.5%

V=8, N≈255:
- N(8) = 255
- N(9) = 511
- Increment probability = 1/(511-255) = 1/256 = 0.39%
```

**Result:** Higher counts increment less frequently

---

#### Implementation

```c
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;  // Saturated

    double r = (double)rand() / RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;

    if (baseval < 0) baseval = 0;

    double p = 1.0 / (baseval * server.lfu_log_factor + 1);

    if (r < p) {
        counter++;
    }

    return counter;
}
```

**`lfu_log_factor`** controls growth rate:
- Higher factor → slower growth → more precision for small counts
- Default: 10

---

### Time-Based Decay

**Problem:** Old hot keys shouldn't stay hot forever

**Solution:** Decay counter over time

#### Last Decay Time (16 bits)

**Store Unix minutes mod 2^16:**

```c
unsigned long LFUGetTimeInMinutes(void) {
    return (server.unixtime / 60) & 65535;  // Wrap at 2^16
}
```

**Range:** 0 to 65535 minutes ≈ 45 days

---

#### Decay Logic

**On every access:**

```c
unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;  // Extract last decay time (16 bits)
    unsigned long counter = o->lru & 255;  // Extract counter (8 bits)

    unsigned long now = LFUGetTimeInMinutes();
    unsigned long elapsed;

    // Handle wraparound
    if (now >= ldt) {
        elapsed = now - ldt;
    } else {
        elapsed = 65535 - ldt + now;
    }

    // Decay: 1 decrement per decay_time minutes
    unsigned long num_periods = elapsed / server.lfu_decay_time;

    if (num_periods) {
        counter = (num_periods > counter) ? 0 : counter - num_periods;
    }

    return counter;
}
```

**Configuration:**

```bash
# redis.conf
lfu-decay-time 1  # Decay every 1 minute
```

---

#### Update on Access

```c
void updateLFU(robj *val) {
    unsigned long counter = LFUDecrAndReturn(val);  // Decay first
    counter = LFULogIncr(counter);                   // Then increment

    unsigned long now = LFUGetTimeInMinutes();
    val->lru = (now << 8) | counter;  // Reconstruct 24 bits
}
```

**Flow:**

```
Before access (t=100):
┌──────┬────────┐
│ 100  │  50    │  (last access: 100 min ago, counter: 50)
└──────┴────────┘

Access at t=105 (5 minutes later, lfu_decay_time=1):
1. Decay: 50 - 5 = 45
2. Increment: 45 → 46 (probabilistic)
3. Update time: 105

After access:
┌──────┬────────┐
│ 105  │  46    │
└──────┴────────┘
```

---

### Saturation at 1 Million

**Counter saturates at 255:**

```
V=255 → N ≈ 2^255 - 1 (huge!)

But in practice, Redis limits to:
N ≈ 1,000,000 accesses
```

**Why?** Beyond 1M, precision doesn't matter for eviction decisions.

---

### LFU Eviction Process

**Same pool-based approach as LRU:**

```c
void evictionPoolPopulate(/* ... */) {
    // Sample random keys
    for (int i = 0; i < sample_size; i++) {
        robj *o = dictGetRandomKey(dict);

        // Get idle time (inverse of frequency)
        unsigned long idle = 255 - LFUDecrAndReturn(o);

        // Add to eviction pool
        evictionPoolAdd(pool, key, idle);
    }
}
```

**Idle Time Calculation:**

```
High frequency → Low idle time → Less likely to evict
Low frequency  → High idle time → More likely to evict
```

---

### Configuration

```bash
# redis.conf

# Eviction policy
maxmemory-policy allkeys-lfu  # or volatile-lfu

# LFU tuning
lfu-log-factor 10      # Higher = slower counter growth
lfu-decay-time 1       # Minutes per decay tick
```

**Tuning Guidelines:**

| Workload | log_factor | decay_time |
|----------|-----------|------------|
| High traffic (millions hits/day) | 10 (default) | 1 (default) |
| Low traffic (thousands hits/day) | 5 | 5 |
| Long-term trending | 10 | 60 |
| Short-term bursts | 20 | 1 |

---

### Performance Analysis

**Memory:**

```
Per object:
- LRU: 24 bits (3 bytes)
- LFU: 24 bits (3 bytes)  ← Same!

No additional overhead
```

**CPU:**

```
Per access:
- Decay calculation: ~10ns
- Probabilistic increment: ~20ns (includes random())
- Total: ~30ns

Negligible compared to command execution (microseconds)
```

---

### LRU vs LFU Comparison

**Test Scenario:**

```
Keys:
- hotkey: Accessed 1000 times at t=0-10
- burst1: Accessed 100 times at t=100
- burst2: Accessed 100 times at t=101
- newkey: Accessed 1 time at t=102

Memory full, need to evict at t=103
```

**LRU Decision:**

```
Last Access Times:
- hotkey: t=10  (oldest)
- burst1: t=100
- burst2: t=101
- newkey: t=102 (newest)

Evict: hotkey (despite being most accessed overall)
```

**LFU Decision:**

```
Access Frequencies (after decay):
- hotkey: ~900 (decayed from 1000)
- burst1: ~100
- burst2: ~100
- newkey: ~1

Evict: newkey (lowest frequency)
```

**When LFU Wins:**
- Workloads with persistent hot keys
- Burst access patterns
- Long cache retention

**When LRU Wins:**
- Temporal locality (recent = future access)
- Sequential scans
- Changing access patterns

---

### Summary

**Morris Counter:**
- ✅ 8-bit counter represents ~1M accesses
- ✅ Logarithmic storage: V ≈ log₂(N)
- ✅ Probabilistic increment prevents saturation
- ✅ Higher precision for small counts

**Time Decay:**
- ✅ 16-bit timestamp (45-day cycle)
- ✅ Configurable decay rate
- ✅ Prevents stale hot keys
- ✅ On-demand decay (no background thread)

**LFU Implementation:**
- ✅ Same 24-bit overhead as LRU
- ✅ Combines frequency + time
- ✅ Handles wrap-around
- ✅ Production-grade approximate algorithm

**Design Brilliance:**
- **3 bytes** store both frequency (~1M range) and time (45 days)
- **No background threads** (decay on access)
- **Probabilistic** to avoid precision waste
- **Beautiful engineering** for constrained resources

---

**END OF COMPREHENSIVE REDIS INTERNALS GUIDE**

---

# Part 8: Redis Data Structures - User Guide and Practical Patterns

> **Purpose of Part 8:** While Parts 1-7 covered how Redis is *built* (internals, algorithms, implementation details), Part 8 focuses on how to *use* Redis effectively. This section provides a comprehensive command reference, real-world use cases, and practical patterns for interview preparation and production development.

> **Cross-Reference Note:** This part frequently references internals covered earlier:
> - **Strings** → See Chapter 25 (SDS internals)
> - **Lists** → See Chapter 22 (Ziplist/Quicklist internals)
> - **Sets** → See Chapter 23 (Intset internals)
> - **Sorted Sets** → See Chapter 24 (Geohash for geospatial), Chapter 22 (Ziplist encoding)
> - **HyperLogLog** → See Chapter 26 (Cardinality estimation algorithm)
> - **Bitmaps** → Built on Strings (Chapter 25)

---

## Chapter 28: Strings - The Foundation of Redis Data Storage

**What are Redis Strings?**

Redis Strings are the most basic data type, representing sequences of bytes that can store text, numbers, or binary data up to 512 MB. Despite being called "strings," they can efficiently handle integers and floating-point numbers with dedicated arithmetic operations.

> **Internals Reference:** See Chapter 25 for Simple Dynamic Strings (SDS) implementation details, including O(1) length operations and smart pre-allocation strategies.

**When to use Strings:**
- Caching application data and web page fragments
- Storing user session information
- Implementing counters and metrics
- Feature flags and configuration values
- Distributed locking mechanisms

### Real-world use cases

#### 1. Web application caching
```bash
# Cache user profile data with expiration
SET user:12345:profile '{"name":"John","email":"john@example.com"}' EX 3600
GET user:12345:profile
```

#### 2. Website visitor counter
```bash
# Initialize daily visitor counter
SET visitors:2024-06-19 0
# Increment on each page view
INCR visitors:2024-06-19
# Returns: (integer) 1 after first visit
```

#### 3. Distributed rate limiting
```bash
# Allow maximum 100 requests per user per minute
SET rate:user:12345:2024-06-19:10:30 1 NX EX 60
INCR rate:user:12345:2024-06-19:10:30
# Returns current request count, reject if > 100
```

### String commands deep dive

#### SET - Store string value
```bash
SET key value [EX seconds] [PX milliseconds] [NX|XX] [KEEPTTL] [GET]
```
**Parameters:**
- `key`: String key name
- `value`: String value to store
- `EX seconds`: Set expiration in seconds
- `NX`: Only set if key doesn't exist
- `XX`: Only set if key exists
- `GET`: Return previous value

**Examples:**
```bash
SET user:token "abc123"                    # Returns: OK
SET user:token "def456" GET                # Returns: "abc123"
SET session:temp "data" EX 300 NX          # Set with 5-minute expiration if not exists
```

#### GET - Retrieve string value
```bash
GET key
```
**Returns:** String value or (nil) if key doesn't exist

```bash
GET user:token                             # Returns: "def456"
GET nonexistent                           # Returns: (nil)
```

#### INCR/DECR - Atomic increment operations
```bash
INCR key                                   # Increment by 1
INCRBY key increment                       # Increment by specific amount
DECR key                                   # Decrement by 1
DECRBY key decrement                       # Decrement by specific amount
INCRBYFLOAT key increment                  # Increment float value
```

**Examples:**
```bash
SET counter 10
INCR counter                               # Returns: (integer) 11
INCRBY counter 5                          # Returns: (integer) 16
INCRBYFLOAT counter 2.5                   # Returns: "18.5"
DECR counter                              # Returns: (integer) 17 (from 18.5 truncated)
```

#### Practical CLI Example: String and Counter Operations

```redis
127.0.0.1:6379> set name Shabbir
OK
127.0.0.1:6379> get name
"Shabbir"
127.0.0.1:6379> set email email@domain.com
OK
127.0.0.1:6379> get email
"email@domain.com"
127.0.0.1:6379> getrange email 0 4
"email"
127.0.0.1:6379> mset lang English technology Redis
OK
127.0.0.1:6379> mget lang technology
1) "English"
2) "Redis"
127.0.0.1:6379> strlen lang
(integer) 7
127.0.0.1:6379> strlen technology
(integer) 5
127.0.0.1:6379> set count 1
OK
127.0.0.1:6379> get count
"1"
127.0.0.1:6379> incr count
(integer) 2
127.0.0.1:6379> incrby count 10
(integer) 12
127.0.0.1:6379> decr count
(integer) 11
127.0.0.1:6379> decrby count 5
(integer) 6
```

#### APPEND - Concatenate strings
```bash
APPEND key value
```
**Returns:** Length of string after append

```bash
SET greeting "Hello"
APPEND greeting " World"                   # Returns: (integer) 11
GET greeting                              # Returns: "Hello World"
```

#### GETRANGE - Get substring
```bash
GETRANGE key start end
```

```bash
SET mykey "Hello World"
GETRANGE mykey 0 4                        # Returns: "Hello"
GETRANGE mykey -5 -1                      # Returns: "World"
```

#### MSET/MGET - Multiple key operations
```bash
MSET key value [key value ...]
MGET key [key ...]
```

```bash
MSET key1 "value1" key2 "value2" key3 "value3"
MGET key1 key2 key3                       # Returns: ["value1", "value2", "value3"]
```

#### STRLEN - Get string length
```bash
STRLEN key
```

```bash
SET mykey "Hello"
STRLEN mykey                              # Returns: (integer) 5
```

#### EXPIRE/TTL - Set and check expiration
```bash
EXPIRE key seconds                         # Set expiration time
TTL key                                   # Get remaining time to live
SETEX key seconds value                   # Set with expiration atomically
```

**Practical CLI Example: Expiration and Float Operations**

```redis
127.0.0.1:6379> set pi 3.14
OK
127.0.0.1:6379> get pi
"3.14"
127.0.0.1:6379> incrbyfloat pi 0.0001
"3.1400999999999998"
127.0.0.1:6379> set a 1
OK
127.0.0.1:6379> get a
"1"
127.0.0.1:6379> expire a 10
(integer) 1
127.0.0.1:6379> ttl a
(integer) 4
127.0.0.1:6379> ttl a
(integer) 2
127.0.0.1:6379> ttl a
(integer) -2
127.0.0.1:6379> get a
(nil)
127.0.0.1:6379> setex b 10 anyvalue
OK
127.0.0.1:6379> get b
"anyvalue"
127.0.0.1:6379> ttl b
(integer) 2
127.0.0.1:6379> ttl b
(integer) -2
127.0.0.1:6379> get b
(nil)
```

---

## Chapter 29: Lists - Ordered Collections for Queues and Stacks

**What are Redis Lists?**

Redis Lists are ordered collections of strings sorted by insertion order. Implemented as doubly-linked lists (or quicklists combining linked lists and ziplists), they provide fast insertion and deletion at both ends, making them perfect for queues, stacks, and activity feeds.

> **Internals Reference:** See Chapter 22 for Ziplist and Quicklist implementation details, including memory-efficient encoding strategies.

**When to use Lists:**
- Message queues and task processing
- Activity feeds and timelines
- Stack-based operations (LIFO)
- Queue-based operations (FIFO)
- Chat message storage

### Real-world use cases

#### 1. Job queue system
```bash
# Producer adds jobs to queue
LPUSH jobs:email "send_welcome_email:user123"
LPUSH jobs:email "send_newsletter:batch42"

# Worker processes jobs
RPOP jobs:email                           # Returns: "send_welcome_email:user123"
```

#### 2. Activity feed timeline
```bash
# Add new activities to user's timeline
LPUSH timeline:user123 "liked_post:456"
LPUSH timeline:user123 "followed:user789"

# Get recent activities (newest first)
LRANGE timeline:user123 0 9              # Returns last 10 activities
```

#### 3. Chat message history
```bash
# Store chat messages
RPUSH chat:room101 "user1: Hello everyone!"
RPUSH chat:room101 "user2: Hi there!"

# Get message history
LRANGE chat:room101 -10 -1               # Returns last 10 messages
```

### List commands deep dive

#### LPUSH/RPUSH - Add elements to list
```bash
LPUSH key element [element ...]           # Add to head (left)
RPUSH key element [element ...]           # Add to tail (right)
```
**Returns:** Length of list after operation

```bash
RPUSH queue:tasks "task1" "task2" "task3" # Returns: (integer) 3
LPUSH queue:tasks "urgent_task"           # Returns: (integer) 4
# List order: ["urgent_task", "task1", "task2", "task3"]
```

#### LPOP/RPOP - Remove elements from list
```bash
LPOP key [count]                          # Remove from head
RPOP key [count]                          # Remove from tail
```
**Returns:** Removed element(s) or (nil) if empty

```bash
RPOP queue:tasks                          # Returns: "task3"
LPOP queue:tasks 2                        # Returns: ["urgent_task", "task1"]
```

#### Practical CLI Example: Basic List Operations

```redis
127.0.0.1:6379> lpush country India
(integer) 1
127.0.0.1:6379> lpush country USA
(integer) 2
127.0.0.1:6379> lrange country 0 -1
1) "USA"
2) "India"
127.0.0.1:6379> lpush country UK
(integer) 3
127.0.0.1:6379> lrange country 0 -1
1) "UK"
2) "USA"
3) "India"
127.0.0.1:6379> lrange country 0 1
1) "UK"
2) "USA"
127.0.0.1:6379> rpush country Australia
(integer) 4
127.0.0.1:6379> lrange country 0 -1
1) "UK"
2) "USA"
3) "India"
4) "Australia"
127.0.0.1:6379> llen country
(integer) 4
127.0.0.1:6379> lpop country
"UK"
127.0.0.1:6379> rpop country
"Australia"
127.0.0.1:6379> lrange country 0 -1
1) "USA"
2) "India"
127.0.0.1:6379> lpush country France
(integer) 3
```

#### LRANGE - Get range of elements
```bash
LRANGE key start stop
```
**Parameters:**
- `start`: Starting index (0-based, negative indices count from end)
- `stop`: Ending index (inclusive)

```bash
RPUSH mylist "a" "b" "c" "d" "e"
LRANGE mylist 0 2                         # Returns: ["a", "b", "c"]
LRANGE mylist -2 -1                       # Returns: ["d", "e"]
LRANGE mylist 0 -1                        # Returns: all elements
```

#### LINDEX - Get element by index
```bash
LINDEX key index
```

```bash
LINDEX mylist 0                           # Returns: "a" (first element)
LINDEX mylist -1                          # Returns: "e" (last element)
```

#### LLEN - Get list length
```bash
LLEN key
```

```bash
LLEN mylist                               # Returns: (integer) 5
LLEN nonexistent                          # Returns: (integer) 0
```

#### LSET - Set element at index
```bash
LSET key index value
```

```bash
LSET mylist 0 "NEW"
LRANGE mylist 0 -1                        # First element is now "NEW"
```

#### LINSERT - Insert before/after element
```bash
LINSERT key BEFORE|AFTER pivot value
```

```bash
LINSERT mylist BEFORE "b" "inserted"
LINSERT mylist AFTER "c" "another"
```

#### Practical CLI Example: Advanced List Operations

```redis
127.0.0.1:6379> lrange country 0 -1
1) "France"
2) "USA"
3) "India"
127.0.0.1:6379> lset country 0 Germany
OK
127.0.0.1:6379> lrange country 0 -1
1) "Germany"
2) "USA"
3) "India"
127.0.0.1:6379> linsert country before Germany "New Zealand"
(integer) 4
127.0.0.1:6379> lrange country 0 -1
1) "New Zealand"
2) "Germany"
3) "USA"
4) "India"
127.0.0.1:6379> linsert country after USA UAE
(integer) 5
127.0.0.1:6379> lrange country 0 -1
1) "New Zealand"
2) "Germany"
3) "USA"
4) "UAE"
5) "India"
127.0.0.1:6379> lindex country 3
"UAE"
127.0.0.1:6379> lindex country 2
"USA"
```

#### LPUSHX/RPUSHX - Push only if list exists
```bash
LPUSHX key element                        # Push to head only if key exists
RPUSHX key element                        # Push to tail only if key exists
```

```bash
LPUSHX newlist "value"                    # Returns: (integer) 0 (list doesn't exist)
LPUSH newlist "first"                     # Create the list
LPUSHX newlist "second"                   # Returns: (integer) 2 (now it works)
```

#### LTRIM - Trim list to range
```bash
LTRIM key start stop
```

```bash
LPUSH mylist "a" "b" "c" "d" "e"
LTRIM mylist 0 2                          # Keep only first 3 elements
LRANGE mylist 0 -1                        # Returns: ["e", "d", "c"]
```

#### BLPOP/BRPOP - Blocking pop operations
```bash
BLPOP key [key ...] timeout               # Blocking left pop
BRPOP key [key ...] timeout               # Blocking right pop
```

**Use case: Worker waiting for jobs**

```bash
# Worker blocks until job arrives (timeout 30 seconds)
BRPOP jobs:queue 30
# Returns: ["jobs:queue", "job_data"] when job is pushed
```

#### Practical CLI Example: Blocking Operations and Sorting

```redis
127.0.0.1:6379> blpop movies 1
(nil)
(1.07s)
127.0.0.1:6379> blpop movies 15
(nil)
(15.10s)
127.0.0.1:6379> blpop movies 15
1) "movies"
2) "abv"
(12.67s)

127.0.0.1:6379> lrange country 0 -1
1) "South Africa"
2) "New Zealand"
3) "Germany"
4) "USA"
5) "UAE"
6) "India"
127.0.0.1:6379> sort country ALPHA
1) "Germany"
2) "India"
3) "New Zealand"
4) "South Africa"
5) "UAE"
6) "USA"
127.0.0.1:6379> sort country desc ALPHA
1) "USA"
2) "UAE"
3) "South Africa"
4) "New Zealand"
5) "India"
6) "Germany"
```

---

## Chapter 30: Sets - Unique Collections for Membership and Relationships

**What are Redis Sets?**

Redis Sets are unordered collections of unique strings that support fast membership testing, intersection, union, and difference operations. They're perfect for representing relationships and implementing tag systems.

> **Internals Reference:** See Chapter 23 for Intset implementation details, including binary search and upgrade mechanisms for different integer sizes.

**When to use Sets:**
- User relationships (followers, friends)
- Tag systems and categories
- Unique visitor tracking
- Recommendation systems
- Access control lists

### Real-world use cases

#### 1. Social media followers
```bash
# User relationships
SADD followers:user123 "user456" "user789" "user101"
SADD following:user456 "user123" "user999"

# Check if user456 follows user123
SISMEMBER followers:user123 "user456"     # Returns: (integer) 1

# Get mutual followers
SINTER followers:user123 followers:user789
```

#### 2. Product tagging system
```bash
# Tag products
SADD tags:electronics "laptop:1" "phone:2" "tablet:3"
SADD tags:portable "phone:2" "tablet:3" "camera:4"

# Find portable electronics
SINTER tags:electronics tags:portable     # Returns: ["phone:2", "tablet:3"]
```

#### 3. Access control permissions
```bash
# Define user permissions
SADD perms:admin "read" "write" "delete" "admin"
SADD perms:editor "read" "write"

# Check if admin can delete
SISMEMBER perms:admin "delete"            # Returns: (integer) 1
```

### Set commands deep dive

#### SADD - Add members to set
```bash
SADD key member [member ...]
```
**Returns:** Number of new members added

```bash
SADD tags:colors "red" "blue" "green"     # Returns: (integer) 3
SADD tags:colors "red" "yellow"           # Returns: (integer) 1 (only yellow is new)
```

#### SREM - Remove members from set
```bash
SREM key member [member ...]
```
**Returns:** Number of members removed

```bash
SREM tags:colors "red" "purple"           # Returns: (integer) 1 (only red existed)
```

#### SMEMBERS - Get all set members
```bash
SMEMBERS key
```
**Returns:** Array of all members

```bash
SMEMBERS tags:colors                      # Returns: ["blue", "green", "yellow"]
```

#### SISMEMBER - Test membership
```bash
SISMEMBER key member
```
**Returns:** 1 if member exists, 0 otherwise

```bash
SISMEMBER tags:colors "blue"              # Returns: (integer) 1
SISMEMBER tags:colors "red"               # Returns: (integer) 0
```

#### SCARD - Get set cardinality
```bash
SCARD key
```

```bash
SCARD tags:colors                         # Returns: (integer) 3
```

#### Set operations: SINTER, SUNION, SDIFF
```bash
SINTER key [key ...]                      # Intersection
SUNION key [key ...]                      # Union
SDIFF key [key ...]                       # Difference
```

**Examples:**

```bash
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"

SINTER set1 set2                          # Returns: ["b", "c"]
SUNION set1 set2                          # Returns: ["a", "b", "c", "d"]
SDIFF set1 set2                           # Returns: ["a"] (in set1 but not set2)
```

#### SDIFFSTORE, SINTERSTORE, SUNIONSTORE - Store operation results
```bash
SDIFFSTORE destination key [key ...]
SINTERSTORE destination key [key ...]
SUNIONSTORE destination key [key ...]
```

```bash
SINTERSTORE result set1 set2
SMEMBERS result                           # Returns intersection stored in 'result'
```

#### Practical CLI Example: Set Operations

```redis
127.0.0.1:6379> sadd technology Java
(integer) 1
127.0.0.1:6379> sadd technology Redis AWS
(integer) 2
127.0.0.1:6379> smembers technology
1) "Redis"
2) "AWS"
3) "Java"
127.0.0.1:6379> sadd technology Java
(integer) 0
127.0.0.1:6379> scard technology
(integer) 3
127.0.0.1:6379> sismember technology Java
(integer) 1
127.0.0.1:6379> sismember technology Spring
(integer) 0
127.0.0.1:6379> sadd frontend JavaScript HTML Nodejs React
(integer) 4
127.0.0.1:6379> smembers frontend
1) "Nodejs"
2) "JavaScript"
3) "React"
4) "HTML"
127.0.0.1:6379> sdiff technology frontend
1) "Redis"
2) "Java"
3) "AWS"
127.0.0.1:6379> sdiffstore newset technology frontend
(integer) 3
127.0.0.1:6379> smembers newset
1) "Redis"
2) "Java"
3) "AWS"
127.0.0.1:6379> sinter technology frontend
1) "Nodejs"
```

#### SPOP - Remove and return random member
```bash
SPOP key [count]
```

```bash
SPOP tags:colors                          # Returns random member and removes it
SPOP tags:colors 2                        # Returns 2 random members
```

#### SRANDMEMBER - Get random member without removing
```bash
SRANDMEMBER key [count]
```

```bash
SRANDMEMBER tags:colors                   # Returns random member
SRANDMEMBER tags:colors 2                 # Returns 2 random members (may repeat)
```

#### SMOVE - Move member between sets
```bash
SMOVE source destination member
```

```bash
SMOVE set1 set2 "a"                       # Move "a" from set1 to set2
```

---

## Chapter 31: Sorted Sets - Ranked Collections with Scores

**What are Redis Sorted Sets (ZSets)?**

Sorted Sets combine the uniqueness of Sets with automatic ordering by score. Each member has an associated floating-point score that determines its position in the set. They enable efficient range queries, ranking operations, and score-based filtering.

> **Internals Reference:** See Chapter 24 for Geohash encoding (used in geospatial sorted sets) and Chapter 22 for Ziplist encoding optimization.

**When to use Sorted Sets:**
- Leaderboards and high scores
- Priority queues with scores
- Time-series data with timestamps
- Geographic coordinates with distances
- Weighted recommendation systems

### Real-world use cases

#### 1. Gaming leaderboard
```bash
# Add player scores
ZADD leaderboard 1500 "player1" 2300 "player2" 1200 "player3"

# Get top 5 players
ZREVRANGE leaderboard 0 4 WITHSCORES      # Returns highest scores first

# Get player rank
ZREVRANK leaderboard "player1"            # Returns: (integer) 1 (2nd place)
```

#### 2. Priority task queue
```bash
# Add tasks with priority scores (higher = more urgent)
ZADD tasks 1 "backup_database" 5 "fix_critical_bug" 3 "update_docs"

# Process highest priority task
ZPOPMAX tasks                             # Returns: ["fix_critical_bug", "5"]
```

#### 3. Time-series events
```bash
# Store events with timestamps as scores
ZADD events 1692629576 "user_login" 1692629580 "page_view" 1692629590 "purchase"

# Get events in time range
ZRANGEBYSCORE events 1692629575 1692629585  # Returns events between timestamps
```

### Sorted Set commands deep dive

#### ZADD - Add members with scores
```bash
ZADD key [options] score member [score member ...]
```
**Options:**
- `XX`: Only update existing members
- `NX`: Only add new members
- `CH`: Return count of changed elements
- `INCR`: Increment the score

```bash
ZADD scoreboard 100 "alice" 200 "bob"     # Returns: (integer) 2
ZADD scoreboard XX 150 "alice"            # Update alice's score to 150
ZADD scoreboard INCR 50 "bob"             # Increment bob's score by 50
```

#### Practical CLI Example: Sorted Set Operations (ZADD, ZCARD, ZREM)

```redis
127.0.0.1:6379> zadd users 1 Shabbir
(integer) 1
127.0.0.1:6379> zadd users 2 Alex 3 Nimah 4 Steve 5 Nich
(integer) 4
127.0.0.1:6379> zrange users 0 -1
1) "Shabbir"
2) "Alex"
3) "Nimah"
4) "Steve"
5) "Nich"
127.0.0.1:6379> zrange users 0 -1 withscores
1) "Shabbir"
2) "1"
3) "Alex"
4) "2"
5) "Nimah"
6) "3"
7) "Steve"
8) "4"
9) "Nich"
10) "5"
127.0.0.1:6379> zcard users
(integer) 5
127.0.0.1:6379> zcount users -inf +inf
(integer) 5
127.0.0.1:6379> zcount users 0 4
(integer) 4
127.0.0.1:6379> zrem users Alex
(integer) 1
127.0.0.1:6379> zrange users 0 -1 withscores
1) "Shabbir"
2) "1"
3) "Nimah"
4) "3"
5) "Steve"
6) "4"
7) "Nich"
8) "5"
```

#### ZRANGE - Get members by rank
```bash
ZRANGE key start stop [WITHSCORES] [REV]
```
**Parameters:**
- `start/stop`: Rank indices (0-based, negative from end)
- `WITHSCORES`: Include scores in output
- `REV`: Return in reverse order

```bash
ZRANGE scoreboard 0 -1                    # Returns: ["alice", "bob"]
ZRANGE scoreboard 0 -1 WITHSCORES         # Returns: ["alice", "150", "bob", "250"]
ZRANGE scoreboard 0 -1 REV                # Returns: ["bob", "alice"] (highest first)
```

#### ZREVRANGE - Get members in reverse order
```bash
ZREVRANGE key start stop [WITHSCORES]
```

```bash
ZREVRANGE scoreboard 0 4 WITHSCORES       # Top 5 scores (highest to lowest)
```

#### ZRANGEBYSCORE / ZREVRANGEBYSCORE - Get members by score range
```bash
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]
```

**Special score values:**
- `-inf`: Negative infinity
- `+inf`: Positive infinity
- `(value`: Exclusive (open interval)

```bash
ZRANGEBYSCORE scoreboard 100 200          # Members with scores 100-200
ZRANGEBYSCORE scoreboard "(100" "+inf"    # Members with scores > 100
ZRANGEBYSCORE scoreboard -inf 200 LIMIT 0 5  # First 5 members with score ≤ 200
```

#### Practical CLI Example: ZINCRBY and ZREVRANGE

```redis
127.0.0.1:6379> zrange users 0 -1 withscores
1) "Shabbir"
2) "1"
3) "Nimah"
4) "3"
5) "Steve"
6) "4"
7) "Nich"
8) "5"
127.0.0.1:6379> zrevrangebyscore users 5 0 withscores
1) "Nich"
2) "5"
3) "Steve"
4) "4"
5) "Nimah"
6) "3"
7) "Shabbir"
8) "1"
127.0.0.1:6379> zincrby users 2 Steve
"6"
127.0.0.1:6379> zrevrangebyscore users 6 0 withscores
1) "Steve"
2) "6"
3) "Nich"
4) "5"
5) "Nimah"
6) "3"
7) "Shabbir"
8) "1"
```

#### ZRANK/ZREVRANK - Get member rank
```bash
ZRANK key member                          # Rank (0-based, lowest score = 0)
ZREVRANK key member                       # Reverse rank (highest score = 0)
```

```bash
ZRANK scoreboard "alice"                  # Returns: (integer) 0
ZREVRANK scoreboard "alice"               # Returns: (integer) 1
```

#### ZSCORE - Get member score
```bash
ZSCORE key member
```

```bash
ZSCORE scoreboard "alice"                 # Returns: "150"
```

#### ZCARD - Get number of members
```bash
ZCARD key
```

```bash
ZCARD scoreboard                          # Returns: (integer) 2
```

#### ZCOUNT - Count members in score range
```bash
ZCOUNT key min max
```

```bash
ZCOUNT scoreboard 100 200                 # Returns: (integer) 2
ZCOUNT scoreboard -inf +inf               # Count all members
```

#### ZINCRBY - Increment member score
```bash
ZINCRBY key increment member
```

```bash
ZINCRBY scoreboard 10 "alice"             # Returns: "160"
```

#### ZREM - Remove members
```bash
ZREM key member [member ...]
```

```bash
ZREM scoreboard "alice" "bob"             # Returns: (integer) 2
```

#### ZREMRANGEBYRANK - Remove members by rank range
```bash
ZREMRANGEBYRANK key start stop
```

```bash
ZREMRANGEBYRANK scoreboard 0 4            # Remove bottom 5 members
```

#### ZREMRANGEBYSCORE - Remove members by score range
```bash
ZREMRANGEBYSCORE key min max
```

```bash
ZREMRANGEBYSCORE scoreboard 0 100         # Remove members with score 0-100
```

#### ZPOPMIN/ZPOPMAX - Remove and return members with lowest/highest scores
```bash
ZPOPMIN key [count]
ZPOPMAX key [count]
```

```bash
ZPOPMAX leaderboard                       # Remove and return highest scorer
ZPOPMIN priority_queue 5                  # Process 5 lowest priority items
```

---

## Chapter 32: Hashes - Field-Value Pairs Within Keys

**What are Redis Hashes?**

Redis Hashes are record types structured as collections of field-value pairs, similar to dictionaries or objects. They represent mappings between string field names and string values, making them ideal for representing objects and structured data.

**When to use Hashes:**
- User profiles and session data
- Product information and inventory
- Configuration settings
- Real-time analytics and counters
- Any object-like data structure

### Real-world use cases

#### 1. User profile management
```bash
# Store user profile with multiple attributes
HSET user:12345 name "John Doe" email "john@example.com" age 30 location "New York" last_login "2024-06-19"

# Quick profile retrieval
HGETALL user:12345
```

#### 2. E-commerce product catalog
```bash
# Product inventory with dynamic pricing
HSET product:bike:1 model "Deimos" brand "Ergonom" type "Enduro bikes" price 4972 stock 15

# Update price without affecting other fields
HINCRBY product:bike:1 price 100          # Increase price by $100
HINCRBY product:bike:1 stock -1           # Decrease stock by 1
```

#### 3. Real-time analytics dashboard
```bash
# Track website metrics
HINCRBY analytics:daily:2024-06-19 page_views 1
HINCRBY analytics:daily:2024-06-19 unique_visitors 1
HINCRBY analytics:daily:2024-06-19 sales_revenue 299

# Get all metrics at once
HGETALL analytics:daily:2024-06-19
```

### Hash commands deep dive

#### HSET - Set field values
```bash
HSET key field value [field value ...]
```
**Returns:** Number of new fields created

```bash
HSET user:100 name "Alice" age 25 city "SF"  # Returns: (integer) 3
HSET user:100 age 26                      # Returns: (integer) 0 (field updated)
```

#### HGET/HMGET - Get field values
```bash
HGET key field                            # Get single field
HMGET key field [field ...]               # Get multiple fields
```

```bash
HGET user:100 name                        # Returns: "Alice"
HMGET user:100 name age city              # Returns: ["Alice", "26", "SF"]
```

#### HGETALL - Get all fields and values
```bash
HGETALL key
```

```bash
HGETALL user:100
# Returns: ["name", "Alice", "age", "26", "city", "SF"]
```

#### HDEL - Delete fields
```bash
HDEL key field [field ...]
```
**Returns:** Number of fields deleted

```bash
HDEL user:100 age city                    # Returns: (integer) 2
```

#### HEXISTS - Check if field exists
```bash
HEXISTS key field
```

```bash
HEXISTS user:100 name                     # Returns: (integer) 1
HEXISTS user:100 deleted_field            # Returns: (integer) 0
```

#### HINCRBY / HINCRBYFLOAT - Increment field value
```bash
HINCRBY key field increment
HINCRBYFLOAT key field increment
```

```bash
HINCRBY user:100 login_count 1            # Returns: (integer) 1
HINCRBY user:100 points -50               # Decrement by 50
HINCRBYFLOAT product:1 price 10.99        # Returns: "210.99"
```

#### HKEYS / HVALS - Get all keys or values
```bash
HKEYS key                                 # Get all field names
HVALS key                                 # Get all values
```

```bash
HKEYS user:100                            # Returns: ["name", "city", "login_count"]
HVALS user:100                            # Returns: ["Alice", "SF", "1"]
```

#### HLEN - Get number of fields
```bash
HLEN key
```

```bash
HLEN user:100                             # Returns: (integer) 3
```

#### HSETNX - Set field only if it doesn't exist
```bash
HSETNX key field value
```

```bash
HSETNX user:100 name "Bob"                # Returns: (integer) 0 (field exists)
HSETNX user:100 phone "555-1234"          # Returns: (integer) 1 (new field)
```

#### HSTRLEN - Get length of field value
```bash
HSTRLEN key field
```

```bash
HSTRLEN user:100 name                     # Returns: (integer) 5 ("Alice")
```

---

## Chapter 33: Bitmaps - Efficient Binary Operations

**What are Redis Bitmaps?**

Redis Bitmaps are not a separate data type but bit-oriented operations on Strings. They treat strings as bit vectors, allowing efficient manipulation of individual bits. Perfect for representing boolean information across large domains with minimal memory usage.

> **Internals Reference:** Bitmaps are built on Simple Dynamic Strings (SDS) - see Chapter 25 for the underlying string implementation.

**When to use Bitmaps:**
- User activity tracking (daily/monthly active users)
- Feature flags and A/B testing
- Real-time analytics
- Event occurrence tracking
- Presence/absence monitoring

### Real-world use cases

#### 1. Daily active user tracking
```bash
# Mark user 1000 as active on June 19, 2024
SETBIT users:active:2024-06-19 1000 1

# Check if user 1000 was active
GETBIT users:active:2024-06-19 1000       # Returns: (integer) 1

# Count total active users for the day
BITCOUNT users:active:2024-06-19          # Returns: (integer) 15347
```

#### 2. Feature flag management
```bash
# Enable feature for specific users
SETBIT features:beta_ui 1000 1            # Enable for user 1000
SETBIT features:beta_ui 2500 1            # Enable for user 2500

# Check if user has feature enabled
GETBIT features:beta_ui 1000              # Returns: (integer) 1
```

#### 3. A/B testing analytics
```bash
# Track user actions across different events
SETBIT events:login:2024-06-19 1000 1
SETBIT events:purchase:2024-06-19 1000 1

# Find users who logged in AND made a purchase
BITOP AND result events:login:2024-06-19 events:purchase:2024-06-19
BITCOUNT result                           # Count users who did both
```

### Bitmap commands deep dive

#### SETBIT/GETBIT - Set and get bit values
```bash
SETBIT key offset value                   # Set bit at offset to 0 or 1
GETBIT key offset                         # Get bit value at offset
```

```bash
SETBIT user:flags 0 1                     # Returns: (integer) 0 (previous value)
GETBIT user:flags 0                       # Returns: (integer) 1
GETBIT user:flags 999                     # Returns: (integer) 0 (unset bits default to 0)
```

#### BITCOUNT - Count set bits
```bash
BITCOUNT key [start end]
```
**Parameters:**
- `start/end`: Byte range (not bit range)

```bash
BITCOUNT user:flags                       # Count all set bits
BITCOUNT user:flags 0 10                  # Count set bits in bytes 0-10
```

#### BITPOS - Find first bit with specified value
```bash
BITPOS key bit [start [end]]
```

```bash
BITPOS user:flags 1                       # Find first set bit position
BITPOS user:flags 0                       # Find first unset bit position
```

#### BITOP - Bitwise operations
```bash
BITOP operation destkey key [key ...]
```
**Operations:** AND, OR, XOR, NOT

```bash
# Find users active on both days
BITOP AND both_days active:2024-06-18 active:2024-06-19

# Find users active on either day
BITOP OR either_day active:2024-06-18 active:2024-06-19

# Find users active on first day but not second
BITOP AND temp active:2024-06-18 active:2024-06-18
BITOP NOT temp2 active:2024-06-19
BITOP AND first_not_second temp temp2
```

#### BITFIELD - Operate on multiple bit fields
```bash
BITFIELD key [GET type offset] [SET type offset value] [INCRBY type offset increment]
```

**Complex example:**

```bash
BITFIELD mykey SET u8 0 255 SET u8 8 128  # Set two 8-bit unsigned integers
BITFIELD mykey GET u8 0 GET u8 8          # Returns: [255, 128]
BITFIELD mykey INCRBY u8 0 1              # Increment first field
```

---

## Chapter 34: HyperLogLogs - Cardinality Estimation at Scale

**What are Redis HyperLogLogs?**

HyperLogLog is a probabilistic data structure that estimates cardinality (number of unique elements) using constant memory (12 KB maximum), regardless of dataset size. It provides approximate counts with 0.81% standard error, making it perfect for large-scale analytics.

> **Internals Reference:** See Chapter 26 for the Flajolet-Martin algorithm implementation, register-based estimation, and stochastic averaging details.

**When to use HyperLogLogs:**
- Unique visitor counting for web analytics
- Unique item tracking in massive datasets
- Cardinality estimation where memory efficiency is critical
- Real-time analytics requiring fast operations

### Real-world use cases

#### 1. Web analytics - unique visitor tracking
```bash
# Track unique visitors per page per day
PFADD visitors:homepage:2024-06-19 192.168.1.1 10.0.0.5 172.16.0.10
PFCOUNT visitors:homepage:2024-06-19       # Returns: (integer) 3

# Track over multiple days
PFADD visitors:homepage:2024-06-20 192.168.1.1 10.0.0.6
PFCOUNT visitors:homepage:2024-06-19 visitors:homepage:2024-06-20  # Union count
```

#### 2. Search analytics - unique query tracking
```bash
# Daily unique search queries
PFADD search_queries:2024-06-19 "redis tutorial" "database optimization"
PFCOUNT search_queries:2024-06-19         # Returns: (integer) 2
```

#### 3. IoT sensor data - unique device tracking
```bash
# Track unique IoT devices per location
PFADD devices:building_A sensor_001 sensor_042 sensor_156
PFADD devices:building_B sensor_042 sensor_200 sensor_305
PFCOUNT devices:building_A devices:building_B  # Returns: (integer) 5 (union count)
```

### HyperLogLog commands deep dive

#### PFADD - Add elements
```bash
PFADD key element [element ...]
```
**Returns:** 1 if cardinality estimate changed, 0 otherwise

```bash
PFADD unique_users user_001 user_002 user_003  # Returns: (integer) 1
PFADD unique_users user_001                  # Returns: (integer) 0 (duplicate)
```

#### PFCOUNT - Count unique elements
```bash
PFCOUNT key [key ...]
```
**Returns:** Estimated cardinality

```bash
PFCOUNT unique_users                      # Returns: (integer) 3
PFCOUNT users_today users_yesterday       # Returns union cardinality
```

#### PFMERGE - Merge HyperLogLogs
```bash
PFMERGE destkey sourcekey [sourcekey ...]
```

```bash
# Merge daily counts into weekly total
PFMERGE weekly_users daily:mon daily:tue daily:wed daily:thu daily:fri
PFCOUNT weekly_users                      # Returns weekly unique count
```

**Memory efficiency example:**

```bash
# Track 1 billion unique users using only 12 KB
# Traditional set would use ~8 GB (assuming 8 bytes per user ID)
PFADD huge_dataset user_000000001 user_000000002 ... user_1000000000
MEMORY USAGE huge_dataset                 # Returns: ~12288 bytes (12 KB)
```

---

## Chapter 35: Streams - Event Logs and Message Queues

**What are Redis Streams?**

Redis Streams are persistent, append-only log data structures that act like sophisticated message queues. They provide reliable message delivery, consumer groups, and blocking operations for real-time data processing with guaranteed delivery semantics.

**When to use Streams:**
- Event sourcing architectures
- Real-time data processing pipelines
- Message queuing with guaranteed delivery
- Log aggregation and analysis
- IoT data collection and processing

### Real-world use cases

#### 1. Event sourcing - user activity tracking
```bash
# Record user events with automatic timestamp-based IDs
XADD user_events:12345 * action "login" ip "192.168.1.100"
XADD user_events:12345 * action "page_view" page "/dashboard" duration "2.3"
XADD user_events:12345 * action "logout" session_duration "1847"

# Read all events for user
XRANGE user_events:12345 - +
```

#### 2. IoT sensor monitoring
```bash
# Temperature sensor data with location and readings
XADD sensors:temperature * device_id "temp_001" location "building_A_floor_2" temperature "23.5" humidity "45.2"

# Read last hour of data
XREVRANGE sensors:temperature + - COUNT 100
```

#### 3. Real-time notifications
```bash
# Add notification to user's stream
XADD notifications:user_789 * type "message" from "user_123" title "New Message" content "Hello there!"

# User reads notifications
XREAD COUNT 10 STREAMS notifications:user_789 0
```

### Stream commands deep dive

#### XADD - Add entry to stream
```bash
XADD key [MAXLEN ~ count] * field value [field value ...]
```
**Parameters:**
- `*`: Auto-generate unique ID (recommended)
- `MAXLEN ~ count`: Approximate stream size limit
- `field value`: One or more field-value pairs

```bash
XADD events * user_id "123" action "click" element "signup_button"
# Returns: "1692629576966-0"

# Add with stream size limit
XADD sensor_data MAXLEN ~ 1000 * temperature "25.3" humidity "48.2"
```

#### XREAD - Read entries from stream
```bash
XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] id [id ...]
```

```bash
# Read from beginning
XREAD COUNT 5 STREAMS events 0

# Read only new entries (blocking)
XREAD BLOCK 1000 STREAMS events $          # Block for 1 second

# Read from specific ID onwards
XREAD STREAMS events 1692629576966-0
```

#### XRANGE / XREVRANGE - Read range of entries
```bash
XRANGE key start end [COUNT count]
XREVRANGE key end start [COUNT count]
```

```bash
XRANGE events - +                         # Get all entries
XRANGE events 1692629576966 1692629586966 # Get entries in time range
XREVRANGE events + - COUNT 10             # Get last 10 entries (newest first)
```

#### XLEN - Get stream length
```bash
XLEN key
```

```bash
XLEN events                               # Returns: (integer) 42
```

#### XDEL - Delete entries
```bash
XDEL key id [id ...]
```

```bash
XDEL events 1692629576966-0 1692629576967-0  # Returns: (integer) 2
```

#### XTRIM - Trim stream to size
```bash
XTRIM key MAXLEN ~ count
```

```bash
XTRIM events MAXLEN ~ 1000                # Keep approximately 1000 entries
```

#### Consumer Groups for Guaranteed Delivery

Consumer groups enable multiple consumers to process messages reliably with at-least-once delivery guarantees.

#### XGROUP CREATE - Create consumer group
```bash
XGROUP CREATE stream group start-id [MKSTREAM]
```

```bash
# Create processing group starting from latest entries
XGROUP CREATE user_events processors $

# Create group from beginning
XGROUP CREATE user_events batch_processors 0
```

#### XREADGROUP - Read as consumer group member
```bash
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] STREAMS stream [stream ...] id [id ...]
```

```bash
# Consumer reads new messages
XREADGROUP GROUP processors worker1 COUNT 5 STREAMS user_events >

# Block until messages arrive
XREADGROUP GROUP processors worker1 BLOCK 5000 STREAMS user_events >
```

#### XACK - Acknowledge processed messages
```bash
XACK stream group id [id ...]
```

```bash
# Process messages, then acknowledge
XACK user_events processors 1692629576966-0 1692629576967-0
```

#### XPENDING - Check pending messages
```bash
XPENDING stream group [start end count [consumer]]
```

```bash
# See pending messages for group
XPENDING user_events processors

# See specific consumer's pending messages
XPENDING user_events processors - + 10 worker1
```

#### XCLAIM - Claim pending messages from another consumer
```bash
XCLAIM stream group consumer min-idle-time id [id ...]
```

```bash
# Claim messages idle for > 60 seconds
XCLAIM user_events processors worker2 60000 1692629576966-0
```

**Complete workflow example:**

```bash
# 1. Create stream and consumer group
XGROUP CREATE orders order_processors $ MKSTREAM

# 2. Add orders to stream
XADD orders * order_id "12345" amount "99.99" customer "alice"
XADD orders * order_id "12346" amount "149.50" customer "bob"

# 3. Consumer 1 reads messages
XREADGROUP GROUP order_processors worker1 COUNT 2 STREAMS orders >
# Returns:
# 1) "orders"
# 2) 1) 1) "1692629576966-0"
#       2) ["order_id", "12345", "amount", "99.99", "customer", "alice"]
#    2) 1) "1692629576967-0"
#       2) ["order_id", "12346", "amount", "149.50", "customer", "bob"]

# 4. Process orders and acknowledge
# ... processing logic ...
XACK orders order_processors 1692629576966-0 1692629576967-0

# 5. Check for failed messages (unacknowledged)
XPENDING orders order_processors - + 10

# 6. Claim failed messages (if worker1 crashed)
XCLAIM orders order_processors worker2 300000 1692629576966-0
```

---

## Chapter 36: Pub/Sub - Messaging Patterns

**What is Redis Pub/Sub?**

Redis Pub/Sub implements the publish/subscribe messaging paradigm where publishers send messages to channels, and subscribers receive messages from channels they're interested in. This is a fire-and-forget messaging pattern with no message persistence.

**When to use Pub/Sub:**
- Real-time notifications and alerts
- Chat applications
- Live updates and feeds
- Event broadcasting to multiple listeners
- Microservice event distribution

**Important characteristics:**
- **No persistence**: Messages not received are lost
- **No acknowledgments**: Fire-and-forget delivery
- **Pattern matching**: Subscribe to channel patterns with wildcards
- **Multiple subscribers**: Each message delivered to all subscribers

### Real-world use cases

#### 1. Real-time chat system
```bash
# Users subscribe to chat room
SUBSCRIBE chat:room:101

# User sends message to room
PUBLISH chat:room:101 "user123: Hello everyone!"
```

#### 2. Live notifications
```bash
# User subscribes to personal notifications
SUBSCRIBE notifications:user:456

# System sends notification
PUBLISH notifications:user:456 '{"type":"message","from":"user789","text":"New message"}'
```

#### 3. Microservice event broadcasting
```bash
# Services subscribe to relevant events
SUBSCRIBE events:order:created
SUBSCRIBE events:order:*

# Order service publishes event
PUBLISH events:order:created '{"order_id":"12345","amount":99.99}'
```

### Pub/Sub commands deep dive

#### SUBSCRIBE - Subscribe to channels
```bash
SUBSCRIBE channel [channel ...]
```

**Enters subscription mode** - only subscription commands available until unsubscribe.

```bash
SUBSCRIBE news updates alerts
# Receives:
# 1) "subscribe"
# 2) "news"
# 3) (integer) 1
```

#### PSUBSCRIBE - Subscribe with pattern matching
```bash
PSUBSCRIBE pattern [pattern ...]
```

**Pattern wildcards:**
- `*`: Matches any characters
- `?`: Matches single character
- `[abc]`: Matches one of the characters
- `[^a]`: Matches any character except 'a'

```bash
PSUBSCRIBE news:*                         # All news channels
PSUBSCRIBE user:*:notifications           # All user notification channels
PSUBSCRIBE events:order:*                 # All order events
```

#### PUBLISH - Publish message to channel
```bash
PUBLISH channel message
```
**Returns:** Number of subscribers that received the message

```bash
PUBLISH news "Breaking: Redis 7.0 released!"  # Returns: (integer) 3
PUBLISH alerts "Server maintenance at 2 AM"   # Returns: (integer) 0 (no subscribers)
```

#### UNSUBSCRIBE - Unsubscribe from channels
```bash
UNSUBSCRIBE [channel [channel ...]]
```

```bash
UNSUBSCRIBE news                          # Unsubscribe from specific channel
UNSUBSCRIBE                               # Unsubscribe from all channels
```

#### PUNSUBSCRIBE - Unsubscribe from patterns
```bash
PUNSUBSCRIBE [pattern [pattern ...]]
```

```bash
PUNSUBSCRIBE news:*                       # Unsubscribe from pattern
PUNSUBSCRIBE                              # Unsubscribe from all patterns
```

#### PUBSUB CHANNELS - List active channels
```bash
PUBSUB CHANNELS [pattern]
```

```bash
PUBSUB CHANNELS                           # List all active channels
PUBSUB CHANNELS news:*                    # List matching channels
```

#### PUBSUB NUMSUB - Count subscribers per channel
```bash
PUBSUB NUMSUB [channel [channel ...]]
```

```bash
PUBSUB NUMSUB news updates
# Returns: ["news", "3", "updates", "5"]
```

#### PUBSUB NUMPAT - Count pattern subscriptions
```bash
PUBSUB NUMPAT
```

```bash
PUBSUB NUMPAT                             # Returns: (integer) 7
```

### Practical CLI Example: Pub/Sub Operations

```redis
# Subscriber terminal (pattern subscription with wildcards)
127.0.0.1:6379> psubscribe news hello broadcast
Reading messages... (press ctrl-c to quit)
1) "psubscribe"
2) "news"
3) (integer) 1
1) "psubscribe"
2) "hello"
3) (integer) 2
1) "psubscribe"
2) "broadcast"
3) (integer) 3

# Receives messages when published:
1) "pmessage"
2) "news"
3) "news"
4) "New Ball"
1) "message"
2) "hello"
3) "Hello"
1) "message"
2) "broadcast"
3) "New Broadcast"

# Publisher terminal
127.0.0.1:6379> publish news "New Breaking News"
(integer) 1
127.0.0.1:6379> publish news "New News"
(integer) 1
127.0.0.1:6379> publish hello Hello
(integer) 1
127.0.0.1:6379> publish ball "New Ball"
(integer) 1
127.0.0.1:6379> publish bill "Bills"
(integer) 1
127.0.0.1:6379> publish broadcast "New Broadcast"
(integer) 1

# Introspection commands
127.0.0.1:6379> pubsub channels
1) "broadcast"
2) "news"
3) "hello"
127.0.0.1:6379> pubsub numsub news
1) "news"
2) (integer) 3
127.0.0.1:6379> pubsub numpat
(integer) 1
```

### Pub/Sub vs Streams comparison

| Feature | Pub/Sub | Streams |
|---------|---------|---------|
| **Persistence** | None (fire-and-forget) | Durable (stored on disk) |
| **Message history** | No | Yes (can read past messages) |
| **Delivery guarantees** | At-most-once | At-least-once (with consumer groups) |
| **Acknowledgments** | No | Yes (XACK) |
| **Consumer groups** | No | Yes |
| **Pattern matching** | Yes (PSUBSCRIBE) | No |
| **Use case** | Real-time broadcasts, live updates | Event sourcing, reliable queuing |
| **Memory** | No overhead | Stores all messages until trimmed |

**When to choose Pub/Sub:**
- Real-time notifications where lost messages are acceptable
- Broadcasting to multiple listeners simultaneously
- Low latency more important than reliability
- No need for message history

**When to choose Streams:**
- Need message persistence and history
- Require delivery acknowledgments
- Multiple consumers processing messages (consumer groups)
- Event sourcing and audit trails

---

## Chapter 37: Lua Scripting and Programmability

**What is Lua Scripting in Redis?**

Redis supports executing Lua scripts on the server side, enabling atomic execution of complex operations that would otherwise require multiple round trips. Scripts have access to all Redis commands and execute atomically without interruption.

**When to use Lua scripting:**
- Atomic multi-step operations (compare-and-swap patterns)
- Complex business logic requiring multiple Redis operations
- Reducing network round trips
- Implementing custom commands
- Rate limiting and quota management

**Key characteristics:**
- **Atomicity**: Entire script executes as a single atomic operation
- **No race conditions**: Scripts cannot be interrupted
- **Server-side execution**: Reduces network latency
- **Caching**: Scripts can be pre-loaded and executed by SHA hash

### Real-world use cases

#### 1. Atomic compare-and-swap
```lua
-- Only update if current value matches expected
local current = redis.call('GET', KEYS[1])
if current == ARGV[1] then
    redis.call('SET', KEYS[1], ARGV[2])
    return 1
else
    return 0
end
```

```bash
EVAL "local current = redis.call('GET', KEYS[1]); if current == ARGV[1] then redis.call('SET', KEYS[1], ARGV[2]); return 1 else return 0 end" 1 mykey "expected" "new_value"
```

#### 2. Rate limiting with sliding window
```lua
-- Implement sliding window rate limiter
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Remove old entries outside window
redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

-- Count current requests
local current = redis.call('ZCARD', key)

if current < limit then
    -- Add new request
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window)
    return 1
else
    return 0
end
```

#### 3. Conditional increment with max value
```lua
-- Increment only if value doesn't exceed max
local current = tonumber(redis.call('GET', KEYS[1]) or 0)
local max = tonumber(ARGV[1])
local increment = tonumber(ARGV[2])

if current + increment <= max then
    return redis.call('INCRBY', KEYS[1], increment)
else
    return current
end
```

### Lua scripting commands deep dive

#### EVAL - Execute Lua script
```bash
EVAL script numkeys key [key ...] arg [arg ...]
```

**Parameters:**
- `script`: Lua script code
- `numkeys`: Number of keys (separates KEYS from ARGV)
- `key`: Keys accessed by script (available as KEYS table)
- `arg`: Arguments to script (available as ARGV table)

**Simple example:**

```bash
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey "Hello"
# Returns: OK
```

**Multiple operations:**

```bash
EVAL "redis.call('SET', KEYS[1], ARGV[1]); redis.call('INCR', KEYS[2]); return redis.call('GET', KEYS[2])" 2 name counter "Alice"
# Sets name="Alice", increments counter, returns counter value
```

#### Practical CLI Example: Lua Scripting with EVAL

```redis
127.0.0.1:6379> eval "redis.call('set', KEYS[1], ARGV[1])" 1 name Shabbir
(nil)
127.0.0.1:6379> get name
"Shabbir"

127.0.0.1:6379> eval "redis.call('mset', KEYS[1], ARGV[1], KEYS[2], ARGV[2])" 2 name last_name Shabbir Dawoodi
(nil)
127.0.0.1:6379> get last_name
"Dawoodi"
```

#### Advanced Example: Combining Multiple Data Structures

```redis
# Create data
127.0.0.1:6379> hmset country_cap India "New Delhi" USA "Washington, D.C." Russia Moscow Germany Berlin Japan Tokyo Italy Rome
OK
127.0.0.1:6379> zadd country 1 Italy 2 India 3 USA
(integer) 3
127.0.0.1:6379> zrange country 0 -1
1) "Italy"
2) "India"
3) "USA"

# Use Lua to get sorted set order and lookup hash values
127.0.0.1:6379> eval "local order = redis.call('zrange', KEYS[1], 0, -1); return redis.call('hmget', KEYS[2], unpack(order));" 2 country country_cap
1) "Rome"
2) "New Delhi"
3) "Washington, D.C."
```

**Explanation:** This script atomically:
1. Gets ordered country names from sorted set
2. Uses those names to lookup capitals from hash
3. Returns capitals in sorted set order

#### EVALSHA - Execute cached script by SHA hash
```bash
EVALSHA sha1 numkeys key [key ...] arg [arg ...]
```

**Benefits:**
- Reduces bandwidth (send hash instead of full script)
- Faster execution (script already parsed)
- Script reuse across multiple calls

**Workflow:**

```bash
# 1. Load script and get SHA hash
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# Returns: "a42059b356c875f0717db19a51f6aaca9ae659ea"

# 2. Execute by SHA (much more efficient)
EVALSHA a42059b356c875f0717db19a51f6aaca9ae659ea 1 mykey
# Returns: value of mykey
```

#### Practical CLI Example: Script Caching

```redis
127.0.0.1:6379> script load "local order = redis.call('zrange', KEYS[1], 0, -1); return redis.call('hmget', KEYS[2], unpack(order));"
"030395796a98abf5fa2af083ba1680aae53efd6"

127.0.0.1:6379> evalsha 030395796a98abf5fa2af083ba1680aae53efd6 2 country country_cap
1) "Rome"
2) "New Delhi"
3) "Washington, D.C."

127.0.0.1:6379> script exists 030395796a98abf5fa2af083ba1680aae53efd6
1) (integer) 1

127.0.0.1:6379> script flush
OK

127.0.0.1:6379> script exists 030395796a98abf5fa2af083ba1680aae53efd6
1) (integer) 0
```

#### SCRIPT LOAD - Pre-load script without executing
```bash
SCRIPT LOAD script
```

```bash
SCRIPT LOAD "return redis.call('INCR', KEYS[1])"
# Returns SHA: "e0e1f9fabfc9d4800c877a703b823ac0578ff8db"
```

#### SCRIPT EXISTS - Check if scripts are cached
```bash
SCRIPT EXISTS sha1 [sha1 ...]
```

```bash
SCRIPT EXISTS e0e1f9fabfc9d4800c877a703b823ac0578ff8db a42059b356c875f0717db19a51f6aaca9ae659ea
# Returns: [(integer) 1, (integer) 0]
```

#### SCRIPT FLUSH - Remove all cached scripts
```bash
SCRIPT FLUSH [ASYNC|SYNC]
```

```bash
SCRIPT FLUSH                              # Clear all cached scripts
```

#### SCRIPT KILL - Kill currently running script
```bash
SCRIPT KILL
```

**Use case:** Kill a long-running script that hasn't modified data

```bash
SCRIPT KILL
# Returns: OK (if script hasn't written data)
# Returns: Error (if script has written data - must use SHUTDOWN NOSAVE)
```

### Lua scripting best practices

#### 1. Always use KEYS and ARGV properly

```lua
-- CORRECT: Keys in KEYS, values in ARGV
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey myvalue

-- WRONG: Hardcoded keys prevent cluster compatibility
EVAL "return redis.call('SET', 'mykey', ARGV[1])" 0 myvalue
```

#### 2. Return meaningful values

```lua
-- Return operation result
local result = redis.call('INCR', KEYS[1])
if result > 100 then
    redis.call('SET', KEYS[2], 'threshold_exceeded')
    return {result, 'exceeded'}
else
    return {result, 'ok'}
end
```

#### 3. Handle nil values carefully

```lua
local value = redis.call('GET', KEYS[1])
if not value then
    value = 0  -- Default value
end
return tonumber(value) + tonumber(ARGV[1])
```

#### 4. Use local variables

```lua
-- GOOD: Use local variables
local key = KEYS[1]
local increment = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or 0)
return redis.call('SET', key, current + increment)

-- AVOID: Repeated table lookups
return redis.call('SET', KEYS[1], tonumber(redis.call('GET', KEYS[1]) or 0) + tonumber(ARGV[1]))
```

#### 5. Keep scripts short and focused

```lua
-- GOOD: Single responsibility
-- Atomic counter with max value
local current = tonumber(redis.call('GET', KEYS[1]) or 0)
local max = tonumber(ARGV[1])
if current < max then
    return redis.call('INCR', KEYS[1])
else
    return max
end

-- AVOID: Complex multi-purpose scripts that could be split
```

### Common Lua scripting patterns

#### Pattern 1: Distributed lock with timeout

```lua
-- Acquire lock only if not already held
local lock_key = KEYS[1]
local lock_value = ARGV[1]
local ttl = tonumber(ARGV[2])

local result = redis.call('SET', lock_key, lock_value, 'NX', 'EX', ttl)
if result then
    return 1
else
    return 0
end
```

#### Pattern 2: Leaky bucket rate limiter

```lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local leak_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Get last update time and current level
local last_update = tonumber(redis.call('HGET', key, 'last_update') or now)
local level = tonumber(redis.call('HGET', key, 'level') or 0)

-- Calculate leaked amount
local elapsed = now - last_update
local leaked = elapsed * leak_rate
level = math.max(0, level - leaked)

-- Try to add request
if level < capacity then
    level = level + 1
    redis.call('HMSET', key, 'level', level, 'last_update', now)
    redis.call('EXPIRE', key, 3600)
    return 1
else
    return 0
end
```

#### Pattern 3: Conditional list append

```lua
-- Append to list only if total size doesn't exceed max
local list_key = KEYS[1]
local max_size = tonumber(ARGV[1])
local value = ARGV[2]

local current_size = redis.call('LLEN', list_key)
if current_size < max_size then
    redis.call('RPUSH', list_key, value)
    return current_size + 1
else
    return -1  -- Indicate failure
end
```

---

## Chapter 38: Geospatial Commands - Location-Based Queries

**What are Redis Geospatial commands?**

Redis provides specialized commands for storing and querying geographic coordinates using Sorted Sets with Geohash encoding. These commands enable efficient proximity searches, distance calculations, and location-based services.

> **Internals Reference:** See Chapter 24 for Geohash algorithm details, including 2D coordinate encoding, bit interleaving, and spatial indexing.

**When to use Geospatial:**
- Location-based services (find nearby places)
- Delivery and ride-sharing applications
- Store/restaurant locators
- Geo-fencing and proximity alerts
- Real-time tracking systems

### Real-world use cases

#### 1. Restaurant finder
```bash
# Add restaurants with coordinates
GEOADD restaurants 77.5946 12.9716 "Bangalore_Restaurant" 72.8777 19.0760 "Mumbai_Cafe"

# Find restaurants within 500km of user
GEORADIUS restaurants 77.5 13.0 500 km WITHDIST
```

#### 2. Ride-sharing driver matching
```bash
# Add available drivers
GEOADD drivers 77.6001 12.9345 "driver:123" 77.6100 12.9400 "driver:456"

# Find drivers within 2km of passenger
GEORADIUS drivers 77.6050 12.9370 2 km WITHCOORD WITHDIST
```

#### 3. Store locator
```bash
# Add store locations
GEOADD stores -122.4194 37.7749 "SF_Store" -118.2437 34.0522 "LA_Store"

# Get 5 nearest stores to user location
GEORADIUS stores -122.4000 37.7800 50 km COUNT 5 ASC
```

### Geospatial commands deep dive

#### GEOADD - Add locations
```bash
GEOADD key longitude latitude member [longitude latitude member ...]
```

**Important:** Longitude comes before latitude (X before Y)

**Valid ranges:**
- Longitude: -180 to 180 degrees
- Latitude: -85.05112878 to 85.05112878 degrees

```bash
GEOADD cities 77.5946 12.9716 "Bangalore" 72.8777 19.0760 "Mumbai"
# Returns: (integer) 2
```

#### Practical CLI Example: Geospatial Operations

```redis
127.0.0.1:6379> GEOADD maps 72.585022 23.033863 Ahmedabad
(integer) 1
127.0.0.1:6379> GEOADD maps 72.877426 19.076090 Mumbai 77.594562 12.971420 Bangalore
(integer) 2
127.0.0.1:6379> zrange maps 0 -1
1) "Bangalore"
2) "Mumbai"
3) "Ahmedabad"

127.0.0.1:6379> GEOHASH maps Ahmedabad
1) "ts59nvcK8"

127.0.0.1:6379> GEOPOS maps Ahmedabad
1) 1) "72.58502233864136"
   2) "23.033864180583168"

127.0.0.1:6379> GEOADD maps 73.856255 18.516726 Pune
(integer) 1
127.0.0.1:6379> zrange maps 0 -1
1) "Bangalore"
2) "Mumbai"
3) "Pune"
4) "Ahmedabad"

127.0.0.1:6379> GEODIST maps Mumbai Pune
"120837.5912"
127.0.0.1:6379> GEODIST maps Mumbai Pune km
"120.8376"
127.0.0.1:6379> GEODIST maps Mumbai Pune mi
"75.0860"
127.0.0.1:6379> GEODIST maps Mumbai Ahmedabad km
"441.2531"
```

#### GEOPOS - Get coordinates of locations
```bash
GEOPOS key member [member ...]
```

```bash
GEOPOS cities "Bangalore" "Mumbai"
# Returns:
# 1) 1) "77.59465605020523071"
#    2) "12.97159894623566846"
# 2) 1) "72.87770152092576027"
#    2) "19.07599971824207629"
```

#### GEODIST - Calculate distance between locations
```bash
GEODIST key member1 member2 [unit]
```

**Units:** `m` (meters), `km` (kilometers), `mi` (miles), `ft` (feet)

```bash
GEODIST cities "Bangalore" "Mumbai" km
# Returns: "842.2891"

GEODIST cities "Bangalore" "Mumbai" mi
# Returns: "523.4218"
```

#### GEORADIUS - Find locations within radius
```bash
GEORADIUS key longitude latitude radius unit [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]
```

**Options:**
- `WITHCOORD`: Include coordinates in results
- `WITHDIST`: Include distance from center
- `WITHHASH`: Include geohash integer
- `COUNT`: Limit results
- `ASC|DESC`: Sort by distance
- `STORE`: Store result locations in another key
- `STOREDIST`: Store result distances in sorted set

```bash
# Find cities within 500km of coordinates
GEORADIUS cities 77.0 13.0 500 km WITHDIST WITHCOORD ASC
# Returns:
# 1) 1) "Bangalore"
#    2) "68.8947"
#    3) 1) "77.59465605020523071"
#       2) "12.97159894623566846"
```

```bash
# Find 3 nearest cities
GEORADIUS cities 77.0 13.0 1000 km COUNT 3 ASC
```

#### GEORADIUSBYMEMBER - Find locations within radius of a member
```bash
GEORADIUSBYMEMBER key member radius unit [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]
```

**Use case:** Find locations near a known member

```bash
# Find cities within 500km of Bangalore
GEORADIUSBYMEMBER cities "Bangalore" 500 km WITHDIST
# Returns:
# 1) 1) "Bangalore"
#    2) "0.0000"
# 2) 1) "Hyderabad"
#    2) "498.2341"
```

#### GEOHASH - Get geohash string representation
```bash
GEOHASH key member [member ...]
```

**Returns:** Base32 geohash string (11 characters)

```bash
GEOHASH cities "Bangalore" "Mumbai"
# Returns:
# 1) "tdnu2scrn00"
# 2) "te7t96hppz0"
```

**Note:** Geohashes with similar prefixes are geographically close.

#### GEOSEARCH / GEOSEARCHSTORE - Modern radius search (Redis 6.2+)

```bash
GEOSEARCH key FROMMEMBER member | FROMLONLAT longitude latitude BYRADIUS radius unit | BYBOX width height unit [ASC|DESC] [COUNT count] [WITHCOORD] [WITHDIST] [WITHHASH]
```

**More flexible than GEORADIUS:**

```bash
# Search by radius from member
GEOSEARCH cities FROMMEMBER "Bangalore" BYRADIUS 500 km WITHDIST

# Search by bounding box
GEOSEARCH cities FROMLONLAT 77.0 13.0 BYBOX 400 400 km ASC COUNT 5

# Store results
GEOSEARCHSTORE nearby_cities cities FROMLONLAT 77.0 13.0 BYRADIUS 300 km
```

### Geospatial implementation details

**Under the hood:**
- Geospatial data stored in Sorted Sets
- Scores are 52-bit geohash integers
- Standard sorted set commands work on geo keys

```bash
GEOADD cities 77.5946 12.9716 "Bangalore"

# Equivalent to:
ZADD cities 3481342659657022 "Bangalore"

# You can use sorted set commands:
ZRANGE cities 0 -1                        # List all cities
ZREM cities "Bangalore"                   # Remove city
ZSCORE cities "Mumbai"                    # Get geohash score
```

**Memory efficiency:**
- Each location uses ~40 bytes (member string + 8-byte score)
- Efficient for millions of locations
- No additional indexing structures needed

### Best practices for geospatial queries

#### 1. Choose appropriate radius

```bash
# TOO LARGE: May return thousands of results
GEORADIUS cities 77.0 13.0 5000 km        # Half of Earth

# BETTER: Reasonable search radius
GEORADIUS cities 77.0 13.0 50 km COUNT 10 # Nearby locations
```

#### 2. Use COUNT to limit results

```bash
GEORADIUS cities 77.0 13.0 100 km COUNT 5 ASC  # Get 5 nearest only
```

#### 3. Store and reuse search results

```bash
# Store search results for reuse
GEORADIUS cities 77.0 13.0 100 km STORE search_result

# Use stored results
ZRANGE search_result 0 -1
```

#### 4. Combine with other Redis data structures

```bash
# Get nearby drivers and check availability
GEORADIUS drivers 77.6 12.9 5 km WITHCOORD | while read driver; do
    HGET driver_status "$driver"
done
```

#### 5. Clean up old locations periodically

```bash
# Remove inactive locations using sorted set commands
ZREM cities "OldCity1" "OldCity2"
```

---

## Chapter 39: Performance Considerations and Best Practices

This chapter synthesizes best practices for using Redis data structures efficiently in production systems, focusing on performance optimization, memory management, and design patterns.

### Memory optimization strategies

#### String optimization

**Best practices:**
- Use appropriate data types (prefer INCR over SET for counters)
- Set expiration times for temporary data
- Use compression for large text values
- Consider shared objects for frequently repeated values

```bash
# GOOD: Use INCR for counters
INCR page_views:home

# AVOID: GET, increment, SET (3 operations, not atomic)
GET page_views:home
# ... increment in application ...
SET page_views:home 1001

# GOOD: Set TTL for session data
SETEX session:abc123 3600 "user_data"

# AVOID: Separate SET and EXPIRE
SET session:abc123 "user_data"
EXPIRE session:abc123 3600
```

**Memory impact:**

| Approach | Memory per key | Example |
|----------|---------------|---------|
| String with JSON | ~100-500 bytes | `SET user:1 '{"name":"Alice","age":30}'` |
| Hash | ~50-100 bytes | `HMSET user:1 name Alice age 30` |
| Compressed string | ~30-150 bytes | Use application-level compression |

#### List optimization

**Best practices:**
- Trim lists regularly with LTRIM to control memory usage
- Use blocking operations (BLPOP/BRPOP) for real-time processing
- Consider Redis Streams for complex message queuing
- Monitor list length and set maximum sizes

```bash
# GOOD: Maintain fixed-size activity feed
LPUSH timeline:user:123 "new_activity"
LTRIM timeline:user:123 0 99              # Keep latest 100 items

# GOOD: Worker with blocking pop (no busy-waiting)
BRPOP jobs:queue 30                       # Block for 30 seconds

# AVOID: Unbounded list growth
LPUSH logs:all "log_entry"                # Can grow indefinitely
```

**List encoding optimization:**

Redis automatically uses ziplist encoding for small lists:
- `list-max-ziplist-size` (default: -2, ~8KB per node)
- `list-compress-depth` (default: 0, no compression)

```bash
# Check encoding
OBJECT ENCODING mylist                    # Returns: "quicklist" or "ziplist"

# Memory usage
MEMORY USAGE mylist                       # Returns bytes used
```

#### Set and Sorted Set optimization

**Best practices:**
- Monitor set cardinality - very large sets impact performance
- Use appropriate operations (SSCAN for large set iteration)
- Consider data partitioning for massive sorted sets
- Use ZPOPMIN/ZPOPMAX for priority queue patterns

```bash
# GOOD: Iterate large sets with cursor
SSCAN large_set 0 COUNT 100

# AVOID: SMEMBERS on large sets
SMEMBERS large_set                        # Blocks Redis for large sets

# GOOD: Partition large sorted sets
ZADD leaderboard:shard:0 100 "player1"
ZADD leaderboard:shard:1 200 "player2"

# GOOD: Priority queue pattern
ZADD tasks 1 "low_priority" 5 "high_priority"
ZPOPMAX tasks                             # Always get highest priority
```

**Sorted set encoding:**

- Uses ziplist for small sorted sets (configurable)
- `zset-max-ziplist-entries` (default: 128)
- `zset-max-ziplist-value` (default: 64 bytes)

#### Hash optimization

**Best practices:**
- Keep individual hashes small (< 100 fields) for memory optimization
- Use consistent field naming to reduce overhead
- Prefer HMGET over multiple HGET calls
- Consider hash partitioning for large objects

```bash
# GOOD: Batch field retrieval
HMGET user:123 name email age

# AVOID: Multiple round trips
HGET user:123 name
HGET user:123 email
HGET user:123 age

# GOOD: Partition large hash
HSET user:123:profile name "Alice" email "alice@example.com"
HSET user:123:stats login_count 42 posts_count 156

# AVOID: Single massive hash
HSET user:123 field1 val1 field2 val2 ... field1000 val1000
```

**Hash encoding optimization:**

- `hash-max-ziplist-entries` (default: 512)
- `hash-max-ziplist-value` (default: 64 bytes)

**Memory comparison:**

```bash
# 1000 small hashes (100 fields each)
# Uses ziplist encoding: ~50KB per hash = 50MB total

# 1 large hash (100,000 fields)
# Uses hash table encoding: ~100MB total

# Conclusion: Prefer multiple small hashes over one massive hash
```

### Choosing the right data structure

#### Decision flowchart

```
Need to store data?
├─ Simple value (text, number, binary)
│  └─ Use STRING
├─ Ordered collection?
│  ├─ Need unique elements?
│  │  ├─ Need scoring/ranking?
│  │  │  └─ Use SORTED SET (leaderboards, time-series)
│  │  └─ No scoring needed
│  │     └─ Use SET (tags, relationships)
│  └─ Allow duplicates?
│     └─ Use LIST (queues, activity feeds)
├─ Key-value pairs (object-like)?
│  └─ Use HASH (user profiles, products)
├─ Track yes/no for many items?
│  └─ Use BITMAP (user activity, feature flags)
├─ Count unique items (huge scale)?
│  └─ Use HYPERLOGLOG (unique visitors)
└─ Event log with guaranteed delivery?
   └─ Use STREAM (event sourcing, message queues)
```

#### Use case to data structure mapping

| Use Case | Primary Structure | Alternative | Reason |
|----------|------------------|-------------|--------|
| **User session** | Hash or String | - | Hash for structured data, String for serialized |
| **Page view counter** | String (INCR) | - | Atomic increments, minimal memory |
| **Job queue** | List | Stream | List for simple FIFO, Stream for reliability |
| **Leaderboard** | Sorted Set | - | Score-based ranking, efficient range queries |
| **User followers** | Set | - | Unique members, set operations (intersection) |
| **Activity feed** | List | Stream | List for recent items, Stream for history |
| **Rate limiting** | String or Sorted Set | Lua script | String for fixed window, ZSet for sliding window |
| **Caching** | String | Hash | String for simple values, Hash for objects |
| **Daily active users** | Bitmap | HyperLogLog | Bitmap for exact count, HLL for estimates |
| **Real-time notifications** | Pub/Sub | Stream | Pub/Sub for broadcast, Stream for persistence |
| **Geospatial search** | Sorted Set (Geo) | - | Built-in geo commands, efficient radius search |
| **Unique visitor count** | HyperLogLog | Set | HLL for large scale (constant memory) |

### System design patterns

#### Pattern 1: Cache-aside (Lazy loading)

```bash
# Application pseudocode:
function getUser(userId) {
    # Try cache first
    cached = HGETALL user:{userId}
    if cached:
        return cached

    # Cache miss: load from database
    userData = database.query("SELECT * FROM users WHERE id = ?", userId)

    # Populate cache with TTL
    HMSET user:{userId} userData
    EXPIRE user:{userId} 3600

    return userData
}
```

**Advantages:**
- Only requested data is cached
- Cache failures don't break application

**Disadvantages:**
- Initial request is slow (cache miss)
- Potential cache stampede on popular items

#### Pattern 2: Write-through cache

```bash
# Application pseudocode:
function updateUser(userId, updates) {
    # Update database first
    database.update("UPDATE users SET ... WHERE id = ?", userId, updates)

    # Update cache atomically
    HMSET user:{userId} updates
    EXPIRE user:{userId} 3600

    return success
}
```

**Advantages:**
- Cache always consistent with database
- No stale data

**Disadvantages:**
- Write latency includes cache update
- May cache data that's never read

#### Pattern 3: Rate limiting with sliding window (Sorted Set)

```bash
# Sliding window rate limiter
function isAllowed(userId, limit, window) {
    now = currentTimestamp()
    key = "rate_limit:{userId}"

    # Remove old entries outside window
    ZREMRANGEBYSCORE key 0 (now - window)

    # Count requests in current window
    count = ZCARD key

    if count < limit:
        # Add current request
        ZADD key now now
        EXPIRE key window
        return true
    else:
        return false
}
```

**Example usage:**

```bash
# Allow 100 requests per hour (3600 seconds)
isAllowed("user:123", 100, 3600)
```

#### Pattern 4: Distributed locking

```bash
# Acquire lock with unique token
SET lock:resource unique_token NX EX 30

# Do work...

# Release lock only if we own it (Lua script)
EVAL """
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
else
    return 0
end
""" 1 lock:resource unique_token
```

**Important:** Always use unique token to prevent releasing someone else's lock.

#### Pattern 5: Leaderboard with user rank

```bash
# Add/update player score
ZADD leaderboard 1500 "player:123"

# Get top 10
ZREVRANGE leaderboard 0 9 WITHSCORES

# Get player rank (0-based)
ZREVRANK leaderboard "player:123"

# Get players around user (-10 to +10 ranks)
rank = ZREVRANK leaderboard "player:123"
ZREVRANGE leaderboard (rank - 10) (rank + 10) WITHSCORES
```

#### Pattern 6: Session management

```bash
# Create session with auto-expiration
HSET session:{token} user_id 123 created_at 1692629576 role "admin"
EXPIRE session:{token} 3600

# Extend session on activity
EXPIRE session:{token} 3600

# Get session data
HGETALL session:{token}

# Destroy session
DEL session:{token}
```

#### Pattern 7: Pub/Sub with Streams for reliability

```bash
# Publisher: Add event to stream
XADD events * type "order_created" order_id "12345"

# Consumer group: Guaranteed delivery
XGROUP CREATE events processors $ MKSTREAM
XREADGROUP GROUP processors worker1 COUNT 10 STREAMS events >

# Process messages...

# Acknowledge processed messages
XACK events processors 1692629576966-0
```

**Why Streams over Pub/Sub for reliability:**
- Message persistence (survives crashes)
- Consumer groups (multiple workers)
- Acknowledgments (at-least-once delivery)
- Message history (replay events)

### Performance tuning tips

#### 1. Pipeline commands to reduce round trips

```bash
# BAD: 1000 round trips
for i in range(1000):
    SET key{i} value{i}

# GOOD: 1 round trip with pipelining
pipe = redis.pipeline()
for i in range(1000):
    pipe.set(f"key{i}", f"value{i}")
pipe.execute()
```

> **Internals Reference:** See Chapter 12 for command pipelining implementation details.

#### 2. Use Lua scripting for complex atomic operations

```bash
# BAD: Multiple round trips, race conditions possible
count = GET counter
if count < 100:
    INCR counter

# GOOD: Atomic with Lua script
EVAL "local c = tonumber(redis.call('GET', KEYS[1]) or 0); if c < tonumber(ARGV[1]) then return redis.call('INCR', KEYS[1]) else return c end" 1 counter 100
```

#### 3. Monitor slow commands

```bash
# Configure slow log
CONFIG SET slowlog-log-slower-than 10000  # 10ms threshold
CONFIG SET slowlog-max-len 128

# View slow commands
SLOWLOG GET 10
```

#### 4. Use appropriate eviction policies

```bash
# For cache use cases
CONFIG SET maxmemory-policy allkeys-lru

# For mixed workload (cache + persistent data)
CONFIG SET maxmemory-policy volatile-lru

# For least frequently used
CONFIG SET maxmemory-policy allkeys-lfu
```

> **Internals Reference:** See Chapter 16-17 for LRU algorithm details and Chapter 27 for LFU implementation.

#### 5. Monitor memory usage

```bash
# Overall memory stats
INFO memory

# Per-key memory usage
MEMORY USAGE mykey

# Find large keys
redis-cli --bigkeys

# Memory doctor
MEMORY DOCTOR
```

#### 6. Use connection pooling

```python
# BAD: Create connection per request
def handle_request():
    r = redis.Redis(host='localhost')
    r.get('key')
    r.close()

# GOOD: Connection pool
pool = redis.ConnectionPool(host='localhost', max_connections=10)
r = redis.Redis(connection_pool=pool)

def handle_request():
    r.get('key')  # Reuses connection from pool
```

### Common anti-patterns to avoid

#### ❌ Anti-pattern 1: Using Redis as primary database without persistence

```bash
# WRONG: Treating Redis as disk-based database
# Data loss on restart if AOF/RDB not configured
```

**Fix:** Enable AOF or RDB persistence for critical data.

#### ❌ Anti-pattern 2: Storing large values in strings

```bash
# WRONG: 100MB video file in string
SET video:123 [100MB binary data]
```

**Fix:** Store large files in object storage (S3), store references in Redis.

#### ❌ Anti-pattern 3: Using KEYS in production

```bash
# WRONG: Blocks Redis server
KEYS user:*  # Scans entire keyspace
```

**Fix:** Use SCAN for safe iteration:

```bash
SCAN 0 MATCH user:* COUNT 100
```

#### ❌ Anti-pattern 4: Unbounded list/set growth

```bash
# WRONG: No size limits
LPUSH logs:all "log entry"  # Grows forever
```

**Fix:** Use LTRIM or switch to Streams with MAXLEN:

```bash
LPUSH logs:all "log entry"
LTRIM logs:all 0 9999  # Keep latest 10,000
```

#### ❌ Anti-pattern 5: Not handling connection failures

```python
# WRONG: No retry logic
r.get('key')  # Crashes on network error
```

**Fix:** Use retry with exponential backoff and circuit breaker.

---

### Summary: Data Structure Selection Guide

**Quick reference for interviews:**

| Scenario | Data Structure | Key Commands |
|----------|---------------|--------------|
| User session | Hash | HSET, HGETALL, EXPIRE |
| Counter | String | INCR, INCRBY, GET |
| Recent items (feed) | List | LPUSH, LRANGE, LTRIM |
| Unique tags | Set | SADD, SMEMBERS, SINTER |
| Leaderboard | Sorted Set | ZADD, ZREVRANGE, ZRANK |
| Job queue | List or Stream | BRPOP (List), XADD/XREADGROUP (Stream) |
| Cache | String or Hash | SET/GET (String), HSET/HGET (Hash) |
| Rate limiting | String or Sorted Set | INCR+EXPIRE (fixed), ZADD+ZREMRANGEBYSCORE (sliding) |
| Feature flags | Bitmap or Set | SETBIT/GETBIT (Bitmap), SADD/SISMEMBER (Set) |
| Unique visitors (millions) | HyperLogLog | PFADD, PFCOUNT |
| Geo-location | Sorted Set (Geo) | GEOADD, GEORADIUS |
| Real-time events | Pub/Sub or Stream | PUBLISH/SUBSCRIBE (Pub/Sub), XADD/XREAD (Stream) |
| Multi-step atomic ops | Lua Script | EVAL, EVALSHA |

**Performance characteristics:**

| Operation | Time Complexity | Notes |
|-----------|----------------|-------|
| GET/SET | O(1) | Constant time |
| HGET/HSET | O(1) | Hash field access |
| LPUSH/RPUSH | O(1) | List head/tail |
| SADD/SREM | O(1) | Set operations |
| ZADD | O(log N) | Sorted set insert |
| ZRANGE | O(log N + M) | M = elements returned |
| LRANGE | O(S + N) | S = offset, N = elements |
| SMEMBERS | O(N) | Returns all members |
| GEORADIUS | O(N + log M) | N = items checked, M = results |
| PFADD | O(1) | HyperLogLog add |
| PFCOUNT | O(1) | HyperLogLog count |

---

**This completes Part 8: Redis Data Structures - User Guide and Practical Patterns**

You now have a comprehensive understanding of both:
- **How Redis is built** (Parts 1-7: Internals, algorithms, implementation)
- **How to use Redis** (Part 8: Commands, patterns, best practices)

This unified guide provides everything needed for:
- ✅ **System design interviews** (architecture, trade-offs, internals)
- ✅ **Coding interviews** (commands, data structures, patterns)
- ✅ **Production development** (best practices, performance optimization)
- ✅ **Interview preparation** (comprehensive single resource)

---

---

## Chapter 40: Probabilistic Data Structures at Scale

**What are Probabilistic Data Structures?**

Probabilistic data structures trade perfect accuracy for dramatic improvements in memory efficiency and performance. Redis 8 natively integrates these structures, enabling applications to handle petabyte-scale analytics with minimal resources. **Companies like Netflix, Google, and Meta achieve 10-1000x performance improvements** by accepting 1-5% error rates in exchange for massive resource savings.

> **Key Insight:** These structures are not experimental - they power production systems processing trillions of operations daily at the world's largest tech companies.

**When to use Probabilistic Data Structures:**
- Massive-scale analytics (billions of unique elements)
- Real-time stream processing
- Memory-constrained environments
- High-throughput applications where approximations are acceptable
- Distributed systems requiring mergeable data structures

### Mathematical foundations and error guarantees

Probabilistic structures provide **rigorous mathematical guarantees** about error bounds:

- **HyperLogLog:** 0.81% standard error, estimates up to 2^64 unique elements
- **Bloom Filters:** Configurable false positive rates (typical: 0.01-1%), zero false negatives
- **Count-Min Sketch:** Error bounded by ε × total_count with probability 1-δ
- **Cuckoo Filters:** Similar to Bloom but supports deletions
- **Top-K:** 99%+ accuracy for true heavy hitters
- **t-digest:** Parts-per-million accuracy for extreme percentiles

---

## HyperLogLog: Cardinality Estimation Champion

> **Internals Reference:** See Chapter 26 for detailed HyperLogLog algorithm implementation using the Flajolet-Martin approach.

**Core principle:** Analyzes bit patterns in hash values to estimate set size. If you see a hash starting with k leading zeros, approximately 2^k unique values have been observed.

### Memory efficiency breakthrough

**Fixed 12KB memory** regardless of dataset size:
- Counts thousands of elements: 12KB
- Counts billions of elements: 12KB
- Counts trillions of elements: 12KB

**Contrast with exact counting:**
- 1 billion unique user IDs (8 bytes each) = 8GB
- HyperLogLog for same dataset = 12KB
- **Memory reduction: 99.9985%**

### Commands and usage

```bash
# Add elements (O(1) per element)
PFADD unique_visitors alice bob carol dave
# Returns: 1 (cardinality estimate changed)

# Get cardinality estimate (O(1) with caching)
PFCOUNT unique_visitors
# Returns: 4 (approximate count)

# Union operation across multiple HyperLogLogs
PFCOUNT visitors_site1 visitors_site2 visitors_site3
# Returns union cardinality

# Merge HyperLogLogs (O(N) where N = number of registers)
PFMERGE all_visitors site1_visitors site2_visitors
PFCOUNT all_visitors
```

### Production use cases

#### 1. Massive-scale unique visitor counting
```bash
# Track unique visitors per page per day
PFADD visitors:homepage:2024-11-13 user:12345 user:67890
PFADD visitors:products:2024-11-13 user:12345 user:54321

# Get daily unique visitors across entire site
PFCOUNT visitors:homepage:2024-11-13 visitors:products:2024-11-13
# Returns: 3 (union of unique visitors)

# Weekly aggregation
PFMERGE visitors:weekly:2024-W46 \
  visitors:homepage:2024-11-11 \
  visitors:homepage:2024-11-12 \
  visitors:homepage:2024-11-13
```

#### 2. Google BigQuery implementation
**Real-world results:**
- Dataset: 3 billion Reddit comments
- HyperLogLog time: 5.7 seconds
- Exact counting time: 28 seconds
- **Speedup: 5x with 0.2% error rate**
- Memory: 32KB vs 166MB (99.98% reduction)

#### 3. Meta's Presto integration
**APPROX_DISTINCT function:**
- Queries that took 12+ hours → Complete in minutes
- Memory: Under 1MB regardless of dataset size
- **Performance improvement: 7x to 1000x depending on query**

### Accuracy characteristics

**Standard error: 0.81%**

Practical interpretation:
- True cardinality: 1,000,000
- 99% of estimates fall within: 980,000 - 1,020,000 (±2%)
- Median error: ~0.5%

**Optimization strategies:**
- **Sparse encoding:** Efficient for small cardinalities (< 256 elements)
- **Dense encoding:** Optimal for large datasets
- **Automatic transition:** Redis switches between encodings transparently
- **Caching:** Last cardinality cached in 8-byte suffix (sub-millisecond response)

### Mergeability for distributed systems

```bash
# Server 1: Track users in region A
PFADD users:region_a user1 user2 user3

# Server 2: Track users in region B
PFADD users:region_b user3 user4 user5

# Central aggregation: Merge for global count
PFMERGE users:global users:region_a users:region_b
PFCOUNT users:global
# Returns: 5 (correctly handles overlap)
```

**Key property:** Merging produces mathematically identical result to building single HyperLogLog from combined dataset.

---

## Bloom Filters: Probabilistic Membership Testing

**Core principle:** Answer "Have I seen this element?" with **zero false negatives** but configurable false positive rates.

**Trade-off:**
- Element definitely NOT in set → 100% accurate
- Element possibly IN set → May be false positive

### Memory efficiency formula

```
Memory (bits) = capacity × (-ln(error_rate) / ln(2)²)
```

**Practical examples:**
- 1 million elements, 1% error rate = 1.2MB (9.6 bits per element)
- 1 million elements, 0.1% error rate = 1.8MB (14.4 bits per element)
- **Compare to exact Set:** 1 million strings (avg 20 bytes) = 20MB+

**Memory savings: 90-95% compared to exact sets**

### Commands and usage

#### BF.RESERVE - Create Bloom filter
```bash
BF.RESERVE user_emails 0.01 1000000 EXPANSION 2
# Error rate: 0.01 (1%)
# Capacity: 1 million elements
# Expansion: 2 (auto-scale factor when full)
```

**Parameters:**
- `error_rate`: False positive probability (0.001 to 0.1 typical)
- `capacity`: Expected number of elements
- `EXPANSION`: Growth multiplier for scalable filters (optional)
- `NONSCALING`: Disable auto-scaling (optional)

#### BF.ADD / BF.MADD - Insert elements
```bash
# Single addition
BF.ADD user_emails "alice@example.com"
# Returns: 1 (element added, may already exist)

# Multiple additions (batch operation)
BF.MADD user_emails "bob@example.com" "carol@example.com" "dave@example.com"
# Returns: [1, 1, 1]
```

#### BF.EXISTS / BF.MEXISTS - Test membership
```bash
# Check single element
BF.EXISTS user_emails "alice@example.com"
# Returns: 1 (possibly exists)

BF.EXISTS user_emails "unknown@example.com"
# Returns: 0 (definitely doesn't exist)

# Check multiple elements
BF.MEXISTS user_emails "alice@example.com" "bob@example.com" "xyz@test.com"
# Returns: [1, 1, 0]
```

### Production use cases

#### 1. Cache warming optimization
```bash
# Problem: Avoid caching "one-hit wonders"
# Solution: Cache only on second request

# First request
BF.EXISTS content:seen "article:12345"
# Returns: 0 (first time)
# Mark as seen but DON'T cache yet
BF.ADD content:seen "article:12345"

# Second request
BF.EXISTS content:seen "article:12345"
# Returns: 1 (seen before)
# NOW cache the content
```

**Akamai's results:**
- **75% reduction in cache storage** across 325,000 servers
- Finding: 75% of content accessed only once
- Bloom filter tracks access patterns with minimal memory

#### 2. Malicious URL filtering (Google Chrome)
**Previous approach:**
- 20MB database of malicious URLs
- Downloaded to every client
- Memory and bandwidth intensive

**Bloom filter approach:**
- 3.59MB filter with 0.0001% false positive rate
- **82% memory reduction**
- Downloads significantly faster
- False positives verified with server check

#### 3. Apache Cassandra SSTable optimization
```bash
# Problem: Disk I/O expensive for non-existent keys
# Solution: Bloom filter in memory per SSTable

# Query: "Does user:12345 exist in SSTable?"
# Bloom filter check: O(1) memory operation
# If returns 0: Skip disk read (98% of queries)
# If returns 1: Perform disk read (2% includes false positives)
```

**Production metrics:**
- False positive rate: 0.84% in production
- **98% disk I/O reduction**
- Processing billions of operations daily

### False positive rate behavior

**Error rate increases with saturation:**

| Filter Saturation | Actual False Positive Rate |
|------------------|---------------------------|
| 0-50% | ≈ Target rate (1%) |
| 50-70% | 1-2% |
| 70-85% | 2-5% |
| 85-95% | 5-15% |
| >95% | Exponential growth |

**Auto-scaling solution:**
```bash
# Scalable Bloom filter creates layers automatically
BF.RESERVE scalable_filter 0.01 100000 EXPANSION 2

# When 50% full: Creates new layer with 2x capacity
# Query checks all layers (slightly slower, maintains accuracy)
# Trade-off: Constant accuracy vs slightly increased latency
```

### Optimal parameter selection

**Hash function count (k):**
```
k = (m/n) × ln(2)
```

**Bit array size (m):**
```
m = -n × ln(p) / (ln(2))²
```

Where:
- n = expected element count
- p = desired false positive rate
- m = bit array size
- k = number of hash functions

**Practical guidelines:**
- 1% error rate → 7 hash functions, 9.6 bits/element
- 0.1% error rate → 10 hash functions, 14.4 bits/element
- Lower error rate = More hash functions = Slower operations

---

## Cuckoo Filters: Membership with Deletion Support

**Key advantage over Bloom filters:** Supports element deletion while maintaining similar space efficiency.

**Implementation:** Uses cuckoo hashing with fingerprints instead of bit arrays.

### Commands and usage

```bash
# Create Cuckoo filter
CF.RESERVE active_sessions 1000000 BUCKETSIZE 4

# Add elements
CF.ADD active_sessions "session:abc123"
# Returns: 1

# Add only if not exists
CF.ADDNX active_sessions "session:abc123"
# Returns: 0 (already exists)

# Delete elements (key difference from Bloom filters)
CF.DEL active_sessions "session:abc123"
# Returns: 1 (deleted successfully)

# Count occurrences
CF.COUNT active_sessions "session:abc123"
# Returns: 0 (after deletion)
```

### Memory trade-off

**Cuckoo vs Bloom comparison:**
- Cuckoo filter: ~32% more memory than Bloom
- Benefit: Deletion support
- Use case: When elements must be removed dynamically

**Example: Session management**
```bash
# User logs in
CF.ADD active_sessions "user:12345:session:xyz"

# User logs out
CF.DEL active_sessions "user:12345:session:xyz"

# Check if session active
CF.EXISTS active_sessions "user:12345:session:xyz"
```

### Performance characteristics

**Load factor:** Up to 95% occupancy before performance degrades

**Time complexity:**
- Insert: O(1) average case
- Lookup: O(1) average case (max 2 memory locations)
- Delete: O(1) average case

**Warning:** Deletions may cause rare false negatives if fingerprint collisions occur.

---

## Count-Min Sketch: Frequency Estimation

**Core principle:** Estimate element frequencies in data streams with bounded error guarantees.

**Key characteristic:** Never underestimates (only overestimates due to hash collisions).

### Algorithm structure

**2D array of counters:**
- Width (w): Affects accuracy
- Depth (d): Affects confidence

**Update operation:** Increment counters in each row
**Query operation:** Return minimum across rows (reduces collision impact)

### Commands and usage

#### Create by probability
```bash
# Error rate: 0.001 (0.1%)
# Confidence: 0.998 (99.8%)
CMS.INITBYPROB page_views 0.001 0.002

# Redis calculates dimensions automatically:
# Width = ceil(e / error_rate)
# Depth = ceil(ln(1 / probability))
```

#### Create by dimensions
```bash
# Direct control over memory usage
CMS.INITBYDIM request_counts 2000 5
# Width: 2000
# Depth: 5
# Memory: 2000 × 5 × 4 bytes = 40KB
```

#### Update and query
```bash
# Increment frequencies (batch operation)
CMS.INCRBY page_views "/home" 1 "/products" 5 "/about" 2 "/contact" 1
# Returns: [1, 5, 2, 1] (estimated frequencies)

# Query frequencies
CMS.QUERY page_views "/products" "/home"
# Returns: [5, 1]

# Get structure info
CMS.INFO page_views
# Returns: width, depth, total count
```

#### Merge sketches
```bash
# Distributed counting with local aggregation
CMS.MERGE combined_counts 2 morning_counts evening_counts
CMS.QUERY combined_counts "/home"
# Returns: accurate union frequency
```

### Error bounds and accuracy

**Mathematical guarantee:**
```
With probability ≥ (1-δ), error ≤ ε × ||f||₁
```

Where:
- δ = failure probability (1 - confidence)
- ε = error rate
- ||f||₁ = total count across all elements

**Practical interpretation:**
- Configuration: ε=0.001, δ=0.002
- Total count: 1 million
- Error threshold: 0.001 × 1,000,000 = 1,000
- With 99.8% confidence, estimates within ±1,000 of true frequency

**Important:** High-frequency elements have proportionally lower relative error.

### Production use cases

#### 1. Heavy hitter detection (Twitter trending)
```bash
# Track hashtag frequencies in real-time
CMS.INCRBY trending_hashtags #redis 1 #database 1 #ai 3 #ml 2

# Threshold for "trending": 0.1% of total tweets
# Only hashtags above threshold are reliable
# Noise below threshold filtered out automatically
```

**Twitter's implementation:**
- Processes millions of tweets per second
- Identifies trending topics in real-time
- Focuses on high-frequency elements (where accuracy is best)

#### 2. Rate limiting by IP address
```bash
# Track request counts per IP
CMS.INCRBY request_counts "192.168.1.100" 1

# Check if over limit
CMS.QUERY request_counts "192.168.1.100"
# If result > 1000: Rate limit triggered
```

### Dimension selection guide

**Width (w):**
```
w = ceil(e / ε)
```
- ε=0.01 (1% error) → w = 272
- ε=0.001 (0.1% error) → w = 2718

**Depth (d):**
```
d = ceil(ln(1/δ))
```
- δ=0.01 (99% confidence) → d = 5
- δ=0.001 (99.9% confidence) → d = 7

**Memory usage:**
```
Memory = w × d × 4 bytes
```
- Example: w=2000, d=5 → 40KB

---

## Top-K: Heavy Hitters Detection

**Algorithm:** HeavyKeeper with exponential decay

**Purpose:** Maintain the K most frequent elements in data streams.

### Commands and usage

```bash
# Create Top-K structure
TOPK.RESERVE trending_topics 10 2000 7 0.925
# K: 10 (track top 10 items)
# Width: 2000
# Depth: 7
# Decay: 0.925 (prevents sticky old items)

# Add elements
TOPK.ADD trending_topics #redis #python #javascript #go #rust
# Returns: [null, null, null, null, null] (no items expelled)

# Query membership in top-K
TOPK.QUERY trending_topics #redis #cobol
# Returns: [1, 0] (#redis in top-K, #cobol not)

# Get current top-K list
TOPK.LIST trending_topics
# Returns: ["#redis", "#python", "#javascript", ...]

# Get frequency estimates
TOPK.COUNT trending_topics #redis #python
# Returns: [1247, 856]
```

### Exponential decay mechanism

**Purpose:** Prevent old popular items from dominating rankings forever.

**Decay constant:** 0.9 to 0.925 typical

**Behavior:**
- Recent popularity weighted more heavily
- Old popularity fades exponentially
- Adapts to changing trends automatically

### Performance vs Sorted Sets

**Memory comparison:**
- Sorted Set (1M elements): ~50MB
- Top-K (K=100, tracking 1M): ~2.5MB
- **Memory reduction: 95%**

**Throughput:**
- Top-K: 100-150K ops/sec
- Sorted Set: 30-50K ops/sec
- **Throughput improvement: 3x**

**Trade-off:** Approximate rankings vs exact rankings

### Netflix production use case

**Rollup Pipeline system:**
- Processes 2 trillion messages daily
- Real-time counter aggregation
- Heavy hitters detection across millions of events
- Eventual consistency with accurate top-K tracking

---

## t-digest: Percentile Estimation

**Purpose:** Accurate percentile and quantile estimation from streaming data.

**Key feature:** Exceptional accuracy for extreme percentiles (P95, P99, P99.9).

### Algorithm approach

**Adaptive histogram clustering:**
- Higher precision at distribution extremes (0th and 100th percentiles)
- Lower precision in the middle
- Optimal for performance monitoring and SLA management

### Commands and usage

```bash
# Create t-digest
TDIGEST.CREATE response_times COMPRESSION 100
# Compression: Higher = better accuracy, more memory

# Add measurements
TDIGEST.ADD response_times 23.5 45.2 12.8 156.9 89.3 234.7

# Get percentiles
TDIGEST.QUANTILE response_times 0.5 0.95 0.99 0.999
# Returns: ["45.2", "214.5", "232.1", "234.5"]
# P50 (median), P95, P99, P99.9

# Cumulative distribution function
TDIGEST.CDF response_times 50.0 100.0 200.0
# Returns: [0.6, 0.85, 0.96]
# 60%, 85%, 96% of values below thresholds

# Trimmed mean (exclude outliers)
TDIGEST.TRIMMED_MEAN response_times 0.1 0.9
# Returns: "67.8"
# Mean excluding bottom 10% and top 10%

# Merge t-digests
TDIGEST.MERGE combined_latencies 2 server1_latencies server2_latencies
```

### Accuracy characteristics

**Compression parameter impact:**
- Compression=100: Parts-per-million accuracy for extreme percentiles
- Compression=200: Even better accuracy, 2x memory
- Compression=50: Lower accuracy, half memory

**Memory usage:**
```
Memory ≈ compression × 12 bytes
```
- Compression=100 → ~1.2KB
- Independent of data volume

### Production use cases

#### 1. Latency monitoring (P50, P95, P99)
```bash
# Application response time tracking
TDIGEST.CREATE api_latency COMPRESSION 100

# Record each request latency
TDIGEST.ADD api_latency 12.3 45.6 23.1 156.2 ...

# SLA monitoring: P95 < 200ms, P99 < 500ms
TDIGEST.QUANTILE api_latency 0.95 0.99
# Returns: ["187.4", "456.8"]
# SLA: Met ✓
```

#### 2. Anomaly detection
```bash
# Identify outliers beyond normal range
TDIGEST.CDF response_times 1000.0
# Returns: 0.998
# 99.8% of requests faster than 1000ms
# Requests >1000ms are anomalies (0.2%)
```

#### 3. Distributed percentile computation
```bash
# Each server maintains local t-digest
# Central aggregation merges them
TDIGEST.MERGE global_latencies 10 \
  server1_latencies \
  server2_latencies \
  ... \
  server10_latencies

# Global percentiles across distributed system
TDIGEST.QUANTILE global_latencies 0.50 0.95 0.99
```

---

## Performance Comparison and Decision Guide

### Memory usage comparison

| Structure | Memory per Element | Fixed Memory | Best For |
|-----------|-------------------|--------------|----------|
| HyperLogLog | 0.01 bits | 12KB max | Cardinality (unique counting) |
| Bloom Filter | 10-15 bits | Scales with capacity | Membership (seen before?) |
| Cuckoo Filter | 12-20 bits | Scales with capacity | Membership + deletion |
| Count-Min Sketch | 0.1-1 bits | width × depth × 4B | Frequency (how many times?) |
| Top-K | Variable | K × log(K) × 32 bits | Heavy hitters (top N most frequent) |
| t-digest | Variable | compression × 12B | Percentiles (P50, P95, P99) |

### Throughput benchmarks (ops/second)

| Structure | Throughput | Latency |
|-----------|-----------|---------|
| HyperLogLog | 1M+ ops/sec | Sub-millisecond |
| Bloom Filter | 150-200K ops/sec | 118ns |
| Cuckoo Filter | 120-180K ops/sec | 200-300ns |
| Count-Min Sketch | 180-250K ops/sec | 150ns |
| Top-K | 100-150K ops/sec | 300-500ns |
| t-digest | 80-120K ops/sec | 400-600ns |

### Decision framework

#### Use HyperLogLog when:
- ✅ Need to count unique elements
- ✅ Dataset size unknown or massive (billions+)
- ✅ Exact count not required (0.81% error acceptable)
- ✅ Fixed memory budget (12KB)
- ✅ Need to merge counts from distributed systems

**Examples:** Unique visitors, distinct users, unique IPs, unique products viewed

#### Use Bloom Filter when:
- ✅ Need to test membership ("have I seen this?")
- ✅ False positives acceptable, false negatives NOT acceptable
- ✅ Elements never removed
- ✅ Memory constrained (vs exact set)

**Examples:** Cache warming, duplicate detection, malicious URL filtering, spam detection

#### Use Cuckoo Filter when:
- ✅ Need membership testing WITH deletion support
- ✅ Can accept 32% more memory than Bloom filter
- ✅ Dynamic element set (add and remove)

**Examples:** Session management, token validation, dynamic security filtering

#### Use Count-Min Sketch when:
- ✅ Need frequency estimation in streams
- ✅ Focus on high-frequency elements (heavy hitters)
- ✅ Overestimation acceptable, underestimation NOT acceptable
- ✅ Need bounded error guarantees

**Examples:** Trending topics, rate limiting, heavy hitter detection, abuse detection

#### Use Top-K when:
- ✅ Need to track K most frequent elements
- ✅ Want automatic ranking maintenance
- ✅ Need exponential decay (recent > old)
- ✅ Memory constrained vs full sorted set

**Examples:** Trending hashtags, popular products, top users, hot keys

#### Use t-digest when:
- ✅ Need accurate percentile estimates (P50, P95, P99)
- ✅ Stream processing of numeric values
- ✅ SLA monitoring and performance tracking
- ✅ Need to merge percentiles from distributed systems

**Examples:** Latency monitoring, performance analytics, anomaly detection

### Configuration recommendations

#### Memory-constrained environments
**Priority order:**
1. **HyperLogLog** (0.01 bits/element, 12KB max) - Most efficient
2. **Count-Min Sketch** (0.1-1 bits/element) - Very efficient
3. **Bloom Filter** (10-15 bits/element) - Efficient

#### Accuracy-critical applications
- **Bloom Filter:** 0.001% error rate (20 bits/element)
- **HyperLogLog:** 16K registers for 0.5% error (80KB)
- **Count-Min Sketch:** ε=0.0001, δ=0.001 (very high precision)

#### High-throughput requirements
- **Optimize:** Use batch operations (MADD, INCRBY)
- **Parallelize:** Multiple independent structures
- **Hash functions:** MurmurHash (800% faster than crypto hashes)

---

## Real-World Production Examples

### Google BigTable: Bloom filter optimization

**Problem:** Minimize disk I/O for non-existent row-column pairs

**Solution:** Bloom filter per SSTable

**2024 improvements:**
- **Hybrid Bloom filters:** 4x utilization increase
- **CPU overhead reduction:** 60-70% through local caching
- **Prefetching optimization:** 50% cost reduction
- Processing exabyte-scale data globally

### Meta Presto: HyperLogLog for analytics

**APPROX_DISTINCT function:**
- **Speedup:** 7x to 1,000x depending on dataset
- **Memory:** <1MB vs terabytes for exact counting
- **Query time:** 12+ hours → minutes
- Processing petabyte-scale datasets

**Sparse layout optimization:**
- Exact counts for ≤256 elements
- Switch to dense at 8KB
- Automatic optimization transparent to users

### Netflix Keystone: Stream processing at scale

**Statistics:**
- **2 trillion messages daily**
- **3PB incoming, 7PB outgoing data**
- **2000+ streaming use cases**

**Architecture:**
- Apache Flink with probabilistic structures
- Real-time personalization
- A/B testing infrastructure
- Operational monitoring

### Akamai CDN: Cache optimization

**Discovery:** 75% of cached content = "one-hit wonders"

**Bloom filter solution:**
- Track content access patterns
- Cache only on second request
- **Result: 75% cache storage reduction** across 325,000 servers in 135 countries

---

## Best Practices and Operational Guidelines

### Monitoring and observability

**Key metrics to track:**
```bash
# Bloom filter saturation
BF.INFO user_emails
# Monitor: size, num_filters (layers), expansion_rate

# HyperLogLog accuracy validation
# Compare sample exact counts with estimates
# Alert if error exceeds theoretical bounds

# Count-Min Sketch counter overflow
CMS.INFO page_views
# Monitor: total_count, max counter value
```

**Performance indicators:**
- Command latency percentiles (P50, P95, P99)
- Memory usage growth rate
- Error rate vs theoretical expectations
- Hot key detection

### Security considerations

**Recent 2024 research:** 10 novel attacks on Redis probabilistic structures

**Threats:**
- **Hash collision attacks:** Degrade performance
- **False positive amplification:** Increase error rates maliciously
- **Bloom filter poisoning:** Force false positives for specific elements

**Mitigations:**
- Input validation and sanitization
- Rate limiting on structure operations
- Monitor for error rate anomalies
- Network security (Redis AUTH, TLS encryption)
- Redis ACLs for command-level restrictions

### Distributed systems and merging

**HyperLogLog merging:**
```bash
# Distributedcardinality estimation
PFMERGE global_users region1_users region2_users region3_users
```

**Bloom filter unions:**
```bash
# Bitwise OR operation
# Requires same parameters (size, hash functions)
# Result equivalent to single filter with all elements
```

**Count-Min Sketch merging:**
```bash
CMS.MERGE global_frequencies local1 local2 local3
# Element-wise matrix addition
# Error bounds maintained
```

### Fallback and error handling

**Graceful degradation patterns:**

```python
def get_cardinality(key):
    try:
        # Try HyperLogLog first
        return redis.pfcount(key)
    except RedisError:
        # Fallback to exact counting (slower, accurate)
        return len(redis.smembers(key))
```

**Circuit breaker for accuracy:**
```python
# Dual-write for validation
redis.pfadd('unique_visitors_hll', user_id)
redis.sadd('unique_visitors_exact', user_id)

# Periodic accuracy check
hll_count = redis.pfcount('unique_visitors_hll')
exact_count = redis.scard('unique_visitors_exact')
error_rate = abs(hll_count - exact_count) / exact_count

if error_rate > 0.05:  # 5% threshold
    alert("HyperLogLog accuracy degraded")
```

---

## Integration with Redis 8 Native Features

### I/O threading benefits

**Redis 8 multi-threaded I/O:**
```bash
# Enable I/O threading (redis.conf)
io-threads 4
io-threads-do-reads yes
```

**Performance impact on probabilistic structures:**
- **Throughput increase:** Up to 112% with 8 threads
- **Latency reduction:** 87% improvement
- **Concurrency:** Better handling of parallel operations

### ACL configuration

```bash
# Create role for analytics team (read-only probabilistic)
ACL SETUSER analytics_user on >password ~analytics:* +@read +@bloom +@cms +@topk +@tdigest

# Create role for data ingestion (write-only)
ACL SETUSER ingest_user on >password ~stream:* +pfadd +bf.add +cms.incrby

# Restrict dangerous operations
ACL SETUSER limited_user on >password ~data:* +@all -@dangerous -script
```

### Persistence considerations

**RDB snapshots:**
- All probabilistic structures serialized efficiently
- Size typically <5% additional overhead
- Fast reload on restart

**AOF replication:**
- Commands replayed correctly
- Merging operations preserved
- Clustering supported with hash tags

**Backup strategies:**
```bash
# HyperLogLog backup/restore via GET/SET
GET unique_visitors_backup
SET unique_visitors_restore <serialized_data>

# RedisBloom structures use SCANDUMP/LOADCHUNK
BF.SCANDUMP user_emails 0
```

---

## Future Trends and Emerging Applications

### AI and ML integration

**Feature stores:**
- HyperLogLog for feature cardinality
- Count-Min Sketch for feature frequency
- Real-time model serving with low latency

**Online learning:**
- Stream processing with probabilistic structures
- Continuous model updates
- Efficient feature extraction

### Edge computing and IoT

**Constrained environments:**
- Fixed memory budgets
- Limited bandwidth
- Hierarchical aggregation (edge → cloud)

**Sensor data analytics:**
- Real-time anomaly detection (t-digest)
- Unique device tracking (HyperLogLog)
- Frequency-based alerts (Count-Min Sketch)

### Network security and fraud detection

**DDoS detection:**
- Top-K for identifying attack sources
- Count-Min Sketch for request rate analysis
- Real-time threshold alerting

**Fraud pattern detection:**
- Bloom filters for blacklist checking
- HyperLogLog for unique device fingerprinting
- Behavioral anomaly detection

---

## Summary: Probabilistic Structures Decision Matrix

### Quick Reference Table

| Use Case | Structure | Memory | Accuracy | Speed |
|----------|-----------|--------|----------|-------|
| Unique counting (billions) | HyperLogLog | 12KB fixed | 0.81% error | 1M+ ops/sec |
| Seen before? (no deletion) | Bloom Filter | 10 bits/elem | 1% false pos | 200K ops/sec |
| Seen before? (with deletion) | Cuckoo Filter | 15 bits/elem | 2% false pos | 150K ops/sec |
| How many times? | Count-Min Sketch | 1 bit/elem | ε×N overest | 250K ops/sec |
| Top N most frequent | Top-K | K×log(K)×32b | 99% for top | 150K ops/sec |
| P50/P95/P99 latency | t-digest | C×12 bytes | PPM accuracy | 100K ops/sec |

### Key Takeaways

**When to use probabilistic structures:**
✅ Massive datasets (billions of elements)
✅ Memory constraints critical
✅ Approximate answers acceptable (bounded error)
✅ Real-time requirements (constant time operations)
✅ Distributed systems (mergeable structures)

**When NOT to use probabilistic structures:**
❌ Small datasets (< 10K elements) where exact sets fit in memory
❌ Perfect accuracy required (financial transactions, legal records)
❌ Debugging or troubleshooting (need exact values)
❌ Regulatory compliance requiring audit trails

**Production readiness:**
- ✅ Battle-tested at Google, Meta, Netflix, Twitter
- ✅ Mathematical guarantees on error bounds
- ✅ Native Redis 8 integration (no modules)
- ✅ Excellent performance characteristics
- ✅ Mature tooling and monitoring

**The bottom line:** Probabilistic data structures enable previously impossible analytics at scale. By accepting 1-5% error, you gain 10-1000x performance improvements and 90-99% memory reduction. This trade-off powers the infrastructure of the world's largest tech companies processing trillions of operations daily.

---
---

## Chapter 41: Java Integration with Jedis - Production Patterns

**What is Jedis?**

Jedis is a high-performance Java client for Redis, providing direct access to all Redis commands through a simple API. It's widely used in enterprise production systems for its reliability, connection pooling, and comprehensive feature support.

**When to use Jedis:**
- Spring Boot microservices requiring Redis
- High-throughput Java applications
- Production systems needing connection pooling
- Applications using probabilistic data structures
- Real-time analytics and caching layers

> **Key Advantage:** Jedis provides low-level access to Redis commands including RedisBloom (probabilistic structures), enabling advanced use cases like Count-Min Sketch and Top-K tracking.

---

### Maven Dependencies

```xml
<dependencies>
    <!-- Jedis Redis Client -->
    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>5.0.0</version>
    </dependency>
    
    <!-- Spring Boot (if using Spring) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

---

### Basic Configuration

#### Spring Boot Configuration

```java
@Configuration
public class RedisConfig {
    
    @Bean
    public Jedis jedis() {
        return new Jedis("localhost", 6379);
    }
    
    @Bean
    public JedisPool jedisPool() {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxTotal(128);
        poolConfig.setMaxIdle(128);
        poolConfig.setMinIdle(16);
        poolConfig.setTestOnBorrow(true);
        poolConfig.setTestOnReturn(true);
        poolConfig.setTestWhileIdle(true);
        
        return new JedisPool(poolConfig, "localhost", 6379);
    }
}
```

**Connection pooling benefits:**
- Reuses connections (no TCP handshake overhead)
- Thread-safe for concurrent access
- Automatic connection health checks
- Configurable pool sizing for throughput

---

### Production Pattern 1: Spotify-Style Top-K Song Tracking

This real-world example demonstrates Count-Min Sketch + Sorted Sets for memory-efficient leaderboards.

#### Architecture Overview

```
┌─────────────────┐
│  Song Plays     │
│  (Millions/sec) │
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  Count-Min Sketch       │
│  (Approximate Frequency)│
│  Memory: O(ε⁻¹ log δ⁻¹) │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│  Redis ZSET (Top-K)     │
│  (Sorted Leaderboard)   │
│  Memory: O(K × log K)   │
└─────────────────────────┘
```

#### Data Models

```java
public class PlayEvent {
    private String songId;
    private String genre;
    private long timestamp;
    
    // Getters/setters
}
```

#### Redis Service Implementation

```java
@Service
public class RedisTopKService {
    
    private final Jedis jedis;
    
    public RedisTopKService(Jedis jedis) {
        this.jedis = jedis;
    }
    
    /**
     * Increment song play count in Count-Min Sketch
     * O(d) time where d = depth parameter
     */
    public void incrementSongPlay(String genre, String songId) {
        String cmsKey = "cms:genre:" + genre;
        jedis.sendCommand(
            Protocol.Command.valueOf("CMS.INCRBY"), 
            cmsKey, 
            songId, 
            "1"
        );
    }
    
    /**
     * Query approximate play count from CMS
     * Returns overestimate (never underestimates)
     */
    public long getApproxCount(String genre, String songId) {
        String cmsKey = "cms:genre:" + genre;
        List<String> result = jedis.sendCommand(
            Protocol.Command.valueOf("CMS.QUERY"), 
            cmsKey, 
            songId
        );
        return Long.parseLong(result.get(0));
    }
    
    /**
     * Sync CMS count to Sorted Set for ranking
     * Called periodically or on-demand
     */
    public void syncToTopK(String genre, String songId) {
        long count = getApproxCount(genre, songId);
        String zsetKey = "zset:genre:" + genre;
        jedis.zadd(zsetKey, count, songId);
    }
    
    /**
     * Get Top-K songs from sorted set
     * O(log N + K) time complexity
     */
    public Set<Tuple> getTopK(String genre, int k) {
        String zsetKey = "zset:genre:" + genre;
        return jedis.zrevrangeWithScores(zsetKey, 0, k - 1);
    }
    
    /**
     * Batch sync for multiple songs (more efficient)
     */
    public void batchSyncToTopK(String genre, Set<String> songIds) {
        String cmsKey = "cms:genre:" + genre;
        String zsetKey = "zset:genre:" + genre;
        
        Pipeline pipeline = jedis.pipelined();
        
        for (String songId : songIds) {
            long count = getApproxCount(genre, songId);
            pipeline.zadd(zsetKey, count, songId);
        }
        
        pipeline.sync();
    }
}
```

#### REST Controller

```java
@RestController
@RequestMapping("/songs")
public class SongController {
    
    private final RedisTopKService redisService;
    
    public SongController(RedisTopKService redisService) {
        this.redisService = redisService;
    }
    
    @PostMapping("/play")
    public ResponseEntity<String> playSong(@RequestBody PlayEvent event) {
        redisService.incrementSongPlay(event.getGenre(), event.getSongId());
        return ResponseEntity.ok("Play counted");
    }
    
    @GetMapping("/topk")
    public ResponseEntity<List<Map<String, Object>>> getTopK(
        @RequestParam String genre,
        @RequestParam(defaultValue = "10") int k) {
        
        Set<Tuple> topK = redisService.getTopK(genre, k);
        
        List<Map<String, Object>> result = topK.stream()
            .map(tuple -> Map.of(
                "song_id", tuple.getElement(),
                "plays", (long) tuple.getScore()
            ))
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(result);
    }
}
```

#### Scheduled Sync Task

```java
@Component
public class TopKSyncScheduler {
    
    private final RedisTopKService redisService;
    
    @Scheduled(fixedRate = 60000) // Every 60 seconds
    public void syncTopKRankings() {
        List<String> genres = List.of("pop", "rock", "jazz", "electronic");
        
        for (String genre : genres) {
            // Get candidate songs (from recent plays or trending list)
            Set<String> candidates = getCandidateSongs(genre);
            
            // Batch sync to sorted set
            redisService.batchSyncToTopK(genre, candidates);
            
            // Trim to keep only top 100
            String zsetKey = "zset:genre:" + genre;
            jedis.zremrangeByRank(zsetKey, 0, -101);
        }
    }
    
    private Set<String> getCandidateSongs(String genre) {
        // Implementation depends on your architecture
        // Could be:
        // 1. Union of current top-K + recently played songs
        // 2. Songs with recent activity from stream processor
        // 3. Bloom filter of active songs
        return Set.of(); // Placeholder
    }
}
```

---

### Production Pattern 2: Redis Top-K Sketch (Native)

RedisBloom provides native Top-K sketch support with automatic heavy hitters tracking.

```java
@Service
public class TopKSketchService {
    
    private final Jedis jedis;
    
    public TopKSketchService(Jedis jedis) {
        this.jedis = jedis;
    }
    
    /**
     * Initialize Top-K sketch
     * @param k - number of top items to track
     * @param width - affects accuracy (higher = more accurate)
     * @param depth - number of hash functions
     * @param decay - exponential decay factor (0.9-0.925 typical)
     */
    public void initTopK(String key, int k, int width, int depth, double decay) {
        jedis.sendCommand(
            Protocol.Command.valueOf("TOPK.RESERVE"),
            key,
            String.valueOf(k),
            String.valueOf(width),
            String.valueOf(depth),
            String.valueOf(decay)
        );
    }
    
    /**
     * Add item to Top-K
     * Returns expelled item if evicted from top-K
     */
    public void addToTopK(String key, String item) {
        jedis.sendCommand(
            Protocol.Command.valueOf("TOPK.ADD"),
            key,
            item
        );
    }
    
    /**
     * Get current top-K list
     */
    public List<String> getTopK(String key) {
        return (List<String>) jedis.sendCommand(
            Protocol.Command.valueOf("TOPK.LIST"),
            key
        );
    }
    
    /**
     * Query if item is in top-K
     */
    public boolean isInTopK(String key, String item) {
        List<Long> result = (List<Long>) jedis.sendCommand(
            Protocol.Command.valueOf("TOPK.QUERY"),
            key,
            item
        );
        return result.get(0) == 1;
    }
    
    /**
     * Get frequency estimates for items
     */
    public Map<String, Long> getFrequencies(String key, String... items) {
        List<String> command = new ArrayList<>();
        command.add(key);
        command.addAll(Arrays.asList(items));
        
        List<Long> counts = (List<Long>) jedis.sendCommand(
            Protocol.Command.valueOf("TOPK.COUNT"),
            command.toArray(new String[0])
        );
        
        Map<String, Long> result = new HashMap<>();
        for (int i = 0; i < items.length; i++) {
            result.put(items[i], counts.get(i));
        }
        return result;
    }
}
```

**Real-world usage:**

```java
// Initialize for tracking top 100 trending hashtags
topKService.initTopK("trending:hashtags", 100, 2000, 7, 0.925);

// Stream processing: increment on each tweet
topKService.addToTopK("trending:hashtags", "#redis");
topKService.addToTopK("trending:hashtags", "#java");

// Get current trending hashtags
List<String> trending = topKService.getTopK("trending:hashtags");
// Returns: ["#redis", "#java", "#spring", ...]
```

---

### Production Pattern 3: Bloom Filter for Duplicate Detection

Prevent caching "one-hit wonders" - only cache content accessed multiple times.

```java
@Service
public class BloomFilterCacheService {
    
    private final Jedis jedis;
    private final ContentCache contentCache;
    
    public BloomFilterCacheService(Jedis jedis, ContentCache contentCache) {
        this.jedis = jedis;
        this.contentCache = contentCache;
    }
    
    /**
     * Initialize Bloom filter
     * @param errorRate - false positive probability (0.01 = 1%)
     * @param capacity - expected number of elements
     */
    public void initBloomFilter(String key, double errorRate, long capacity) {
        jedis.sendCommand(
            Protocol.Command.valueOf("BF.RESERVE"),
            key,
            String.valueOf(errorRate),
            String.valueOf(capacity)
        );
    }
    
    /**
     * Smart caching: only cache on second access
     * Avoids wasting cache on one-hit wonders
     */
    public Content getContent(String contentId) {
        String bloomKey = "content:seen";
        
        // Check if we've seen this content before
        boolean seenBefore = checkBloomFilter(bloomKey, contentId);
        
        if (!seenBefore) {
            // First access: mark as seen but don't cache
            addToBloomFilter(bloomKey, contentId);
            return loadFromDatabase(contentId);
        } else {
            // Second+ access: check cache, populate if needed
            Content cached = contentCache.get(contentId);
            if (cached != null) {
                return cached;
            }
            
            // Cache miss: load and cache
            Content content = loadFromDatabase(contentId);
            contentCache.put(contentId, content);
            return content;
        }
    }
    
    private boolean checkBloomFilter(String key, String item) {
        List<Long> result = (List<Long>) jedis.sendCommand(
            Protocol.Command.valueOf("BF.EXISTS"),
            key,
            item
        );
        return result.get(0) == 1;
    }
    
    private void addToBloomFilter(String key, String item) {
        jedis.sendCommand(
            Protocol.Command.valueOf("BF.ADD"),
            key,
            item
        );
    }
    
    private Content loadFromDatabase(String contentId) {
        // Database query logic
        return null; // Placeholder
    }
}
```

**Akamai's results with this pattern:**
- **75% cache storage reduction** across 325,000 servers
- Finding: 75% of content accessed only once
- Bloom filter overhead: ~10 bits per item

---

### Production Pattern 4: Performance Optimization with Pipelining

Reduce network round trips for batch operations.

```java
@Service
public class RedisPerformanceService {
    
    private final JedisPool jedisPool;
    
    /**
     * Slow approach: Individual commands (N round trips)
     */
    public void slowBatchUpdate(Map<String, String> keyValues) {
        try (Jedis jedis = jedisPool.getResource()) {
            for (Map.Entry<String, String> entry : keyValues.entrySet()) {
                jedis.set(entry.getKey(), entry.getValue());
                // Each SET = 1 round trip
            }
        }
    }
    
    /**
     * Fast approach 1: MSET (1 round trip)
     */
    public void fastBatchUpdateMSET(Map<String, String> keyValues) {
        try (Jedis jedis = jedisPool.getResource()) {
            String[] keysAndValues = keyValues.entrySet().stream()
                .flatMap(e -> Stream.of(e.getKey(), e.getValue()))
                .toArray(String[]::new);
            
            jedis.mset(keysAndValues);
            // Single round trip for all keys
        }
    }
    
    /**
     * Fast approach 2: Pipeline (N commands, 1 round trip)
     */
    public void fastBatchUpdatePipeline(Map<String, String> keyValues) {
        try (Jedis jedis = jedisPool.getResource()) {
            Pipeline pipeline = jedis.pipelined();
            
            for (Map.Entry<String, String> entry : keyValues.entrySet()) {
                pipeline.set(entry.getKey(), entry.getValue());
            }
            
            pipeline.sync(); // Execute all commands in one round trip
        }
    }
    
    /**
     * Performance comparison results:
     * 
     * 1000 individual SETs:  ~50ms (1ms per round trip × 1000)
     * 1 MSET (1000 pairs):   ~1ms
     * Pipeline (1000 SETs):  ~2ms
     * 
     * Speedup: 25-50x faster!
     */
}
```

---

### Production Pattern 5: Using Hashes for Objects

Store objects efficiently instead of multiple keys.

```java
@Service
public class UserProfileService {
    
    private final JedisPool jedisPool;
    
    /**
     * Anti-pattern: Multiple keys per user
     * Problems: More memory overhead, slower multi-field queries
     */
    public void saveUserAntiPattern(String userId, UserProfile profile) {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.set("user:" + userId + ":name", profile.getName());
            jedis.set("user:" + userId + ":email", profile.getEmail());
            jedis.set("user:" + userId + ":age", String.valueOf(profile.getAge()));
            // 3 round trips, 3 separate keys
        }
    }
    
    /**
     * Best practice: Use hash for object
     * Benefits: Single key, atomic operations, memory efficient
     */
    public void saveUserBestPractice(String userId, UserProfile profile) {
        try (Jedis jedis = jedisPool.getResource()) {
            Map<String, String> fields = new HashMap<>();
            fields.put("name", profile.getName());
            fields.put("email", profile.getEmail());
            fields.put("age", String.valueOf(profile.getAge()));
            fields.put("created_at", String.valueOf(System.currentTimeMillis()));
            
            jedis.hset("user:" + userId, fields);
            // Single round trip, single key
        }
    }
    
    /**
     * Retrieve full profile
     */
    public UserProfile getUserProfile(String userId) {
        try (Jedis jedis = jedisPool.getResource()) {
            Map<String, String> fields = jedis.hgetAll("user:" + userId);
            
            if (fields.isEmpty()) {
                return null;
            }
            
            return new UserProfile(
                fields.get("name"),
                fields.get("email"),
                Integer.parseInt(fields.get("age"))
            );
        }
    }
    
    /**
     * Retrieve specific fields only
     */
    public String getUserEmail(String userId) {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.hget("user:" + userId, "email");
        }
    }
    
    /**
     * Atomic field increment
     */
    public long incrementLoginCount(String userId) {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.hincrBy("user:" + userId, "login_count", 1);
        }
    }
}
```

**Memory savings:**
- Multiple keys: ~56 bytes overhead per key
- Hash fields: ~24 bytes overhead per field
- For 100 fields: 5600 bytes vs 2400 bytes (58% reduction)

---

### Production Pattern 6: Lua Scripting for Atomic Operations

Guarantee atomicity for complex operations.

```java
@Service
public class RedisAtomicService {
    
    private final JedisPool jedisPool;
    
    /**
     * Atomic compare-and-swap
     * Only update if current value matches expected
     */
    private static final String CAS_SCRIPT =
        "local current = redis.call('GET', KEYS[1])\n" +
        "if current == ARGV[1] then\n" +
        "    redis.call('SET', KEYS[1], ARGV[2])\n" +
        "    return 1\n" +
        "else\n" +
        "    return 0\n" +
        "end";
    
    public boolean compareAndSwap(String key, String expectedValue, String newValue) {
        try (Jedis jedis = jedisPool.getResource()) {
            Object result = jedis.eval(
                CAS_SCRIPT,
                Collections.singletonList(key),
                Arrays.asList(expectedValue, newValue)
            );
            return ((Long) result) == 1;
        }
    }
    
    /**
     * Atomic increment with max value
     * Prevents counter from exceeding limit
     */
    private static final String INCR_WITH_MAX_SCRIPT =
        "local current = tonumber(redis.call('GET', KEYS[1]) or 0)\n" +
        "local max = tonumber(ARGV[1])\n" +
        "local increment = tonumber(ARGV[2])\n" +
        "if current + increment <= max then\n" +
        "    return redis.call('INCRBY', KEYS[1], increment)\n" +
        "else\n" +
        "    return current\n" +
        "end";
    
    public long incrementWithMax(String key, long max, long increment) {
        try (Jedis jedis = jedisPool.getResource()) {
            Object result = jedis.eval(
                INCR_WITH_MAX_SCRIPT,
                Collections.singletonList(key),
                Arrays.asList(String.valueOf(max), String.valueOf(increment))
            );
            return (Long) result;
        }
    }
    
    /**
     * Sliding window rate limiter
     */
    private static final String RATE_LIMIT_SCRIPT =
        "local key = KEYS[1]\n" +
        "local limit = tonumber(ARGV[1])\n" +
        "local window = tonumber(ARGV[2])\n" +
        "local now = tonumber(ARGV[3])\n" +
        "\n" +
        "redis.call('ZREMRANGEBYSCORE', key, 0, now - window)\n" +
        "local current = redis.call('ZCARD', key)\n" +
        "\n" +
        "if current < limit then\n" +
        "    redis.call('ZADD', key, now, now)\n" +
        "    redis.call('EXPIRE', key, window)\n" +
        "    return 1\n" +
        "else\n" +
        "    return 0\n" +
        "end";
    
    public boolean checkRateLimit(String userId, int limit, int windowSeconds) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = "rate_limit:" + userId;
            long now = System.currentTimeMillis() / 1000;
            
            Object result = jedis.eval(
                RATE_LIMIT_SCRIPT,
                Collections.singletonList(key),
                Arrays.asList(
                    String.valueOf(limit),
                    String.valueOf(windowSeconds),
                    String.valueOf(now)
                )
            );
            return ((Long) result) == 1;
        }
    }
}
```

---

### Production Best Practices Summary

#### Connection Pooling

```java
@Bean
public JedisPool jedisPool() {
    JedisPoolConfig config = new JedisPoolConfig();
    
    // Pool sizing for high throughput
    config.setMaxTotal(128);        // Maximum connections
    config.setMaxIdle(128);         // Maximum idle connections
    config.setMinIdle(16);          // Minimum idle connections
    
    // Connection health
    config.setTestOnBorrow(true);   // Validate before use
    config.setTestOnReturn(true);   // Validate after use
    config.setTestWhileIdle(true);  // Validate idle connections
    
    // Eviction policy
    config.setMinEvictableIdleTimeMillis(60000);
    config.setTimeBetweenEvictionRunsMillis(30000);
    
    // Wait behavior
    config.setBlockWhenExhausted(true);
    config.setMaxWaitMillis(3000);  // Max wait for connection
    
    return new JedisPool(config, "localhost", 6379, 2000);
}
```

#### Resource Management

```java
// Always use try-with-resources
public String getValue(String key) {
    try (Jedis jedis = jedisPool.getResource()) {
        return jedis.get(key);
    }
    // Connection automatically returned to pool
}

// Pipeline usage
public void batchOperations() {
    try (Jedis jedis = jedisPool.getResource()) {
        Pipeline pipeline = jedis.pipelined();
        
        // Queue operations
        pipeline.set("key1", "value1");
        pipeline.set("key2", "value2");
        pipeline.incr("counter");
        
        // Execute all
        pipeline.sync();
    }
}
```

#### Error Handling

```java
public String robustGet(String key) {
    int maxRetries = 3;
    int retryDelay = 100; // milliseconds
    
    for (int i = 0; i < maxRetries; i++) {
        try (Jedis jedis = jedisPool.getResource()) {
            return jedis.get(key);
        } catch (JedisConnectionException e) {
            if (i == maxRetries - 1) {
                throw e; // Last retry failed
            }
            try {
                Thread.sleep(retryDelay * (i + 1)); // Exponential backoff
            } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(ie);
            }
        }
    }
    return null;
}
```

---

### Performance Benchmarks

**Test environment:**
- Redis 7.2, localhost
- Jedis 5.0.0
- Intel i7, 16GB RAM

| Operation | Throughput | Latency (P99) |
|-----------|-----------|---------------|
| GET (pooled) | 150K ops/sec | 0.8ms |
| SET (pooled) | 140K ops/sec | 0.9ms |
| MGET (100 keys) | 80K ops/sec | 1.2ms |
| Pipeline (100 SETs) | 200K ops/sec | 0.5ms |
| HGETALL (10 fields) | 120K ops/sec | 1.0ms |
| Lua script (CAS) | 90K ops/sec | 1.3ms |

**Key findings:**
- Pipeline: 25-50x faster than individual commands
- Connection pooling: 3-5x faster than creating connections
- MGET/MSET: 10-20x faster than individual operations

---

### Real-World Architecture: Spotify Top-K

```
┌────────────────────────────────────────────────────┐
│              Music Streaming Events                 │
│         (Millions of plays per second)              │
└───────────────────┬────────────────────────────────┘
                    │
                    ▼
        ┌──────────────────────────┐
        │  Spring Boot Services    │
        │  (Kafka Consumers)       │
        └────────┬─────────────────┘
                 │
                 ▼
    ┌────────────────────────────────┐
    │   Redis Cluster (Sharded)      │
    ├────────────────────────────────┤
    │  Shard 1: Genre "pop"          │
    │   └─ CMS: cms:genre:pop        │
    │   └─ ZSET: zset:genre:pop      │
    ├────────────────────────────────┤
    │  Shard 2: Genre "rock"         │
    │   └─ CMS: cms:genre:rock       │
    │   └─ ZSET: zset:genre:rock     │
    └────────────────────────────────┘
                 │
                 ▼
    ┌────────────────────────────────┐
    │  Scheduled Sync Job (60s)      │
    │  - Query CMS for candidates    │
    │  - Update ZSET rankings        │
    │  - Trim to top 100             │
    └────────────────────────────────┘
                 │
                 ▼
    ┌────────────────────────────────┐
    │   API: GET /topk?genre=pop     │
    │   Returns: Top 10 songs        │
    └────────────────────────────────┘
```

**Scaling characteristics:**
- CMS memory per genre: ~40KB (for 0.1% error)
- ZSET memory per genre: ~5KB (100 songs × 50 bytes)
- Total memory for 100 genres: ~4.5MB
- Throughput: 1M+ plays/sec per shard

---

This completes Chapter 41 on Java/Jedis integration with production patterns demonstrating real-world Redis usage in high-scale systems.

---
