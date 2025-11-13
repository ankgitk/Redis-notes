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

This comprehensive introduction sets the stage for deep technical exploration. In the following chapters, we'll build Redis from scratch, understanding every decision, trade-off, and optimization that makes it one of the fastest databases in the world.

