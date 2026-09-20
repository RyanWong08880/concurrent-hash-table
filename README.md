# Concurrent Hash Table

This project implements thread-safe hash table operations using pthread mutexes. The implementation uses separate chaining to resolve collisions and provides two different locking strategies.

## Building
```shell
make
```

This creates the `hash-benchmark` executable.

## Running
```shell
./hash-table-tester -t <num_threads> -s <entries_per_thread>
```

Example:
```shell
./hash-benchmark -t 4 -s 25000
```

**Example output:**
```
Generation: 18,073 usec
Hash table base: 60,331 usec
  - 0 missing
Hash table v1: 190,894 usec
  - 0 missing
Hash table v2: 36,729 usec
  - 0 missing
```

## First Implementation

In the `hash_table_v1_add_entry` function, I added a single pthread_mutex_t to the `hash_table_v1` structure. I initialized the mutex in `hash_table_v1_create()`, then locked the mutex at the start of `hash_table_v1_add_entry()`, then unlocked it before returning in all code paths, and finally destroyed the mutex in `hash_table_v1_destroy()`.

This is correct because this single mutex ensures mutual exclusion for all add_entry operations. Since the entire operation (getting the hash table entry, searching the list, and potentially inserting a new entry) is protected by the lock, race conditions are prevented. All threads are serialized when accessing the hash table, guaranteeing that no two threads can simultaneously modify the data structure.

### Performance
```shell
./hash-benchmark -t 4 -s 25000
```

**Results:**
- Base implementation: 60,331 usec
- V1 implementation: 190,894 usec

Version 1 is a little slower than the base version as thread creation overhead and lock contention are present. Even though we have 4 threads available, the single mutex forces all threads to wait for each other, resulting in serial execution with the added cost of mutex operations. The performance degrades because of context switching overhead and serialization is forced (because all threads contend for the single lock).

## Second Implementation

In the `hash_table_v2_add_entry` function, I implemented fine-grained locking by adding a pthread_mutex_t to each hash_table_entry (bucket). First, I added `pthread_mutex_t mutex;` field to `struct hash_table_entry`, and initialized all 4096 bucket mutexes in `hash_table_v2_create()`. Then, in `hash_table_v2_add_entry()`:
1. Compute which bucket the key hashes to
2. Lock only that specific bucket's mutex
3. Perform the lookup and insertion
4. Unlock that bucket's mutex
Finally, I destroyed all mutexes in `hash_table_v2_destroy()`.

This per-bucket locking is not only correct but also better parallelized. Each bucket's mutex protects only the operations on that specific bucket. Since different threads are likely to hash to different buckets, they can operate concurrently without blocking each other. The hash function (bernstein_hash) distributes keys across all 4096 buckets, so threads rarely contend for the same lock.

### Performance
```shell
./hash-benchmark -t 4 -s 25000
```

**Results:**
- Base implementation: 60,331 usec
- V2 implementation: 36,729 usec

V2 is 1.64x faster than the base implementation because of the per-bucket locking strategy:
1. With 4 threads and 4096 buckets, threads rarely contend for the same lock
2. The probability of two threads hashing to the same bucket is low, allowing concurrent operations
3. The fine-grained locking reduces serialization while maintaining correctness

## Cleaning up
```shell
make clean
```

This removes all object files and the executable.