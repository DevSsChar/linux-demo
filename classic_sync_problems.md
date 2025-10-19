# Classic Synchronization Problems - Complete Guide
## Producer-Consumer | Reader-Writer | Dining Philosophers | Peterson's Solution

---

## 7. Compilation & Testing

### 7.1 Compilation Commands

```bash
# Peterson's Solution
gcc peterson_solution.c -o peterson -pthread

# Producer-Consumer
gcc producer_consumer_semaphore.c -o prodcons_sem -pthread
gcc producer_consumer_condvar.c -o prodcons_cond -pthread

# Reader-Writer
gcc reader_writer_readers_first.c -o rw_readers -pthread
gcc reader_writer_writers_first.c -o rw_writers -pthread

# Dining Philosophers
gcc dining_philosophers.c -o dining -pthread
gcc dining_philosophers_ordering.c -o dining_order -pthread

# Compile all at once
gcc peterson_solution.c -o peterson -pthread && \
gcc producer_consumer_semaphore.c -o prodcons_sem -pthread && \
gcc producer_consumer_condvar.c -o prodcons_cond -pthread && \
gcc reader_writer_readers_first.c -o rw_readers -pthread && \
gcc reader_writer_writers_first.c -o rw_writers -pthread && \
gcc dining_philosophers.c -o dining -pthread && \
gcc dining_philosophers_ordering.c -o dining_order -pthread && \
echo "All programs compiled successfully!"
```

### 7.2 Execution Examples

```bash
# Peterson's Solution
./peterson
# Watch mutual exclusion in action

# Producer-Consumer (Semaphore)
./prodcons_sem
# Observe buffer fill/empty patterns

# Producer-Consumer (Condition Variables)
./prodcons_cond
# See waiting and signaling

# Reader-Writer (Readers first)
./rw_readers
# Notice multiple simultaneous readers

# Reader-Writer (Writers first)
./rw_writers
# See writers getting priority

# Dining Philosophers
./dining
# Watch philosophers think, eat, avoid deadlock

# Dining Philosophers (Ordering)
./dining_order
# See resource ordering preventing deadlock
```

### 7.3 Testing for Race Conditions

**Stress Test Producer-Consumer:**
```bash
# Modify NUM_ITEMS to large value
# In source: #define NUM_ITEMS 1000

gcc producer_consumer_semaphore.c -o prodcons_test -pthread
time ./prodcons_test

# Should complete without errors
# Check: All items produced = All items consumed
```

**Test Reader-Writer Starvation:**
```bash
# Increase reader count
# In readers_first: Create 10 readers, 2 writers
# Observe: Writers may wait long time (starvation)

# In writers_first: Create 3 readers, 10 writers
# Observe: Readers may be blocked frequently
```

**Test Dining Philosophers Deadlock:**
```bash
# Run multiple times
./dining

# If deadlock occurs, program hangs
# Our solutions prevent this!
```

### 7.4 Debugging with GDB

```bash
# Compile with debug symbols
gcc -g peterson_solution.c -o peterson_debug -pthread

# Run with GDB
gdb ./peterson_debug

# GDB commands:
(gdb) break process          # Set breakpoint
(gdb) run                    # Start program
(gdb) info threads           # Show all threads
(gdb) thread 2               # Switch to thread 2
(gdb) print shared_counter   # Check variable
(gdb) continue               # Continue execution
```

### 7.5 Monitoring with Valgrind

```bash
# Check for thread errors
valgrind --tool=helgrind ./prodcons_sem

# Check for memory leaks
valgrind --leak-check=full ./dining
```

---

## 8. Troubleshooting

### 8.1 Common Compilation Errors

#### Error: "undefined reference to pthread_create"
```bash
# Missing -pthread flag
# ❌ Wrong:
gcc program.c -o program

# ✅ Correct:
gcc program.c -o program -pthread
```

#### Error: "sem_init implicit declaration"
```c
// Missing header
// ✅ Add:
#include <semaphore.h>
```

#### Error: "conflicting types for 'philosopher'"
```c
// Forward declaration issue
// ✅ Add function prototype:
void* philosopher(void* arg);
```

### 8.2 Runtime Issues

#### Issue: Program Hangs (Deadlock)

**Symptoms:**
- Program stops responding
- No output
- CPU usage at 0% or 100% (busy waiting)

**Common Causes:**

1. **Forgot to unlock mutex:**
```c
// ❌ Wrong:
pthread_mutex_lock(&mutex);
if (error) return;  // Mutex never unlocked!
// do work

// ✅ Correct:
pthread_mutex_lock(&mutex);
if (error) {
    pthread_mutex_unlock(&mutex);
    return;
}
// do work
pthread_mutex_unlock(&mutex);
```

2. **Circular wait:**
```c
// ❌ Wrong: Thread 1 and 2 can deadlock
Thread 1: lock(A) → lock(B)
Thread 2: lock(B) → lock(A)

// ✅ Correct: Always lock in same order
Thread 1: lock(A) → lock(B)
Thread 2: lock(A) → lock(B)
```

3. **Forgot sem_post:**
```c
// ❌ Wrong:
sem_wait(&sem);
// critical section
// Forgot sem_post!

// ✅ Correct:
sem_wait(&sem);
// critical section
sem_post(&sem);
```

**How to Debug:**
```bash
# Send SIGQUIT to get thread dump
kill -QUIT <PID>

# Use gdb to see where threads are stuck
gdb -p <PID>
(gdb) info threads
(gdb) thread apply all bt
```

#### Issue: Race Condition Still Occurs

**Symptoms:**
- Incorrect final values
- Inconsistent results between runs
- Data corruption

**Checklist:**
```c
// ✅ Check: All shared variables protected
int shared_var;  // Is this locked?

// ✅ Check: Critical section fully covered
pthread_mutex_lock(&mutex);
// Are ALL operations on shared data here?
pthread_mutex_unlock(&mutex);

// ✅ Check: All threads use SAME lock
// Thread 1:
pthread_mutex_lock(&mutex1);  // ❌ Wrong lock!

// Thread 2:
pthread_mutex_lock(&mutex2);  // ❌ Different lock!

// ✅ Should both use mutex1
```

#### Issue: Starvation

**Symptoms:**
- Some threads never execute
- Unfair resource allocation
- Long waiting times for certain threads

**Solutions:**

1. **Readers-Preference causing writer starvation:**
```c
// Use writers-preference or fair solution
// Add maximum reader count
if (read_count >= MAX_READERS) {
    // Block new readers
}
```

2. **Add fairness mechanism:**
```c
// Use queue for waiting threads
// Implement fair scheduling
```

### 8.3 Performance Issues

#### Issue: Poor Performance (Excessive Contention)

**Symptoms:**
- High CPU usage
- Slow execution
- Threads spending time waiting

**Solutions:**

1. **Reduce critical section size:**
```c
// ❌ Bad: Large critical section
pthread_mutex_lock(&mutex);
expensive_computation();  // Takes 1 second
shared_var++;
pthread_mutex_unlock(&mutex);

// ✅ Good: Minimal critical section
expensive_computation();  // Outside lock
pthread_mutex_lock(&mutex);
shared_var++;  // Quick operation
pthread_mutex_unlock(&mutex);
```

2. **Use fine-grained locking:**
```c
// ❌ One lock for everything
pthread_mutex_t global_lock;

// ✅ Separate locks for different data
pthread_mutex_t queue_lock;
pthread_mutex_t counter_lock;
pthread_mutex_t cache_lock;
```

3. **Avoid busy waiting:**
```c
// ❌ Bad: Wastes CPU
while (!ready) {
    // Busy wait - burns CPU!
}

// ✅ Good: Sleep while waiting
pthread_mutex_lock(&mutex);
while (!ready) {
    pthread_cond_wait(&cond, &mutex);
}
pthread_mutex_unlock(&mutex);
```

### 8.4 Debugging Tips

#### Print Debugging

```c
// Add thread-safe logging
void log_msg(const char* msg) {
    pthread_mutex_lock(&log_mutex);
    printf("[%ld] Thread %lu: %s\n", 
           time(NULL), pthread_self(), msg);
    fflush(stdout);  // Force immediate output
    pthread_mutex_unlock(&log_mutex);
}
```

#### Assertions

```c
#include <assert.h>

// Check invariants
pthread_mutex_lock(&mutex);
assert(count >= 0 && count <= BUFFER_SIZE);
// critical section
pthread_mutex_unlock(&mutex);
```

#### Thread Sanitizer

```bash
# Compile with thread sanitizer
gcc -fsanitize=thread -g program.c -o program -pthread

# Run program
./program

# Will report data races and deadlocks
```

### 8.5 Common Mistakes Summary

| Mistake | Effect | Solution |
|---------|--------|----------|
| Forgot `-pthread` flag | Compilation error | Add `-pthread` |
| Forgot to unlock mutex | Deadlock | Always unlock |
| Wrong semaphore init value | Deadlock/race | Check sem_init() value |
| Not using volatile for Peterson's | Optimization breaks it | Use `volatile` |
| Same fork for all philosophers | Deadlock | Use resource ordering |
| Large critical section | Poor performance | Minimize locked code |
| No error checking | Hidden bugs | Check return values |

---

## 9. Advanced Topics

### 9.1 Reader-Writer Fair Solution

**Problem:** Both readers and writers can starve in previous solutions.

**Solution:** Use ticket/queue system:

```c
// File: reader_writer_fair.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

int shared_data = 0;
int read_count = 0;

sem_t order;      // Maintains FIFO order
sem_t access;     // Controls actual read/write
pthread_mutex_t read_mutex = PTHREAD_MUTEX_INITIALIZER;

void* reader(void* arg) {
    int id = *(int*)arg;
    
    // Get ticket
    sem_wait(&order);
    
    pthread_mutex_lock(&read_mutex);
    read_count++;
    if (read_count == 1) {
        sem_wait(&access);  // First reader locks writers
    }
    pthread_mutex_unlock(&read_mutex);
    
    sem_post(&order);  // Release ticket
    
    // Reading
    printf("Reader %d: Reading data = %d\n", id, shared_data);
    usleep(100000);
    
    pthread_mutex_lock(&read_mutex);
    read_count--;
    if (read_count == 0) {
        sem_post(&access);  // Last reader unlocks writers
    }
    pthread_mutex_unlock(&read_mutex);
    
    return NULL;
}

void* writer(void* arg) {
    int id = *(int*)arg;
    
    // Get ticket
    sem_wait(&order);
    sem_wait(&access);  // Get exclusive access
    sem_post(&order);   // Release ticket
    
    // Writing
    shared_data++;
    printf("Writer %d: Writing data = %d\n", id, shared_data);
    usleep(150000);
    
    sem_post(&access);
    
    return NULL;
}

int main() {
    printf("=== READER-WRITER (Fair Solution) ===\n\n");
    
    sem_init(&order, 0, 1);
    sem_init(&access, 0, 1);
    
    pthread_t readers[5], writers[3];
    int reader_ids[5] = {1, 2, 3, 4, 5};
    int writer_ids[3] = {1, 2, 3};
    
    // Interleave readers and writers
    for (int i = 0; i < 5; i++) {
        pthread_create(&readers[i], NULL, reader, &reader_ids[i]);
        if (i < 3) {
            pthread_create(&writers[i], NULL, writer, &writer_ids[i]);
        }
    }
    
    for (int i = 0; i < 5; i++) {
        pthread_join(readers[i], NULL);
        if (i < 3) {
            pthread_join(writers[i], NULL);
        }
    }
    
    printf("\n=== Complete. Final data: %d ===\n", shared_data);
    
    sem_destroy(&order);
    sem_destroy(&access);
    pthread_mutex_destroy(&read_mutex);
    
    return 0;
}
```

### 9.2 Dining Philosophers with Waiter

**Solution:** Central waiter controls fork allocation:

```c
// Maximum 4 philosophers can eat simultaneously
#define MAX_EATING 4

sem_t waiter;  // Limits concurrent philosophers

void* philosopher(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 3; i++) {
        printf("Philosopher %d: Thinking\n", id);
        usleep(1000000);
        
        // Ask waiter for permission
        sem_wait(&waiter);
        
        // Pick up forks
        sem_wait(&forks[id]);
        sem_wait(&forks[(id + 1) % NUM_PHILOSOPHERS]);
        
        // Eating
        printf("Philosopher %d: Eating\n", id);
        usleep(500000);
        
        // Put down forks
        sem_post(&forks[id]);
        sem_post(&forks[(id + 1) % NUM_PHILOSOPHERS]);
        
        // Inform waiter
        sem_post(&waiter);
    }
    
    return NULL;
}

// Initialize: sem_init(&waiter, 0, MAX_EATING);
```

### 9.3 Performance Comparison

```c
// File: performance_comparison.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <time.h>

#define ITERATIONS 1000000

int counter = 0;

// Method 1: No synchronization (wrong)
void* no_sync(void* arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        counter++;
    }
    return NULL;
}

// Method 2: Mutex
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
void* with_mutex(void* arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        pthread_mutex_lock(&mutex);
        counter++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

// Method 3: Semaphore
sem_t sem;
void* with_semaphore(void* arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        sem_wait(&sem);
        counter++;
        sem_post(&sem);
    }
    return NULL;
}

double measure_time(void* (*func)(void*), const char* name) {
    counter = 0;
    clock_t start = clock();
    
    pthread_t threads[2];
    pthread_create(&threads[0], NULL, func, NULL);
    pthread_create(&threads[1], NULL, func, NULL);
    
    pthread_join(threads[0], NULL);
    pthread_join(threads[1], NULL);
    
    clock_t end = clock();
    double elapsed = (double)(end - start) / CLOCKS_PER_SEC;
    
    printf("%s: %.3f seconds (counter=%d, expected=%d)\n", 
           name, elapsed, counter, ITERATIONS * 2);
    
    return elapsed;
}

int main() {
    printf("=== SYNCHRONIZATION PERFORMANCE COMPARISON ===\n");
    printf("Iterations per thread: %d\n\n", ITERATIONS);
    
    sem_init(&sem, 0, 1);
    
    measure_time(no_sync, "No Sync (WRONG)");
    measure_time(with_mutex, "Mutex");
    measure_time(with_semaphore, "Semaphore");
    
    sem_destroy(&sem);
    pthread_mutex_destroy(&mutex);
    
    return 0;
}
```

---

## 10. Quick Reference

### 10.1 Synchronization Primitives Cheat Sheet

```c
// PETERSON'S SOLUTION
volatile bool flag[2] = {false, false};
volatile int turn = 0;

// Process i:
flag[i] = true;
turn = j;
while (flag[j] && turn == j);
// Critical Section
flag[i] = false;

// MUTEX
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&mutex);
// Critical Section
pthread_mutex_unlock(&mutex);
pthread_mutex_destroy(&mutex);

// SEMAPHORE
sem_t sem;
sem_init(&sem, 0, initial_value);
sem_wait(&sem);  // P operation (decrement)
// Critical Section
sem_post(&sem);  // V operation (increment)
sem_destroy(&sem);

// CONDITION VARIABLE
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// Waiting:
pthread_mutex_lock(&mutex);
while (!condition) {
    pthread_cond_wait(&cond, &mutex);
}
pthread_mutex_unlock(&mutex);

// Signaling:
pthread_mutex_lock(&mutex);
// change condition
pthread_cond_signal(&cond);  // or broadcast
pthread_mutex_unlock(&mutex);
```

### 10.2 Problem Patterns

```c
// PRODUCER-CONSUMER
sem_t empty, full;
pthread_mutex_t mutex;

Producer:
  sem_wait(&empty);
  pthread_mutex_lock(&mutex);
  // add item
  pthread_mutex_unlock(&mutex);
  sem_post(&full);

Consumer:
  sem_wait(&full);
  pthread_mutex_lock(&mutex);
  // remove item
  pthread_mutex_unlock(&mutex);
  sem_post(&empty);

// READER-WRITER (Readers First)
pthread_mutex_t mutex;
sem_t write_sem;
int read_count = 0;

Reader:
  pthread_mutex_lock(&mutex);
  read_count++;
  if (read_count == 1) sem_wait(&write_sem);
  pthread_mutex_unlock(&mutex);
  // read
  pthread_mutex_lock(&mutex);
  read_count--;
  if (read_count == 0) sem_post(&write_sem);
  pthread_mutex_unlock(&mutex);

Writer:
  sem_wait(&write_sem);
  // write
  sem_post(&write_sem);

// DINING PHILOSOPHERS (Resource Ordering)
sem_t forks[5];

Philosopher i:
  first = min(i, (i+1)%5);
  second = max(i, (i+1)%5);
  sem_wait(&forks[first]);
  sem_wait(&forks[second]);
  // eat
  sem_post(&forks[first]);
  sem_post(&forks[second]);
```

### 10.3 Common Patterns

| Pattern | Use Case | Code |
|---------|----------|------|
| **Mutex** | Simple protection | `lock()` → work → `unlock()` |
| **Binary Semaphore** | Signaling | `wait()` → work → `post()` |
| **Counting Semaphore** | Resource pool | Init with N resources |
| **Condition Variable** | Wait/notify | `wait()` on condition, `signal()` when ready |

---

## 11. Practice Exercises

### Exercise 1: Modify Producer-Consumer
**Task:** Add priority items that skip the queue.

**Hint:** Use two buffers (normal and priority) with separate semaphores.

### Exercise 2: Sleeping Barber Problem
**Description:** Barber shop with waiting room. Barber sleeps when no customers.

**Requirements:**
- Barber waits for customers
- Customers wait if barber busy
- Limited waiting chairs

### Exercise 3: Cigarette Smokers Problem
**Description:** 3 smokers, 1 agent. Each smoker needs 3 ingredients, has 1.

**Requirements:**
- Agent places 2 random ingredients
- Smoker with 3rd ingredient smokes
- Coordinate using semaphores

### Exercise 4: Implement Reader-Writer with pthread_rwlock
**Task:** Use POSIX read-write locks instead of manual semaphores.

```c
pthread_rwlock_t rwlock;
pthread_rwlock_rdlock(&rwlock);  // Reader
pthread_rwlock_wrlock(&rwlock);  // Writer
pthread_rwlock_unlock(&rwlock);
```

---

## 12. Summary

### Key Concepts Mastered

✅ **Peterson's Solution:** Software-based mutual exclusion for 2 processes  
✅ **Producer-Consumer:** Buffer management with semaphores and condition variables  
✅ **Reader-Writer:** Concurrent read access with writer exclusion  
✅ **Dining Philosophers:** Deadlock prevention through resource ordering  

### Synchronization Hierarchy

```
Simple mutual exclusion
    ↓
Resource counting
    ↓
Complex coordination
    ↓
Deadlock prevention
```

### Best Practices

1. **Always unlock what you lock**
2. **Keep critical sections small**
3. **Use appropriate primitive for the problem**
4. **Test with multiple threads**
5. **Handle errors properly**
6. **Avoid busy waiting**
7. **Document synchronization strategy**

---

## 13. Additional Resources

### Books
- "Operating System Concepts" - Silberschatz, Galvin, Gagne
- "The Art of Multiprocessor Programming" - Herlihy, Shavit
- "Programming with POSIX Threads" - Butenhof

### Online Resources
- POSIX Threads Tutorial: https://computing.llnl.gov/tutorials/pthreads/
- Linux man pages: `man pthread_mutex_lock`, `man sem_wait`
- GDB Tutorial: https://www.gnu.org/software/gdb/documentation/

### Practice Platforms
- LeetCode (Concurrency problems)
- HackerRank (Threading challenges)
- Local experimentation with provided code

---

**Author:** OS Synchronization Expert  
**Version:** 2.0  
**Last Updated:** 2025  
**License:** Educational Use

---

*"In concurrent programming, what can go wrong, will go wrong. Synchronization is your safety net."* 📚 Table of Contents

1. [Introduction](#1-introduction)
2. [Peterson's Solution](#2-petersons-solution)
3. [Producer-Consumer Problem](#3-producer-consumer-problem)
4. [Reader-Writer Problem](#4-reader-writer-problem)
5. [Dining Philosophers Problem](#5-dining-philosophers-problem)
6. [Comparison & Selection Guide](#6-comparison--selection-guide)
7. [Compilation & Testing](#7-compilation--testing)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Introduction

### Classic Synchronization Problems

These problems demonstrate fundamental concepts in concurrent programming and process synchronization:

| Problem | Core Concept | Real-World Example |
|---------|-------------|-------------------|
| **Peterson's Solution** | Two-process mutual exclusion | Two CPUs sharing memory |
| **Producer-Consumer** | Bounded buffer management | Print queue, message queue |
| **Reader-Writer** | Shared data access control | Database, file systems |
| **Dining Philosophers** | Deadlock prevention | Resource allocation |

### Why Study These?

- **Understanding race conditions** and critical sections
- **Mastering synchronization primitives** (mutex, semaphore, condition variables)
- **Preventing deadlocks** and starvation
- **Building thread-safe applications**

---

## 2. Peterson's Solution

### 2.1 Overview

**Peterson's Algorithm** is a software-based solution for mutual exclusion between **two processes** without requiring special hardware instructions.

### 2.2 Algorithm Explained

**Key Variables:**
```c
bool flag[2] = {false, false};  // Interest in entering CS
int turn = 0;                    // Whose turn is it?
```

**Logic:**
```
Process i wants to enter critical section:
1. Set flag[i] = true          (I want to enter)
2. Set turn = j                (Give priority to other process)
3. Wait while (flag[j] && turn == j)  (Wait if other wants AND it's their turn)
4. Enter Critical Section
5. Set flag[i] = false         (I'm done)
```

### 2.3 Complete Implementation

```c
// File: peterson_solution.c
#include <stdio.h>
#include <pthread.h>
#include <stdbool.h>
#include <unistd.h>

#define NUM_ITERATIONS 5

// Peterson's algorithm variables
volatile bool flag[2] = {false, false};
volatile int turn = 0;
int shared_counter = 0;  // Shared resource

void* process(void* arg) {
    int id = *(int*)arg;
    int other = 1 - id;  // Other process ID
    
    for (int i = 0; i < NUM_ITERATIONS; i++) {
        // ENTRY SECTION (Peterson's Algorithm)
        flag[id] = true;           // I want to enter
        turn = other;              // Give priority to other
        
        // Wait while other wants to enter AND it's their turn
        while (flag[other] && turn == other) {
            // Busy wait
        }
        
        // === CRITICAL SECTION ===
        printf("Process %d: Entering critical section\n", id);
        int temp = shared_counter;
        usleep(100000);  // Simulate work
        shared_counter = temp + 1;
        printf("Process %d: Counter = %d\n", id, shared_counter);
        // === END CRITICAL SECTION ===
        
        // EXIT SECTION
        flag[id] = false;  // I'm done
        
        printf("Process %d: Exited critical section\n\n", id);
        usleep(50000);  // Some non-critical work
    }
    
    return NULL;
}

int main() {
    printf("=== PETERSON'S SOLUTION FOR MUTUAL EXCLUSION ===\n\n");
    printf("Two processes, %d iterations each\n", NUM_ITERATIONS);
    printf("Expected final counter: %d\n\n", NUM_ITERATIONS * 2);
    
    pthread_t threads[2];
    int ids[2] = {0, 1};
    
    // Create two processes
    pthread_create(&threads[0], NULL, process, &ids[0]);
    pthread_create(&threads[1], NULL, process, &ids[1]);
    
    // Wait for completion
    pthread_join(threads[0], NULL);
    pthread_join(threads[1], NULL);
    
    printf("=== COMPLETED ===\n");
    printf("Final counter: %d (Expected: %d)\n", 
           shared_counter, NUM_ITERATIONS * 2);
    
    return 0;
}
```

### 2.4 Properties

✅ **Mutual Exclusion:** Only one process in CS at a time  
✅ **Progress:** Selection of next process cannot be postponed indefinitely  
✅ **Bounded Waiting:** Process will eventually enter CS  
❌ **Limitation:** Only works for **2 processes**  
❌ **Busy Waiting:** Wastes CPU cycles

---

## 3. Producer-Consumer Problem

### 3.1 Problem Statement

**Scenario:** Producers generate items and place them in a bounded buffer. Consumers remove items from the buffer.

**Constraints:**
- Producer cannot add to full buffer
- Consumer cannot remove from empty buffer
- Only one thread can access buffer at a time

```
Producer Thread          Buffer [5 slots]          Consumer Thread
     │                   ┌─┬─┬─┬─┬─┐                    │
     ├─ Produce item     │X│X│ │ │ │                    │
     │                   └─┴─┴─┴─┴─┘                    │
     ├─ Add to buffer ──>│X│X│X│ │ │                    │
     │                   └─┴─┴─┴─┴─┘                    │
     │                        │                          │
     │                        │          Remove from ──>─┤
     │                   ┌─┬─┬─┬─┬─┐    buffer          │
     │                   │X│X│ │ │ │                    │
     │                   └─┴─┴─┴─┴─┘                    ├─ Consume item
```

### 3.2 Solution Using Semaphores

```c
// File: producer_consumer_semaphore.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>
#include <stdlib.h>

#define BUFFER_SIZE 5
#define NUM_ITEMS 10

int buffer[BUFFER_SIZE];
int in = 0;   // Producer index
int out = 0;  // Consumer index

sem_t empty;  // Count empty slots (initially BUFFER_SIZE)
sem_t full;   // Count full slots (initially 0)
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* producer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < NUM_ITEMS; i++) {
        int item = id * 100 + i;
        
        // Wait for empty slot
        sem_wait(&empty);
        
        // Lock buffer access
        pthread_mutex_lock(&mutex);
        
        // === CRITICAL SECTION: Add item ===
        buffer[in] = item;
        printf("Producer %d: Produced item %d at position %d\n", 
               id, item, in);
        in = (in + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);
        
        // Signal that buffer has one more full slot
        sem_post(&full);
        
        usleep(rand() % 100000);  // Simulate production time
    }
    
    printf("Producer %d: Finished producing\n", id);
    return NULL;
}

void* consumer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < NUM_ITEMS; i++) {
        // Wait for full slot
        sem_wait(&full);
        
        // Lock buffer access
        pthread_mutex_lock(&mutex);
        
        // === CRITICAL SECTION: Remove item ===
        int item = buffer[out];
        printf("Consumer %d: Consumed item %d from position %d\n", 
               id, item, out);
        out = (out + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);
        
        // Signal that buffer has one more empty slot
        sem_post(&empty);
        
        usleep(rand() % 150000);  // Simulate consumption time
    }
    
    printf("Consumer %d: Finished consuming\n", id);
    return NULL;
}

int main() {
    printf("=== PRODUCER-CONSUMER PROBLEM (Semaphore Solution) ===\n\n");
    printf("Buffer Size: %d\n", BUFFER_SIZE);
    printf("Items to produce/consume: %d per thread\n\n", NUM_ITEMS);
    
    srand(time(NULL));
    
    // Initialize semaphores
    sem_init(&empty, 0, BUFFER_SIZE);  // All slots empty
    sem_init(&full, 0, 0);              // No slots full
    
    pthread_t prod[2], cons[2];
    int prod_ids[2] = {1, 2};
    int cons_ids[2] = {1, 2};
    
    // Create producers and consumers
    for (int i = 0; i < 2; i++) {
        pthread_create(&prod[i], NULL, producer, &prod_ids[i]);
        pthread_create(&cons[i], NULL, consumer, &cons_ids[i]);
    }
    
    // Wait for completion
    for (int i = 0; i < 2; i++) {
        pthread_join(prod[i], NULL);
        pthread_join(cons[i], NULL);
    }
    
    printf("\n=== All production and consumption complete ===\n");
    
    // Cleanup
    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&mutex);
    
    return 0;
}
```

### 3.3 Solution Using Condition Variables

```c
// File: producer_consumer_condvar.c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdbool.h>

#define BUFFER_SIZE 5
#define NUM_ITEMS 10

int buffer[BUFFER_SIZE];
int count = 0;  // Number of items in buffer
int in = 0;
int out = 0;

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t not_full = PTHREAD_COND_INITIALIZER;
pthread_cond_t not_empty = PTHREAD_COND_INITIALIZER;

void* producer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < NUM_ITEMS; i++) {
        int item = id * 100 + i;
        
        pthread_mutex_lock(&mutex);
        
        // Wait while buffer is full
        while (count == BUFFER_SIZE) {
            printf("Producer %d: Buffer full, waiting...\n", id);
            pthread_cond_wait(&not_full, &mutex);
        }
        
        // Add item to buffer
        buffer[in] = item;
        printf("Producer %d: Produced item %d (count: %d)\n", 
               id, item, count + 1);
        in = (in + 1) % BUFFER_SIZE;
        count++;
        
        // Signal consumer that buffer is not empty
        pthread_cond_signal(&not_empty);
        
        pthread_mutex_unlock(&mutex);
        usleep(rand() % 100000);
    }
    
    printf("Producer %d: Finished\n", id);
    return NULL;
}

void* consumer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < NUM_ITEMS; i++) {
        pthread_mutex_lock(&mutex);
        
        // Wait while buffer is empty
        while (count == 0) {
            printf("Consumer %d: Buffer empty, waiting...\n", id);
            pthread_cond_wait(&not_empty, &mutex);
        }
        
        // Remove item from buffer
        int item = buffer[out];
        printf("Consumer %d: Consumed item %d (count: %d)\n", 
               id, item, count - 1);
        out = (out + 1) % BUFFER_SIZE;
        count--;
        
        // Signal producer that buffer is not full
        pthread_cond_signal(&not_full);
        
        pthread_mutex_unlock(&mutex);
        usleep(rand() % 150000);
    }
    
    printf("Consumer %d: Finished\n", id);
    return NULL;
}

int main() {
    printf("=== PRODUCER-CONSUMER (Condition Variables) ===\n\n");
    
    srand(time(NULL));
    
    pthread_t prod[2], cons[2];
    int prod_ids[2] = {1, 2};
    int cons_ids[2] = {1, 2};
    
    for (int i = 0; i < 2; i++) {
        pthread_create(&prod[i], NULL, producer, &prod_ids[i]);
        pthread_create(&cons[i], NULL, consumer, &cons_ids[i]);
    }
    
    for (int i = 0; i < 2; i++) {
        pthread_join(prod[i], NULL);
        pthread_join(cons[i], NULL);
    }
    
    printf("\n=== Complete ===\n");
    
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&not_full);
    pthread_cond_destroy(&not_empty);
    
    return 0;
}
```

---

## 4. Reader-Writer Problem

### 4.1 Problem Statement

**Scenario:** Multiple readers and writers access shared data. 

**Constraints:**
- Multiple readers can read simultaneously
- Only one writer can write at a time
- No reader can read while writer is writing

### 4.2 Readers-Preference Solution

**Priority:** Readers are given preference. Writers may starve.

```c
// File: reader_writer_readers_first.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

int shared_data = 0;
int read_count = 0;  // Number of active readers

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;  // Protects read_count
sem_t write_sem;  // Controls writer access

void* reader(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 3; i++) {
        // Entry section for reader
        pthread_mutex_lock(&mutex);
        read_count++;
        if (read_count == 1) {
            // First reader locks out writers
            sem_wait(&write_sem);
        }
        pthread_mutex_unlock(&mutex);
        
        // === READING ===
        printf("Reader %d: Reading data = %d (active readers: %d)\n", 
               id, shared_data, read_count);
        usleep(100000);  // Simulate reading
        
        // Exit section for reader
        pthread_mutex_lock(&mutex);
        read_count--;
        if (read_count == 0) {
            // Last reader unlocks writers
            sem_post(&write_sem);
        }
        pthread_mutex_unlock(&mutex);
        
        usleep(200000);  // Time between reads
    }
    
    printf("Reader %d: Finished\n", id);
    return NULL;
}

void* writer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 2; i++) {
        // Entry section for writer
        sem_wait(&write_sem);
        
        // === WRITING ===
        shared_data++;
        printf("Writer %d: Writing data = %d\n", id, shared_data);
        usleep(150000);  // Simulate writing
        
        // Exit section for writer
        sem_post(&write_sem);
        
        usleep(300000);  // Time between writes
    }
    
    printf("Writer %d: Finished\n", id);
    return NULL;
}

int main() {
    printf("=== READER-WRITER PROBLEM (Readers Preference) ===\n\n");
    
    sem_init(&write_sem, 0, 1);  // Binary semaphore
    
    pthread_t readers[3], writers[2];
    int reader_ids[3] = {1, 2, 3};
    int writer_ids[2] = {1, 2};
    
    // Create readers and writers
    for (int i = 0; i < 3; i++) {
        pthread_create(&readers[i], NULL, reader, &reader_ids[i]);
    }
    for (int i = 0; i < 2; i++) {
        pthread_create(&writers[i], NULL, writer, &writer_ids[i]);
    }
    
    // Wait for completion
    for (int i = 0; i < 3; i++) {
        pthread_join(readers[i], NULL);
    }
    for (int i = 0; i < 2; i++) {
        pthread_join(writers[i], NULL);
    }
    
    printf("\n=== Complete. Final data: %d ===\n", shared_data);
    
    sem_destroy(&write_sem);
    pthread_mutex_destroy(&mutex);
    
    return 0;
}
```

### 4.3 Writers-Preference Solution

**Priority:** Writers are given preference. Readers may be delayed.

```c
// File: reader_writer_writers_first.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

int shared_data = 0;
int read_count = 0;
int write_count = 0;

pthread_mutex_t read_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t write_mutex = PTHREAD_MUTEX_INITIALIZER;
sem_t read_sem;   // Blocks readers when writer waiting
sem_t write_sem;  // Controls actual writing

void* reader(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 3; i++) {
        // Entry section
        sem_wait(&read_sem);  // Wait if writers are waiting
        
        pthread_mutex_lock(&read_mutex);
        read_count++;
        if (read_count == 1) {
            sem_wait(&write_sem);  // First reader blocks writers
        }
        pthread_mutex_unlock(&read_mutex);
        
        sem_post(&read_sem);
        
        // === READING ===
        printf("Reader %d: Reading data = %d (readers: %d)\n", 
               id, shared_data, read_count);
        usleep(100000);
        
        // Exit section
        pthread_mutex_lock(&read_mutex);
        read_count--;
        if (read_count == 0) {
            sem_post(&write_sem);  // Last reader unblocks writers
        }
        pthread_mutex_unlock(&read_mutex);
        
        usleep(200000);
    }
    
    printf("Reader %d: Finished\n", id);
    return NULL;
}

void* writer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 2; i++) {
        // Entry section
        pthread_mutex_lock(&write_mutex);
        write_count++;
        if (write_count == 1) {
            sem_wait(&read_sem);  // First writer blocks new readers
        }
        pthread_mutex_unlock(&write_mutex);
        
        sem_wait(&write_sem);  // Wait for exclusive access
        
        // === WRITING ===
        shared_data++;
        printf("Writer %d: Writing data = %d (writers waiting: %d)\n", 
               id, shared_data, write_count);
        usleep(150000);
        
        sem_post(&write_sem);
        
        // Exit section
        pthread_mutex_lock(&write_mutex);
        write_count--;
        if (write_count == 0) {
            sem_post(&read_sem);  // Last writer unblocks readers
        }
        pthread_mutex_unlock(&write_mutex);
        
        usleep(300000);
    }
    
    printf("Writer %d: Finished\n", id);
    return NULL;
}

int main() {
    printf("=== READER-WRITER PROBLEM (Writers Preference) ===\n\n");
    
    sem_init(&read_sem, 0, 1);
    sem_init(&write_sem, 0, 1);
    
    pthread_t readers[3], writers[2];
    int reader_ids[3] = {1, 2, 3};
    int writer_ids[2] = {1, 2};
    
    for (int i = 0; i < 3; i++) {
        pthread_create(&readers[i], NULL, reader, &reader_ids[i]);
    }
    for (int i = 0; i < 2; i++) {
        pthread_create(&writers[i], NULL, writer, &writer_ids[i]);
    }
    
    for (int i = 0; i < 3; i++) {
        pthread_join(readers[i], NULL);
    }
    for (int i = 0; i < 2; i++) {
        pthread_join(writers[i], NULL);
    }
    
    printf("\n=== Complete. Final data: %d ===\n", shared_data);
    
    sem_destroy(&read_sem);
    sem_destroy(&write_sem);
    pthread_mutex_destroy(&read_mutex);
    pthread_mutex_destroy(&write_mutex);
    
    return 0;
}
```

---

## 5. Dining Philosophers Problem

### 5.1 Problem Statement

**Scenario:** 5 philosophers sit at a round table. Between each pair of philosophers is one fork. To eat, a philosopher needs both left and right forks.

```
        Fork 0
         │
    P0  ─┼─  P1
    │   Fork 1  │
  Fork 4    Fork 2
    │           │
    P4  ─────  P2
         │
       Fork 3
        P3
```

**Challenges:**
- **Deadlock:** All philosophers pick up left fork simultaneously
- **Starvation:** Some philosopher never gets both forks

### 5.2 Solution: Resource Hierarchy (Prevents Deadlock)

**Strategy:** Odd philosophers pick left fork first, even philosophers pick right fork first.

```c
// File: dining_philosophers.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define NUM_PHILOSOPHERS 5
#define THINKING 0
#define HUNGRY 1
#define EATING 2

sem_t forks[NUM_PHILOSOPHERS];
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int state[NUM_PHILOSOPHERS];

void test_philosopher(int id);
void pick_forks(int id);
void put_forks(int id);

void* philosopher(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 3; i++) {  // Each philosopher eats 3 times
        // THINKING
        printf("Philosopher %d: Thinking...\n", id);
        usleep(1000000);  // Think for 1 second
        
        // HUNGRY
        pick_forks(id);
        
        // EATING
        printf("Philosopher %d: Eating (iteration %d)\n", id, i + 1);
        usleep(500000);  // Eat for 0.5 seconds
        
        // FINISHED EATING
        put_forks(id);
    }
    
    printf("Philosopher %d: Finished all meals\n", id);
    return NULL;
}

void pick_forks(int id) {
    pthread_mutex_lock(&mutex);
    
    state[id] = HUNGRY;
    printf("Philosopher %d: Hungry, trying to pick forks\n", id);
    
    test_philosopher(id);
    
    pthread_mutex_unlock(&mutex);
    
    // Wait for both forks
    sem_wait(&forks[id]);
}

void put_forks(int id) {
    pthread_mutex_lock(&mutex);
    
    state[id] = THINKING;
    printf("Philosopher %d: Putting down forks\n", id);
    
    // Test left and right neighbors
    test_philosopher((id + NUM_PHILOSOPHERS - 1) % NUM_PHILOSOPHERS);  // Left
    test_philosopher((id + 1) % NUM_PHILOSOPHERS);  // Right
    
    pthread_mutex_unlock(&mutex);
}

void test_philosopher(int id) {
    int left = (id + NUM_PHILOSOPHERS - 1) % NUM_PHILOSOPHERS;
    int right = (id + 1) % NUM_PHILOSOPHERS;
    
    // Can eat if hungry and neighbors are not eating
    if (state[id] == HUNGRY && 
        state[left] != EATING && 
        state[right] != EATING) {
        
        state[id] = EATING;
        printf("Philosopher %d: Got both forks!\n", id);
        sem_post(&forks[id]);  // Signal that philosopher can start eating
    }
}

int main() {
    printf("=== DINING PHILOSOPHERS PROBLEM ===\n");
    printf("5 Philosophers, 5 Forks\n");
    printf("Each philosopher eats 3 times\n\n");
    
    pthread_t philosophers[NUM_PHILOSOPHERS];
    int ids[NUM_PHILOSOPHERS];
    
    // Initialize semaphores and states
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        sem_init(&forks[i], 0, 0);  // Initially 0 (blocking)
        state[i] = THINKING;
        ids[i] = i;
    }
    
    // Create philosopher threads
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_create(&philosophers[i], NULL, philosopher, &ids[i]);
    }
    
    // Wait for all philosophers to finish
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_join(philosophers[i], NULL);
    }
    
    printf("\n=== All philosophers finished dining ===\n");
    
    // Cleanup
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        sem_destroy(&forks[i]);
    }
    pthread_mutex_destroy(&mutex);
    
    return 0;
}
```

### 5.3 Alternative Solution: Resource Ordering

```c
// File: dining_philosophers_ordering.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define NUM_PHILOSOPHERS 5

sem_t forks[NUM_PHILOSOPHERS];

void* philosopher(void* arg) {
    int id = *(int*)arg;
    int left_fork = id;
    int right_fork = (id + 1) % NUM_PHILOSOPHERS;
    
    // Resource ordering: Pick lower numbered fork first
    int first_fork = (left_fork < right_fork) ? left_fork : right_fork;
    int second_fork = (left_fork < right_fork) ? right_fork : left_fork;
    
    for (int i = 0; i < 3; i++) {
        printf("Philosopher %d: Thinking...\n", id);
        usleep(1000000);
        
        // Pick forks in order (prevents circular wait)
        printf("Philosopher %d: Trying to pick fork %d\n", id, first_fork);
        sem_wait(&forks[first_fork]);
        printf("Philosopher %d: Picked fork %d\n", id, first_fork);
        
        printf("Philosopher %d: Trying to pick fork %d\n", id, second_fork);
        sem_wait(&forks[second_fork]);
        printf("Philosopher %d: Picked fork %d\n", id, second_fork);
        
        // Eating
        printf("Philosopher %d: EATING (meal %d/3)\n", id, i + 1);
        usleep(500000);
        
        // Put down forks
        sem_post(&forks[first_fork]);
        sem_post(&forks[second_fork]);
        printf("Philosopher %d: Put down both forks\n", id);
    }
    
    printf("Philosopher %d: Finished all meals\n", id);
    return NULL;
}

int main() {
    printf("=== DINING PHILOSOPHERS (Resource Ordering) ===\n\n");
    
    pthread_t philosophers[NUM_PHILOSOPHERS];
    int ids[NUM_PHILOSOPHERS];
    
    // Initialize fork semaphores (1 = available)
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        sem_init(&forks[i], 0, 1);
        ids[i] = i;
    }
    
    // Create philosophers
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_create(&philosophers[i], NULL, philosopher, &ids[i]);
    }
    
    // Wait for completion
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_join(philosophers[i], NULL);
    }
    
    printf("\n=== All philosophers satisfied ===\n");
    
    // Cleanup
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        sem_destroy(&forks[i]);
    }
    
    return 0;
}
```

---

## 6. Comparison & Selection Guide

### 6.1 Problem Comparison

| Problem | Key Challenge | Solution Approach | Potential Issues |
|---------|--------------|-------------------|------------------|
| **Peterson's** | 2-process mutual exclusion | Software algorithm | Busy waiting, 2 processes only |
| **Producer-Consumer** | Buffer management | Semaphores/Cond vars | Buffer overflow/underflow |
| **Reader-Writer** | Concurrent read access | Multiple locks | Starvation |
| **Dining Philosophers** | Resource deadlock | Resource ordering | Starvation, fairness |

### 6.2 When to Use What?

**Use Peterson's Solution when:**
- Exactly 2 processes
- No OS synchronization primitives available
- Learning/academic purposes

**Use Producer-Consumer pattern when:**
- Queue/buffer management
- Thread pools
- Event processing systems
- Message passing

**Use Reader-Writer pattern when:**
- Database systems
- Caching systems
- Shared configuration data
- File systems

**Use Dining Philosophers concepts when:**
- Multiple resource allocation
- Avoiding deadlock in complex systems
- Resource scheduling

### 6.3 Synchronization Primitive Selection

```
Simple mutual exclusion → Mutex
Resource counting → Semaphore
Wait/notify patterns → Condition Variables
2-process only → Peterson's Algorithm
```

---

##