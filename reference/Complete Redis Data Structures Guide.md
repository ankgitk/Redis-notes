# Complete Redis Data Structures Guide

Redis offers eight core data structures that enable developers to build sophisticated applications with high performance and minimal complexity. Each data structure is optimized for specific use cases and provides atomic operations that ensure data integrity in concurrent environments.

## Strings: The foundation of Redis data storage

**What are Redis Strings?**

Redis Strings are the most basic data type, representing sequences of bytes that can store text, numbers, or binary data up to 512 MB. Despite being called "strings," they can efficiently handle integers and floating-point numbers with dedicated arithmetic operations.

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

#### EXPIRE/TTL - Set and check expiration
```bash
EXPIRE key seconds                         # Set expiration time
TTL key                                   # Get remaining time to live
```

```bash
SET temp:data "temporary"
EXPIRE temp:data 300                      # Expires in 5 minutes
TTL temp:data                             # Returns remaining seconds
```

---

## Lists: Ordered collections for queues and stacks

**What are Redis Lists?**

Redis Lists are ordered collections of strings sorted by insertion order. Implemented as doubly-linked lists, they provide fast insertion and deletion at both ends, making them perfect for queues, stacks, and activity feeds.

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
```

---

## Sets: Unique collections for membership and relationships

**What are Redis Sets?**

Redis Sets are unordered collections of unique strings that support fast membership testing, intersection, union, and difference operations. They're perfect for representing relationships and implementing tag systems.

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

#### Set operations
```bash
SINTER key [key ...]                      # Intersection
SUNION key [key ...]                      # Union  
SDIFF key [key ...]                       # Difference
```

```bash
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"

SINTER set1 set2                          # Returns: ["b", "c"]
SUNION set1 set2                          # Returns: ["a", "b", "c", "d"]
SDIFF set1 set2                           # Returns: ["a"] (in set1 but not set2)
```

---

## Sorted Sets: Ranked collections with scores

**What are Redis Sorted Sets (ZSets)?**

Sorted Sets combine the uniqueness of Sets with automatic ordering by score. Each member has an associated floating-point score that determines its position in the set. They enable efficient range queries, ranking operations, and score-based filtering.

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
ZREVRANGE leaderboard 0 4                 # Returns: ["player2", "player1", "player3"]

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

#### ZRANGEBYSCORE - Get members by score range
```bash
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
```

```bash
ZRANGEBYSCORE scoreboard 100 200          # Returns members with scores 100-200
ZRANGEBYSCORE scoreboard "(100" "+inf"    # Returns members with scores > 100
ZRANGEBYSCORE scoreboard -inf 200 LIMIT 0 5  # Returns first 5 members with score ≤ 200
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

---

## Hashes: Field-value pairs within keys

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

#### HINCRBY - Increment field value
```bash
HINCRBY key field increment
```

```bash
HINCRBY user:100 login_count 1            # Returns: (integer) 1
HINCRBY user:100 points -50               # Decrement by 50
```

---

## Bitmaps: Efficient binary operations

**What are Redis Bitmaps?**

Redis Bitmaps are not a separate data type but bit-oriented operations on Strings. They treat strings as bit vectors, allowing efficient manipulation of individual bits. Perfect for representing boolean information across large domains with minimal memory usage.

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
```

#### BITCOUNT - Count set bits
```bash
BITCOUNT key [start end]
```

```bash
BITCOUNT user:flags                       # Count all set bits
BITCOUNT user:flags 0 10                  # Count set bits in bytes 0-10
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
```

---

## HyperLogLogs: Cardinality estimation at scale

**What are Redis HyperLogLogs?**

HyperLogLog is a probabilistic data structure that estimates cardinality (number of unique elements) using constant memory (12 KB maximum), regardless of dataset size. It provides approximate counts with 0.81% standard error, making it perfect for large-scale analytics.

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
PFMERGE weekly_users daily:mon daily:tue daily:wed
PFCOUNT weekly_users                      # Returns weekly unique count
```

---

## Streams: Event logs and message queues

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
```

#### 2. IoT sensor monitoring
```bash
# Temperature sensor data with location and readings
XADD sensors:temperature * device_id "temp_001" location "building_A_floor_2" temperature "23.5" humidity "45.2"
```

#### 3. Real-time notifications
```bash
# Add notification to user's stream
XADD notifications:user_789 * type "message" from "user_123" title "New Message" content "Hello there!"
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

#### XRANGE - Read range of entries
```bash
XRANGE key start end [COUNT count]
```

```bash
XRANGE events - +                         # Get all entries
XRANGE events 1692629576966 1692629586966 # Get entries in time range
XREVRANGE events + - COUNT 10             # Get last 10 entries (newest first)
```

#### Consumer groups for guaranteed delivery
```bash
# Create consumer group
XGROUP CREATE stream group start-id

# Read as consumer group member
XREADGROUP GROUP group consumer COUNT count STREAMS stream >

# Acknowledge processed messages
XACK stream group message-id [message-id ...]
```

**Example workflow:**
```bash
# Create processing group starting from latest entries
XGROUP CREATE user_events processors $

# Consumer reads new messages
XREADGROUP GROUP processors worker1 COUNT 5 STREAMS user_events >

# Process messages, then acknowledge
XACK user_events processors 1692629576966-0 1692629576967-0
```

---

## Performance considerations and best practices

### Memory optimization strategies

**String optimization:**
- Use appropriate data types (prefer INCR over SET for counters)
- Set expiration times for temporary data
- Use compression for large text values

**List optimization:**
- Trim lists regularly with LTRIM to control memory usage
- Use blocking operations (BLPOP/BRPOP) for real-time processing
- Consider Redis Streams for complex message queuing

**Set and Sorted Set optimization:**
- Monitor set cardinality - very large sets impact performance
- Use appropriate operations (SSCAN for large set iteration)
- Consider data partitioning for massive sorted sets

**Hash optimization:**
- Keep individual hashes small (< 100 fields) for memory optimization
- Use consistent field naming to reduce overhead
- Prefer HMGET over multiple HGET calls

### Choosing the right data structure

**Use Strings when:**
- Storing simple key-value pairs
- Implementing counters or rate limiting
- Caching serialized data

**Use Lists when:**
- Order matters (FIFO/LIFO operations)
- Implementing queues or activity feeds
- You need blocking operations

**Use Sets when:**
- Uniqueness is important
- You need set operations (intersection, union)
- Implementing tagging or relationship systems

**Use Sorted Sets when:**
- You need ordered data with scores
- Building leaderboards or ranking systems
- Time-series data with range queries

**Use Hashes when:**
- Representing objects with multiple attributes
- You need atomic field updates
- Modeling structured data

**Use Bitmaps when:**
- Tracking binary states efficiently
- Large-scale user activity analytics
- Memory efficiency is critical

**Use HyperLogLogs when:**
- Estimating large cardinalities
- Memory usage must be constant
- Approximate counts are acceptable

**Use Streams when:**
- Building event-driven systems
- You need guaranteed message delivery
- Processing real-time data pipelines

### System design patterns

**Caching pattern:**
```bash
# Cache-aside pattern with automatic expiration
SET cache:user:123 '{"name":"John"}' EX 300
GET cache:user:123
```

**Rate limiting pattern:**
```bash
# Sliding window rate limiter with Sorted Sets
ZADD rate:user:123 1692629576 "request1"
ZREMRANGEBYSCORE rate:user:123 0 1692626976  # Remove old entries
ZCARD rate:user:123                           # Check current request count
```

**Pub/Sub with Streams:**
```bash
# Reliable message processing with consumer groups
XADD events * type "order" user_id "123" amount "99.99"
XREADGROUP GROUP processors worker1 STREAMS events >
```

**Analytics pipeline:**
```bash
# Multi-stage analytics using different data structures
PFADD daily_uniques:2024-06-19 user_123     # Track unique users
HINCRBY metrics:2024-06-19 page_views 1     # Count page views
ZADD top_pages:2024-06-19 1 "/homepage"     # Track popular pages
```

This comprehensive guide provides both foundational understanding and practical implementation patterns for all Redis data structures. Each data type offers unique advantages for specific use cases, and combining them effectively enables building sophisticated, high-performance applications that scale efficiently.
