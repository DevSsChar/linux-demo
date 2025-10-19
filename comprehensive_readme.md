# Operating System Practicals - Comprehensive Guide
## Process Management & Thread Synchronization

---

## 📚 Table of Contents

1. [Introduction](#introduction)
2. [Process Management](#process-management)
3. [Thread Synchronization Fundamentals](#thread-synchronization-fundamentals)
4. [Problem Solutions](#problem-solutions)
5. [Complete Code Examples](#complete-code-examples)
6. [Compilation & Execution](#compilation--execution)
7. [Troubleshooting](#troubleshooting)

---

## 1. Introduction

This guide covers two critical aspects of Operating Systems:

### **Part A: Process Management**
- Creating and managing child processes
- Process synchronization with wait/waitpid
- Process monitoring and control
- Resource management

### **Part B: Thread Synchronization**
- Race conditions and critical sections
- Mutex locks (pthread_mutex_t)
- Semaphores (sem_t)
- Condition variables (pthread_cond_t)
- Producer-Consumer problem
- Reader-Writer problem

---

## 2. Process Management

### 2.1 Core Concepts

#### **Process Creation: fork()**
```c
pid_t fork(void);
```
- Creates a duplicate of the calling process
- Returns 0 to child, child's PID to parent
- Both processes execute independently

#### **Process Synchronization: wait() and waitpid()**
```c
pid_t wait(int *status);
pid_t waitpid(pid_t pid, int *status, int options);
```
- Parent waits for child to complete
- Prevents zombie processes
- Retrieves exit status

#### **Process Monitoring Commands**

| Command | Purpose | Example |
|---------|---------|---------|
| `ps aux` | List all processes | `ps aux \| grep firefox` |
| `top` | Real-time process viewer | `top -u username` |
| `htop` | Enhanced process viewer | `htop` (interactive) |
| `pgrep` | Find process by name | `pgrep firefox` |
| `kill` | Terminate process | `kill -9 PID` |
| `nice` | Set process priority | `nice -n 10 command` |
| `renice` | Change priority | `renice -n 5 -p PID` |

### 2.2 Process States

```
┌─────────┐     fork()      ┌─────────┐
│ PARENT  │ ─────────────> │  CHILD  │
└─────────┘                 └─────────┘
     │                           │
     │ wait()                    │ exit()
     │                           │
     └───────────┬───────────────┘
                 ▼
            ┌─────────┐
            │ ZOMBIE  │ (if parent doesn't wait)
            └─────────┘
```

---

## 3. Thread Synchronization Fundamentals

### 3.1 Why Synchronization?

**Race Condition Example:**
```
Thread 1: Read tickets = 10
Thread 2: Read tickets = 10
Thread 1: tickets = 10 - 1 = 9
Thread 2: tickets = 10 - 1 = 9
Result: tickets = 9 (should be 8!)
```

### 3.2 Critical Section Problem

A **critical section** is code that accesses shared resources and must not be executed by more than one thread at a time.

**Requirements:**
1. **Mutual Exclusion**: Only one thread in critical section
2. **Progress**: If no thread is in CS, selection cannot be postponed
3. **Bounded Waiting**: Limit on waiting time

### 3.3 Synchronization Mechanisms

#### **A. Mutex (Mutual Exclusion)**

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// Usage
pthread_mutex_lock(&mutex);
// Critical Section
pthread_mutex_unlock(&mutex);
```

**Characteristics:**
- Binary lock (locked/unlocked)
- Ownership (only locker can unlock)
- Best for simple mutual exclusion

**Example Flow:**
```
Thread 1              Mutex                Thread 2
   │                   │                      │
   ├─ lock() ────────> LOCKED                 │
   │                   │                      │
   │  Critical Section │                      │
   │                   │                  ├── lock() (BLOCKED)
   │                   │                      │ (waiting...)
   ├─ unlock() ──────> UNLOCKED               │
   │                   │                      │
   │                   │ <──── lock() ────────┤
   │                   LOCKED                 │
   │                   │        Critical Section
   │                   │                      │
```

#### **B. Semaphore**

```c
sem_t semaphore;
sem_init(&semaphore, 0, initial_value);

// Usage
sem_wait(&semaphore);  // P operation (decrement)
// Critical Section
sem_post(&semaphore);  // V operation (increment)

sem_destroy(&semaphore);
```

**Types:**

1. **Binary Semaphore** (value: 0 or 1)
   - Similar to mutex
   - Used for mutual exclusion

2. **Counting Semaphore** (value: 0 to N)
   - Controls access to N resources
   - Example: Database connection pool

**Example: Resource Pool**
```
Semaphore = 3 (3 available resources)

Thread 1: sem_wait() → Semaphore = 2 ✓ (gets resource)
Thread 2: sem_wait() → Semaphore = 1 ✓ (gets resource)
Thread 3: sem_wait() → Semaphore = 0 ✓ (gets resource)
Thread 4: sem_wait() → BLOCKED (waiting...)
Thread 1: sem_post() → Semaphore = 1 (releases resource)
Thread 4: UNBLOCKED → Semaphore = 0 ✓ (gets resource)
```

#### **C. Condition Variables**

```c
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// Waiting thread
pthread_mutex_lock(&mutex);
while (condition_not_met) {
    pthread_cond_wait(&cond, &mutex);  // Releases mutex, waits
}
// Condition is met
pthread_mutex_unlock(&mutex);

// Signaling thread
pthread_mutex_lock(&mutex);
// Change condition
pthread_cond_signal(&cond);  // Wake one thread
// or pthread_cond_broadcast(&cond);  // Wake all threads
pthread_mutex_unlock(&mutex);
```

**Usage Pattern:**
```
Producer Thread          Consumer Thread
     │                        │
     ├─ lock(mutex)          │
     │                        │
     ├─ produce_item()       │
     │                        │
     ├─ signal(cond) ────────>│
     │                        ├─ lock(mutex)
     ├─ unlock(mutex)        │
     │                        ├─ wait(cond) ← (if buffer empty)
     │                        │
     │                        ├─ consume_item()
     │                        │
     │                        ├─ unlock(mutex)
```

### 3.4 Comparison Table

| Feature | Mutex | Semaphore | Condition Variable |
|---------|-------|-----------|-------------------|
| **Purpose** | Mutual exclusion | Resource counting | Thread coordination |
| **Value** | Binary (0/1) | Integer (0 to N) | None (signaling only) |
| **Ownership** | Yes | No | No |
| **Can signal others** | No | Yes | Yes |
| **Best for** | Protecting data | Limiting access | Wait/notify patterns |
| **Header** | pthread.h | semaphore.h | pthread.h |

---

## 4. Problem Solutions

### Problem Q-1: Process Management System

**Scenario:** University research lab with multiple data analysis tasks

**Requirements:**
1. Create child processes for independent jobs
2. Synchronize processes for proper execution order
3. Monitor and manage system resources

**Solution Approach:**
```
Main Process
    │
    ├─> Child 1: Read sensor data
    │       │
    │       └─> Wait for completion
    │
    ├─> Child 2: Process data
    │       │
    │       └─> Wait for completion
    │
    └─> Child 3: Save results
            │
            └─> Wait for completion
```

### Problem Q-2: Thread-Safe Ticket Booking

**Scenario:** Online ticket booking with concurrent users

**Requirements:**
1. Multiple threads booking simultaneously
2. Prevent race conditions
3. Ensure consistent ticket count

**Solution Approach:**

**Without Synchronization (WRONG):**
```c
// Race condition!
void* book_ticket(void* arg) {
    if (tickets > 0) {  // Thread 1 and 2 both see tickets = 1
        tickets--;      // Both decrement
        // Result: tickets = -1 (OVERBOOKING!)
    }
}
```

**With Mutex (CORRECT):**
```c
void* book_ticket(void* arg) {
    pthread_mutex_lock(&mutex);
    if (tickets > 0) {
        tickets--;  // Safe: only one thread at a time
    }
    pthread_mutex_unlock(&mutex);
}
```

**With Semaphore (CORRECT):**
```c
void* book_ticket(void* arg) {
    sem_wait(&semaphore);  // Wait for available ticket
    // Critical section
    tickets--;
    sem_post(&semaphore);  // Release
}
```

### Problem Q-3: System Process Management

**Scenario:** CPU-intensive programs slowing down system

**Steps:**

1. **Identify resource-heavy processes:**
```bash
# Show processes sorted by CPU
top
# Press Shift+P for CPU sort
# Press Shift+M for memory sort

# Or use ps
ps aux --sort=-%cpu | head -n 10
ps aux --sort=-%mem | head -n 10
```

2. **Check specific user's processes:**
```bash
ps aux | grep username
pgrep -u username
```

3. **Terminate process:**
```bash
# Graceful termination
kill PID

# Force kill
kill -9 PID
kill -SIGKILL PID

# Kill by name
pkill program_name
killall program_name
```

4. **Adjust priority:**
```bash
# Lower priority (higher nice value)
renice +10 -p PID

# Higher priority (requires root)
sudo renice -10 -p PID
```

---

## 5. Complete Code Examples

### 5.1 Process Management - Data Processing System

```c
// File: process_management.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <time.h>

void log_message(const char *task, const char *msg) {
    time_t now = time(NULL);
    printf("[%ld] [%s] PID=%d: %s\n", now, task, getpid(), msg);
}

void read_sensor_data() {
    log_message("READER", "Reading sensor data...");
    sleep(2);  // Simulate reading
    log_message("READER", "Data read complete");
}

void process_data() {
    log_message("PROCESSOR", "Processing data...");
    sleep(3);  // Simulate processing
    log_message("PROCESSOR", "Processing complete");
}

void save_results() {
    log_message("SAVER", "Saving results...");
    sleep(1);  // Simulate saving
    log_message("SAVER", "Results saved");
}

int main() {
    printf("=== Data Processing System ===\n\n");
    
    // Task 1: Read data
    pid_t pid1 = fork();
    if (pid1 == 0) {
        read_sensor_data();
        exit(0);
    }
    
    // Wait for reader to complete
    waitpid(pid1, NULL, 0);
    printf("\n--- Reader finished, starting processor ---\n\n");
    
    // Task 2: Process data (only after reading is done)
    pid_t pid2 = fork();
    if (pid2 == 0) {
        process_data();
        exit(0);
    }
    
    // Wait for processor to complete
    waitpid(pid2, NULL, 0);
    printf("\n--- Processor finished, starting saver ---\n\n");
    
    // Task 3: Save results (only after processing is done)
    pid_t pid3 = fork();
    if (pid3 == 0) {
        save_results();
        exit(0);
    }
    
    // Wait for saver to complete
    waitpid(pid3, NULL, 0);
    
    printf("\n=== All tasks completed successfully ===\n");
    return 0;
}
```

### 5.2 Thread Synchronization - Ticket Booking System

#### Part 1: Race Condition (Without Synchronization)

```c
// File: race_condition.c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int tickets = 10;  // Shared resource

void* book_ticket(void* arg) {
    int user_id = *(int*)arg;
    
    // RACE CONDITION: Multiple threads access shared variable
    if (tickets > 0) {
        printf("User %d: Checking... %d tickets available\n", 
               user_id, tickets);
        usleep(100000);  // Simulate processing delay
        
        tickets--;  // UNSAFE: Multiple threads can execute this
        
        printf("User %d: Booked! Remaining: %d\n", 
               user_id, tickets);
    } else {
        printf("User %d: Sorry, sold out!\n", user_id);
    }
    
    return NULL;
}

int main() {
    printf("=== WITHOUT SYNCHRONIZATION (Race Condition) ===\n");
    printf("Initial tickets: %d\n\n", tickets);
    
    pthread_t threads[15];
    int user_ids[15];
    
    // Create 15 users trying to book 10 tickets
    for (int i = 0; i < 15; i++) {
        user_ids[i] = i + 1;
        pthread_create(&threads[i], NULL, book_ticket, &user_ids[i]);
    }
    
    // Wait for all threads
    for (int i = 0; i < 15; i++) {
        pthread_join(threads[i], NULL);
    }
    
    printf("\nFinal tickets: %d\n", tickets);
    printf("Expected: 0 or negative (showing overbooking)\n");
    
    return 0;
}
```

#### Part 2: Thread-Safe with Mutex

```c
// File: mutex_solution.c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

int tickets = 10;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* book_ticket_safe(void* arg) {
    int user_id = *(int*)arg;
    
    pthread_mutex_lock(&mutex);  // LOCK: Enter critical section
    
    if (tickets > 0) {
        printf("User %d: Checking... %d tickets available\n", 
               user_id, tickets);
        usleep(100000);  // Simulate processing
        
        tickets--;  // SAFE: Only one thread at a time
        
        printf("User %d: Booked! Remaining: %d\n", 
               user_id, tickets);
    } else {
        printf("User %d: Sorry, sold out!\n", user_id);
    }
    
    pthread_mutex_unlock(&mutex);  // UNLOCK: Exit critical section
    
    return NULL;
}

int main() {
    printf("=== WITH MUTEX (Thread-Safe) ===\n");
    printf("Initial tickets: %d\n\n", tickets);
    
    pthread_t threads[15];
    int user_ids[15];
    
    for (int i = 0; i < 15; i++) {
        user_ids[i] = i + 1;
        pthread_create(&threads[i], NULL, book_ticket_safe, &user_ids[i]);
    }
    
    for (int i = 0; i < 15; i++) {
        pthread_join(threads[i], NULL);
    }
    
    printf("\nFinal tickets: %d\n", tickets);
    printf("Expected: 0 (no overbooking)\n");
    
    pthread_mutex_destroy(&mutex);
    return 0;
}
```

#### Part 3: Semaphore Solution

```c
// File: semaphore_solution.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

int tickets = 10;
sem_t semaphore;

void* book_ticket_semaphore(void* arg) {
    int user_id = *(int*)arg;
    
    printf("User %d: Attempting to book...\n", user_id);
    
    sem_wait(&semaphore);  // Wait for available resource
    
    if (tickets > 0) {
        printf("User %d: Booking ticket... (%d left)\n", 
               user_id, tickets);
        usleep(100000);
        tickets--;
        printf("User %d: Success! Now %d tickets left\n", 
               user_id, tickets);
    } else {
        printf("User %d: Failed - sold out\n", user_id);
    }
    
    sem_post(&semaphore);  // Release resource
    
    return NULL;
}

int main() {
    printf("=== WITH SEMAPHORE ===\n");
    printf("Initial tickets: %d\n\n", tickets);
    
    // Initialize semaphore (binary: 0 = shared, 1 = initial value)
    sem_init(&semaphore, 0, 1);
    
    pthread_t threads[15];
    int user_ids[15];
    
    for (int i = 0; i < 15; i++) {
        user_ids[i] = i + 1;
        pthread_create(&threads[i], NULL, book_ticket_semaphore, &user_ids[i]);
    }
    
    for (int i = 0; i < 15; i++) {
        pthread_join(threads[i], NULL);
    }
    
    printf("\nFinal tickets: %d\n", tickets);
    
    sem_destroy(&semaphore);
    return 0;
}
```

#### Part 4: Condition Variables (Parent-Child Coordination)

```c
// File: condition_variable.c
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <stdbool.h>

int tickets = 10;
bool data_validated = false;

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

void* parent_validator(void* arg) {
    sleep(2);  // Simulate validation time
    
    pthread_mutex_lock(&mutex);
    
    printf("PARENT: Validating booking data...\n");
    sleep(1);
    
    data_validated = true;
    printf("PARENT: Data validated! Signaling child...\n");
    
    pthread_cond_signal(&cond);  // Wake up child thread
    
    pthread_mutex_unlock(&mutex);
    
    return NULL;
}

void* child_booker(void* arg) {
    int user_id = *(int*)arg;
    
    pthread_mutex_lock(&mutex);
    
    printf("CHILD %d: Waiting for parent validation...\n", user_id);
    
    // Wait until parent validates
    while (!data_validated) {
        pthread_cond_wait(&cond, &mutex);  // Releases mutex, waits for signal
    }
    
    printf("CHILD %d: Validation received! Booking ticket...\n", user_id);
    
    if (tickets > 0) {
        tickets--;
        printf("CHILD %d: Ticket booked! Remaining: %d\n", user_id, tickets);
    } else {
        printf("CHILD %d: No tickets available\n", user_id);
    }
    
    pthread_mutex_unlock(&mutex);
    
    return NULL;
}

int main() {
    printf("=== PARENT-CHILD COORDINATION (Condition Variables) ===\n\n");
    
    pthread_t parent, child;
    int user_id = 1;
    
    // Create child thread (waits for parent)
    pthread_create(&child, NULL, child_booker, &user_id);
    
    // Create parent thread (validates data)
    pthread_create(&parent, NULL, parent_validator, NULL);
    
    pthread_join(parent, NULL);
    pthread_join(child, NULL);
    
    printf("\nFinal tickets: %d\n", tickets);
    
    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond);
    
    return 0;
}
```

### 5.3 Advanced: Producer-Consumer Problem

```c
// File: producer_consumer.c
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define BUFFER_SIZE 5

int buffer[BUFFER_SIZE];
int in = 0, out = 0;

sem_t empty;  // Count empty slots
sem_t full;   // Count full slots
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* producer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 10; i++) {
        int item = id * 100 + i;
        
        sem_wait(&empty);  // Wait for empty slot
        pthread_mutex_lock(&mutex);
        
        // Produce item
        buffer[in] = item;
        printf("Producer %d: Produced %d at position %d\n", id, item, in);
        in = (in + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);
        sem_post(&full);  // Increment full count
        
        usleep(100000);
    }
    
    return NULL;
}

void* consumer(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 10; i++) {
        sem_wait(&full);  // Wait for full slot
        pthread_mutex_lock(&mutex);
        
        // Consume item
        int item = buffer[out];
        printf("Consumer %d: Consumed %d from position %d\n", id, item, out);
        out = (out + 1) % BUFFER_SIZE;
        
        pthread_mutex_unlock(&mutex);
        sem_post(&empty);  // Increment empty count
        
        usleep(150000);
    }
    
    return NULL;
}

int main() {
    printf("=== PRODUCER-CONSUMER PROBLEM ===\n\n");
    
    sem_init(&empty, 0, BUFFER_SIZE);  // All slots empty initially
    sem_init(&full, 0, 0);  // No slots full initially
    
    pthread_t prod[2], cons[2];
    int ids[2] = {1, 2};
    
    // Create producers and consumers
    for (int i = 0; i < 2; i++) {
        pthread_create(&prod[i], NULL, producer, &ids[i]);
        pthread_create(&cons[i], NULL, consumer, &ids[i]);
    }
    
    // Wait for completion
    for (int i = 0; i < 2; i++) {
        pthread_join(prod[i], NULL);
        pthread_join(cons[i], NULL);
    }
    
    printf("\n=== Production-Consumption Complete ===\n");
    
    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&mutex);
    
    return 0;
}
```

---

## 6. Compilation & Execution

### Compilation Commands

```bash
# Process management
gcc process_management.c -o process_mgmt

# Thread programs (need -pthread flag)
gcc race_condition.c -o race -pthread
gcc mutex_solution.c -o mutex -pthread
gcc semaphore_solution.c -o semaphore -pthread
gcc condition_variable.c -o condvar -pthread
gcc producer_consumer.c -o prodcons -pthread
```

### Execution

```bash
# Process management
./process_mgmt

# Thread programs
./race           # Shows race condition
./mutex          # Shows mutex solution
./semaphore      # Shows semaphore solution
./condvar        # Shows condition variable
./prodcons       # Producer-consumer demo
```

### Monitoring

```bash
# Monitor processes
ps aux | grep process_mgmt
top

# Monitor threads
ps -T -p $(pgrep mutex)
htop  # Press 'H' to show threads
```

---

## 7. Troubleshooting

### Common Errors

#### 1. "undefined reference to pthread_create"
**Solution:** Add `-pthread` flag
```bash
gcc program.c -o program -pthread
```

#### 2. "sem_init implicit declaration"
**Solution:** Include semaphore header
```c
#include <semaphore.h>
```

#### 3. Deadlock
**Symptoms:** Program hangs
**Causes:**
- Forgot to unlock mutex
- Circular wait for locks

**Solution:**
```c
// Always unlock in same function
pthread_mutex_lock(&mutex);
// ... do work ...
pthread_mutex_unlock(&mutex);  // Don't forget!
```

#### 4. Race condition still occurs
**Check:**
- Is critical section properly locked?
- Are all threads using the same mutex?
- Is there shared data outside critical section?

### Performance Tips

1. **Minimize critical section:** Lock only what's necessary
```c
// Bad: Large critical section
pthread_mutex_lock(&mutex);
expensive_computation();  // Doesn't need lock
shared_data++;
pthread_mutex_unlock(&mutex);

// Good: Small critical section
expensive_computation();  // Outside lock
pthread_mutex_lock(&mutex);
shared_data++;
pthread_mutex_unlock(&mutex);
```

2. **Use appropriate synchronization:**
- Simple counter → Mutex
- Resource pool → Semaphore
- Wait/notify → Condition variable

3. **Avoid busy waiting:**
```c
// Bad: Busy wait (wastes CPU)
while (!ready) {
    // Spin
}

// Good: Condition variable (sleeps)
pthread_mutex_lock(&mutex);
while (!ready) {
    pthread_cond_wait(&cond, &mutex);
}
pthread_mutex_unlock(&mutex);
```

---

## 8. Quick Reference

### System Calls Cheat Sheet

```c
// Process Management
fork()                    // Create child process
wait(NULL)               // Wait for any child
waitpid(pid, &status, 0) // Wait for specific child
exit(status)             // Terminate process
getpid()                 // Get process ID
getppid()                // Get parent process ID

// Thread Management
pthread_create(&thread, NULL, function, arg)  // Create thread
pthread_join(thread, NULL)                    // Wait for thread
pthread_exit(NULL)                            // Exit thread

// Mutex
pthread_mutex_lock(&mutex)      // Lock
pthread_mutex_unlock(&mutex)    // Unlock
pthread_mutex_destroy(&mutex)   // Clean up

// Semaphore
sem_init(&sem, 0, value)   // Initialize
sem_wait(&sem)             // P operation (wait/decrement)
sem_post(&sem)             // V operation (signal/increment)
sem_destroy(&sem)          // Clean up

// Condition Variable
pthread_cond_wait(&cond, &mutex)      // Wait
pthread_cond_signal(&cond)            // Wake one
pthread_cond_broadcast(&cond)         // Wake all
```

### Compilation Flags

```bash
-pthread        # Link pthread library
-o filename     # Output file name
-Wall           # Show all warnings
-g              # Include debug info
-O2             # Optimize level 2
```

---

## 9. Summary

### Key Takeaways

1. **Process Management:**
   - Use fork() to create child processes
   - Always wait() for children to avoid zombies
   - Monitor with ps, top, htop

2. **Thread Synchronization:**
   - **Mutex:** Binary lock for mutual exclusion
   - **Semaphore:** Counter for resource management
   - **Condition Variable:** For thread coordination

3. **Best Practices:**
   - Always unlock what you lock
   - Keep critical sections small
   - Use appropriate synchronization primitive
   - Test with multiple threads/processes
   - Handle errors properly

### Learning Path

```
1. Understand race conditions
   ↓
2. Learn mutex basics
   ↓
3. Master semaphores
   ↓
4. Explore condition variables
   ↓
5. Solve classic problems (Producer-Consumer, Reader-Writer)
   ↓
6. Build real-world applications
```

---

## 10. Additional Resources

### Man Pages
```bash
man fork
man pthread_create
man pthread_mutex_lock
man sem_init
man pthread_cond_wait
```

### Further Reading
- POSIX Threads Programming: https://computing.llnl.gov/tutorials/pthreads/
- Linux Process Management
- Operating System Concepts (Silberschatz)
- The Linux Programming Interface (Michael Kerrisk)

---

**Author:** OS Practical Expert  
**Last Updated:** 2025  
**License:** Educational Use

---

*For questions or issues, consult your lab instructor or system administrator.*