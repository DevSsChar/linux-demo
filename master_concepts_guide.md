# Master Guide: Process Management & Thread Synchronization
## Everything You Need to Solve Similar Problems

---

## 📚 Table of Contents

1. [Process Management Concepts](#1-process-management-concepts)
2. [Thread Fundamentals](#2-thread-fundamentals)
3. [Synchronization Mechanisms Deep Dive](#3-synchronization-mechanisms-deep-dive)
4. [Problem-Solving Patterns](#4-problem-solving-patterns)
5. [Classic Problems & Solutions](#5-classic-problems--solutions)
6. [Common Pitfalls & How to Avoid Them](#6-common-pitfalls--how-to-avoid-them)
7. [Debugging Techniques](#7-debugging-techniques)
8. [Practice Problems](#8-practice-problems)

---

## 1. Process Management Concepts

### 1.1 Understanding fork()

**What it does:** Creates an exact copy of the calling process.

**Return Values:**
- **< 0**: Fork failed (error)
- **= 0**: You're in the child process
- **> 0**: You're in the parent (value is child's PID)

#### Basic Pattern:
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        // Error handling
        perror("Fork failed");
        return 1;
    }
    else if (pid == 0) {
        // Child process code
        printf("I'm the child, PID=%d\n", getpid());
        // Do child work here
        return 0;  // Child exits
    }
    else {
        // Parent process code
        printf("I'm the parent, child PID=%d\n", pid);
        wait(NULL);  // Wait for child
        printf("Child finished\n");
    }
    
    return 0;
}
```

**Key Points:**
- After fork(), both processes execute the next line
- Variables are copied (not shared)
- File descriptors are shared
- Each process has its own memory space

#### Memory Visualization:
```
BEFORE FORK:              AFTER FORK:
┌─────────────┐          ┌─────────────┐     ┌─────────────┐
│   Parent    │          │   Parent    │     │    Child    │
│   PID=100   │   fork() │   PID=100   │     │   PID=101   │
│   x = 10    │  ──────> │   x = 10    │     │   x = 10    │
└─────────────┘          └─────────────┘     └─────────────┘
                                │                    │
                         If x++ in parent    If x++ in child
                                │                    │
                                ▼                    ▼
                           x = 11              x = 11
                         (independent!)      (independent!)
```

---

### 1.2 Process Synchronization: wait() Family

#### wait() - Wait for any child
```c
#include <sys/wait.h>

int status;
pid_t finished_pid = wait(&status);  // Blocks until ANY child exits

// Check how child exited
if (WIFEXITED(status)) {
    int exit_code = WEXITSTATUS(status);
    printf("Child exited with code %d\n", exit_code);
}
if (WIFSIGNALED(status)) {
    int signal = WTERMSIG(status);
    printf("Child killed by signal %d\n", signal);
}
```

#### waitpid() - Wait for specific child
```c
pid_t child1 = fork();
pid_t child2 = fork();

// Wait for specific child
int status;
waitpid(child2, &status, 0);  // Wait for child2 specifically

// Options:
// 0        - Block until child exits
// WNOHANG  - Return immediately if child hasn't exited
// WUNTRACED - Also return if child is stopped
```

#### Practical Example: Sequential Task Execution
```c
// Task must execute in order: A -> B -> C

void execute_sequential_tasks() {
    // Task A
    pid_t pid_a = fork();
    if (pid_a == 0) {
        printf("Task A: Starting\n");
        sleep(2);
        printf("Task A: Done\n");
        exit(0);
    }
    waitpid(pid_a, NULL, 0);  // Wait for A to finish
    
    // Task B (only after A finishes)
    pid_t pid_b = fork();
    if (pid_b == 0) {
        printf("Task B: Starting\n");
        sleep(2);
        printf("Task B: Done\n");
        exit(0);
    }
    waitpid(pid_b, NULL, 0);  // Wait for B to finish
    
    // Task C (only after B finishes)
    pid_t pid_c = fork();
    if (pid_c == 0) {
        printf("Task C: Starting\n");
        sleep(2);
        printf("Task C: Done\n");
        exit(0);
    }
    waitpid(pid_c, NULL, 0);
}
```

#### Pattern: Parallel Tasks with Final Synchronization
```c
void execute_parallel_tasks() {
    pid_t pids[3];
    
    // Start all tasks in parallel
    for (int i = 0; i < 3; i++) {
        pids[i] = fork();
        if (pids[i] == 0) {
            printf("Task %d: Working\n", i);
            sleep(2);
            exit(0);
        }
    }
    
    // Wait for all to complete
    for (int i = 0; i < 3; i++) {
        waitpid(pids[i], NULL, 0);
        printf("Task %d: Completed\n", i);
    }
}
```

---

### 1.3 Zombie and Orphan Processes

#### Zombie Process
**Definition:** Child has exited but parent hasn't called wait()

```c
void create_zombie() {
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child exits immediately
        printf("Child: Exiting\n");
        exit(0);  // Child is now zombie
    }
    else {
        // Parent doesn't call wait()!
        printf("Parent: Not waiting for child\n");
        sleep(30);  // Child remains zombie for 30 seconds
        
        // Check: ps aux | grep defunct
        
        wait(NULL);  // Finally clean up zombie
    }
}
```

**How to identify:**
```bash
ps aux | grep Z          # Z state = zombie
ps aux | grep defunct    # Shows <defunct>
```

**How to fix:**
- Parent calls wait()
- If parent won't wait, kill parent (zombie disappears)

#### Orphan Process
**Definition:** Parent exits before child

```c
void create_orphan() {
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child continues running
        sleep(5);
        printf("Child: My parent is now %d (init)\n", getppid());
        // Child is adopted by init (PID 1)
    }
    else {
        // Parent exits immediately
        printf("Parent: Exiting\n");
        exit(0);  // Child becomes orphan
    }
}
```

**What happens:**
- init/systemd (PID 1) adopts the orphan
- init automatically reaps (waits for) orphans

---

### 1.4 Process Monitoring & Control

#### Finding Processes
```bash
# By name
ps aux | grep program_name
pgrep program_name        # Just PIDs
pgrep -l program_name     # PIDs with names

# By user
ps aux | grep username
ps -u username
pgrep -u username

# By CPU/Memory usage
ps aux --sort=-%cpu       # Highest CPU first
ps aux --sort=-%mem       # Highest memory first
top                       # Real-time
htop                      # Better interface
```

#### Process Tree
```bash
pstree                    # All processes
pstree -p                 # With PIDs
pstree -p PID            # Subtree for PID
```

#### Killing Processes
```c
#include <signal.h>

// From code
kill(pid, SIGTERM);       // Graceful (15)
kill(pid, SIGKILL);       // Force (9)
kill(pid, SIGSTOP);       // Pause
kill(pid, SIGCONT);       // Resume
```

```bash
# From terminal
kill PID                  # Default: SIGTERM
kill -9 PID              # Force kill
kill -STOP PID           # Pause
kill -CONT PID           # Resume

killall program_name     # Kill all instances
pkill program_name       # Same as killall
```

#### Process Priority
```bash
# Nice values: -20 (highest) to 19 (lowest)
nice -n 10 ./program     # Start with low priority
renice +5 -p PID         # Lower priority
sudo renice -5 -p PID    # Raise priority (needs root)
```

---

## 2. Thread Fundamentals

### 2.1 Creating Threads

**Basic Pattern:**
```c
#include <pthread.h>

void* thread_function(void* arg) {
    int id = *(int*)arg;
    printf("Thread %d: Running\n", id);
    // Do work here
    return NULL;
}

int main() {
    pthread_t thread;
    int thread_id = 1;
    
    // Create thread
    pthread_create(&thread, NULL, thread_function, &thread_id);
    
    // Wait for thread to finish
    pthread_join(thread, NULL);
    
    return 0;
}
```

**Compile:** Always use `-pthread` flag
```bash
gcc program.c -o program -pthread
```

### 2.2 Multiple Threads Pattern
```c
#define NUM_THREADS 5

void* worker(void* arg) {
    int id = *(int*)arg;
    printf("Thread %d working\n", id);
    sleep(1);
    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];
    int thread_ids[NUM_THREADS];
    
    // Create all threads
    for (int i = 0; i < NUM_THREADS; i++) {
        thread_ids[i] = i;
        pthread_create(&threads[i], NULL, worker, &thread_ids[i]);
    }
    
    // Wait for all threads
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    
    return 0;
}
```

### 2.3 Passing Data to Threads

#### Method 1: Simple Integer
```c
void* thread_func(void* arg) {
    int value = *(int*)arg;
    printf("Received: %d\n", value);
    return NULL;
}

int data = 42;
pthread_create(&thread, NULL, thread_func, &data);
```

#### Method 2: Structure
```c
struct ThreadData {
    int id;
    char name[50];
    int* shared_counter;
};

void* thread_func(void* arg) {
    struct ThreadData* data = (struct ThreadData*)arg;
    printf("Thread %d: %s\n", data->id, data->name);
    (*data->shared_counter)++;
    return NULL;
}

struct ThreadData data = {1, "Worker", &counter};
pthread_create(&thread, NULL, thread_func, &data);
```

#### Method 3: Return Value
```c
void* thread_func(void* arg) {
    int* result = malloc(sizeof(int));
    *result = 42;
    return result;  // Return pointer
}

int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, thread_func, NULL);
    
    void* return_value;
    pthread_join(thread, &return_value);
    
    int result = *(int*)return_value;
    printf("Thread returned: %d\n", result);
    free(return_value);
}
```

---

## 3. Synchronization Mechanisms Deep Dive

### 3.1 Mutex (pthread_mutex_t)

**Purpose:** Mutual exclusion - only one thread in critical section

#### Basic Usage Pattern:
```c
#include <pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int shared_counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&mutex);      // LOCK
        shared_counter++;                 // Critical section
        pthread_mutex_unlock(&mutex);    // UNLOCK
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;
    
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    
    printf("Counter: %d\n", shared_counter);  // Should be 2000000
    pthread_mutex_destroy(&mutex);
    return 0;
}
```

#### When to Use Mutex:
✅ Protecting shared data structures
✅ Updating shared counters
✅ Modifying shared variables
✅ Accessing shared files

#### Mutex Best Practices:
```c
// ✓ GOOD: Small critical section
expensive_calculation();  // Outside lock
pthread_mutex_lock(&mutex);
shared_data = result;     // Quick operation
pthread_mutex_unlock(&mutex);

// ✗ BAD: Large critical section
pthread_mutex_lock(&mutex);
expensive_calculation();  // Holding lock too long!
shared_data = result;
pthread_mutex_unlock(&mutex);

// ✓ GOOD: Always unlock
pthread_mutex_lock(&mutex);
if (error) {
    pthread_mutex_unlock(&mutex);  // Don't forget!
    return ERROR;
}
shared_data++;
pthread_mutex_unlock(&mutex);

// ✓ GOOD: Check return values
if (pthread_mutex_lock(&mutex) != 0) {
    perror("Lock failed");
    return ERROR;
}
```

#### Common Mutex Patterns:

**Pattern 1: Protecting a Counter**
```c
pthread_mutex_t counter_lock = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

void increment_counter() {
    pthread_mutex_lock(&counter_lock);
    counter++;
    pthread_mutex_unlock(&counter_lock);
}

int get_counter() {
    pthread_mutex_lock(&counter_lock);
    int value = counter;  // Read also needs protection!
    pthread_mutex_unlock(&counter_lock);
    return value;
}
```

**Pattern 2: Protecting a Queue**
```c
pthread_mutex_t queue_lock = PTHREAD_MUTEX_INITIALIZER;
Queue* shared_queue;

void enqueue(Item item) {
    pthread_mutex_lock(&queue_lock);
    queue_add(shared_queue, item);
    pthread_mutex_unlock(&queue_lock);
}

Item dequeue() {
    pthread_mutex_lock(&queue_lock);
    Item item = queue_remove(shared_queue);
    pthread_mutex_unlock(&queue_lock);
    return item;
}
```

---

### 3.2 Semaphore (sem_t)

**Purpose:** Control access to N resources (counting semaphore)

#### Basic Usage:
```c
#include <semaphore.h>

sem_t semaphore;

// Initialize
sem_init(&semaphore, 0, 3);  // 0=shared between threads, 3=initial value

// Use
sem_wait(&semaphore);   // Decrement (wait if 0)
// Critical section
sem_post(&semaphore);   // Increment

// Cleanup
sem_destroy(&semaphore);
```

#### Binary Semaphore (Like Mutex):
```c
sem_t binary_sem;
sem_init(&binary_sem, 0, 1);  // Initial value 1

void* worker(void* arg) {
    sem_wait(&binary_sem);      // Lock
    // Critical section
    printf("Working\n");
    sem_post(&binary_sem);      // Unlock
    return NULL;
}
```

#### Counting Semaphore (Resource Pool):
```c
#define MAX_CONNECTIONS 5

sem_t connection_pool;
sem_init(&connection_pool, 0, MAX_CONNECTIONS);

void* user_request(void* arg) {
    int user_id = *(int*)arg;
    
    printf("User %d: Waiting for connection...\n", user_id);
    sem_wait(&connection_pool);  // Wait for available connection
    
    printf("User %d: Got connection!\n", user_id);
    sleep(2);  // Use connection
    printf("User %d: Releasing connection\n", user_id);
    
    sem_post(&connection_pool);  // Release connection
    return NULL;
}

// With 10 users and 5 connections:
// - First 5 users get connections immediately
// - Users 6-10 wait until someone releases
```

#### Semaphore vs Mutex:

| Feature | Mutex | Semaphore |
|---------|-------|-----------|
| **Value** | Binary (locked/unlocked) | Integer (0 to N) |
| **Ownership** | Yes (only locker unlocks) | No (any thread can signal) |
| **Use Case** | Mutual exclusion | Resource counting |
| **Example** | Protecting data | Connection pool |

#### When to Use Semaphore:
✅ Limited resource pool (DB connections, file handles)
✅ Producer-consumer with bounded buffer
✅ Controlling concurrent access count
✅ Signaling between threads

---

### 3.3 Condition Variables (pthread_cond_t)

**Purpose:** Thread coordination - wait for condition to become true

#### Basic Pattern:
```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t condition = PTHREAD_COND_INITIALIZER;
bool ready = false;

// Waiting thread
void* waiter(void* arg) {
    pthread_mutex_lock(&mutex);
    
    while (!ready) {  // Always use while, not if!
        pthread_cond_wait(&condition, &mutex);
        // Automatically releases mutex and waits
        // Re-acquires mutex when woken up
    }
    
    printf("Condition met!\n");
    pthread_mutex_unlock(&mutex);
    return NULL;
}

// Signaling thread
void* signaler(void* arg) {
    sleep(2);  // Do some work
    
    pthread_mutex_lock(&mutex);
    ready = true;  // Change condition
    pthread_cond_signal(&condition);  // Wake one waiter
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

#### Why "while" not "if"?
```c
// ✗ WRONG: Using if
pthread_mutex_lock(&mutex);
if (!ready) {
    pthread_cond_wait(&cond, &mutex);  // Spurious wakeup possible!
}
pthread_mutex_unlock(&mutex);

// ✓ CORRECT: Using while
pthread_mutex_lock(&mutex);
while (!ready) {  // Re-check after wakeup
    pthread_cond_wait(&cond, &mutex);
}
pthread_mutex_unlock(&mutex);
```

#### Signal vs Broadcast:
```c
pthread_cond_signal(&cond);     // Wakes ONE waiting thread
pthread_cond_broadcast(&cond);  // Wakes ALL waiting threads
```

#### Practical Example: Task Queue
```c
typedef struct {
    Task* tasks[100];
    int count;
    pthread_mutex_t mutex;
    pthread_cond_t not_empty;
} TaskQueue;

// Producer adds task
void add_task(TaskQueue* q, Task* task) {
    pthread_mutex_lock(&q->mutex);
    
    q->tasks[q->count++] = task;
    
    pthread_cond_signal(&q->not_empty);  // Wake one consumer
    pthread_mutex_unlock(&q->mutex);
}

// Consumer gets task
Task* get_task(TaskQueue* q) {
    pthread_mutex_lock(&q->mutex);
    
    while (q->count == 0) {  // Wait if queue empty
        pthread_cond_wait(&q->not_empty, &q->mutex);
    }
    
    Task* task = q->tasks[--q->count];
    
    pthread_mutex_unlock(&q->mutex);
    return task;
}
```

---

## 4. Problem-Solving Patterns

### Pattern 1: Protecting Shared Counter

**Problem:** Multiple threads incrementing same variable

```c
// WITHOUT synchronization (WRONG)
int counter = 0;
void* increment() {
    for (int i = 0; i < 100000; i++) {
        counter++;  // RACE CONDITION!
    }
}

// WITH mutex (CORRECT)
int counter = 0;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void* increment() {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&lock);
        counter++;
        pthread_mutex_unlock(&lock);
    }
}
```

---

### Pattern 2: Producer-Consumer (Bounded Buffer)

**Problem:** Producers add items, consumers remove items

```c
#define BUFFER_SIZE 10

int buffer[BUFFER_SIZE];
int in = 0, out = 0, count = 0;

sem_t empty;  // Count empty slots
sem_t full;   // Count full slots
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void init() {
    sem_init(&empty, 0, BUFFER_SIZE);  // All empty
    sem_init(&full, 0, 0);              // None full
}

void* producer(void* arg) {
    int item = produce_item();
    
    sem_wait(&empty);  // Wait for empty slot
    pthread_mutex_lock(&mutex);
    
    buffer[in] = item;
    in = (in + 1) % BUFFER_SIZE;
    count++;
    
    pthread_mutex_unlock(&mutex);
    sem_post(&full);  // Signal full slot
}

void* consumer(void* arg) {
    sem_wait(&full);  // Wait for full slot
    pthread_mutex_lock(&mutex);
    
    int item = buffer[out];
    out = (out + 1) % BUFFER_SIZE;
    count--;
    
    pthread_mutex_unlock(&mutex);
    sem_post(&empty);  // Signal empty slot
    
    consume_item(item);
}
```

---

### Pattern 3: Reader-Writer Problem

**Problem:** Multiple readers OK, but writers need exclusive access

```c
int readers = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t write_lock = PTHREAD_MUTEX_INITIALIZER;

void* reader(void* arg) {
    // Entry section
    pthread_mutex_lock(&mutex);
    readers++;
    if (readers == 1) {
        pthread_mutex_lock(&write_lock);  // First reader locks writers
    }
    pthread_mutex_unlock(&mutex);
    
    // Reading (critical section)
    read_data();
    
    // Exit section
    pthread_mutex_lock(&mutex);
    readers--;
    if (readers == 0) {
        pthread_mutex_unlock(&write_lock);  // Last reader unlocks writers
    }
    pthread_mutex_unlock(&mutex);
}

void* writer(void* arg) {
    pthread_mutex_lock(&write_lock);  // Exclusive access
    
    write_data();
    
    pthread_mutex_unlock(&write_lock);
}
```

---

### Pattern 4: Barrier Synchronization

**Problem:** All threads must reach a point before any continues

```c
pthread_mutex_t barrier_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t barrier_cond = PTHREAD_COND_INITIALIZER;
int barrier_count = 0;
int total_threads = 5;

void barrier_wait() {
    pthread_mutex_lock(&barrier_mutex);
    
    barrier_count++;
    
    if (barrier_count == total_threads) {
        // Last thread arrives
        barrier_count = 0;  // Reset for reuse
        pthread_cond_broadcast(&barrier_cond);  // Wake all
    } else {
        // Wait for others
        while (barrier_count != 0) {
            pthread_cond_wait(&barrier_cond, &barrier_mutex);
        }
    }
    
    pthread_mutex_unlock(&barrier_mutex);
}

void* worker(void* arg) {
    printf("Phase 1\n");
    barrier_wait();  // All threads wait here
    
    printf("Phase 2\n");  // Only after all reach barrier
    barrier_wait();
    
    printf("Phase 3\n");
}
```

---

## 5. Classic Problems & Solutions

### 5.1 Dining Philosophers

**Problem:** 5 philosophers, 5 forks, need 2 forks to eat

```c
#define N 5
pthread_mutex_t forks[N];

void* philosopher(void* arg) {
    int id = *(int*)arg;
    int left = id;
    int right = (id + 1) % N;
    
    while (1) {
        // Think
        printf("Phil %d: Thinking\n", id);
        sleep(1);
        
        // Pick up forks (avoid deadlock by ordering)
        if (id % 2 == 0) {
            pthread_mutex_lock(&forks[left]);
            pthread_mutex_lock(&forks[right]);
        } else {
            pthread_mutex_lock(&forks[right]);
            pthread_mutex_lock(&forks[left]);
        }
        
        // Eat
        printf("Phil %d: Eating\n", id);
        sleep(2);
        
        // Put down forks
        pthread_mutex_unlock(&forks[left]);
        pthread_mutex_unlock(&forks[right]);
    }
}
```

---

### 5.2 Sleeping Barber

**Problem:** Barber sleeps if no customers, customers wait if barber busy

```c
#define CHAIRS 5

sem_t customers;  // Waiting customers
sem_t barber;     // Barber ready
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int waiting = 0;

void* barber_thread(void* arg) {
    while (1) {
        sem_wait(&customers);  // Sleep if no customers
        
        pthread_mutex_lock(&mutex);
        waiting--;
        pthread_mutex_unlock(&mutex);
        
        sem_post(&barber);  // Signal ready
        
        cut_hair();
    }
}

void* customer_thread(void* arg) {
    pthread_mutex_lock(&mutex);
    
    if (waiting < CHAIRS) {
        waiting++;
        pthread_mutex_unlock(&mutex);
        
        sem_post(&customers);  // Wake barber
        sem_wait(&barber);     // Wait for barber
        
        get_haircut();
    } else {
        pthread_mutex_unlock(&mutex);
        printf("No chairs, leaving\n");
    }
}
```

---

## 6. Common Pitfalls & How to Avoid Them

### 6.1 Deadlock

**What:** Two threads waiting for each other forever

```c
// ✗ DEADLOCK EXAMPLE
pthread_mutex_t lock1, lock2;

// Thread 1
pthread_mutex_lock(&lock1);
sleep(1);  // Give thread 2 time to lock lock2
pthread_mutex_lock(&lock2);  // Waits forever!

// Thread 2
pthread_mutex_lock(&lock2);
sleep(1);
pthread_mutex_lock(&lock1);  // Waits forever!

// ✓ SOLUTION 1: Lock ordering
// Both threads acquire locks in same order
pthread_mutex_lock(&lock1);  // Always lock1 first
pthread_mutex_lock(&lock2);  // Then lock2

// ✓ SOLUTION 2: Try-lock
if (pthread_mutex_trylock(&lock2) != 0) {
    pthread_mutex_unlock(&lock1);  // Give up lock1
    sleep(1);                       // Wait and retry
}
```

---

### 6.2 Race Condition

**What:** Multiple threads accessing shared data without synchronization

```c
// ✗ RACE CONDITION
int balance = 100;

void* withdraw(void* arg) {
    if (balance >= 50) {      // Thread 1 checks
        sleep(1);              // Thread 2 also checks
        balance -= 50;         // Both withdraw!
    }
}
// Result: balance = 0 (should reject second withdrawal)

// ✓ SOLUTION: Atomic operation
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

void* withdraw_safe(void* arg) {
    pthread_mutex_lock(&lock);
    if (balance >= 50) {
        balance -= 50;
    }
    pthread_mutex_unlock(&lock);
}
```

---

### 6.3 Forgetting to Unlock

**What:** Lock acquired but never released

```c
// ✗ BAD: Lock not released on error
pthread_mutex_lock(&lock);
if (error_condition) {
    return ERROR;  // Lock still held! Deadlock!
}
pthread_mutex_unlock(&lock);

// ✓ GOOD: Always unlock
pthread_mutex_lock(&lock);
if (error_condition) {
    pthread_mutex_unlock(&lock);  // Unlock before return
    return ERROR;
}
pthread_mutex_unlock(&lock);

// ✓ BETTER: Use goto for cleanup
pthread_mutex_lock(&lock);
if (error_condition) {
    goto cleanup;
}
// Do work
cleanup:
    pthread_mutex_unlock(&lock);
    return result;
```

---

### 6.4 Priority Inversion

**What:** Low-priority thread holds lock needed by high-priority thread

```c
// Problem occurs when:
// 1. Low-priority thread locks mutex
// 2. High-priority thread waits for mutex
// 3. Medium-priority thread preempts low-priority
// 4. High-priority can't run (waiting for low-priority to unlock)

// ✓ SOLUTION: Priority inheritance
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);

pthread_mutex_t mutex;
pthread_mutex_init(&mutex, &attr);
```

---

## 7. Debugging Techniques

### 7.1 Detecting Race Conditions

```c
// Add delays to expose races
void* thread_func(void* arg) {
    if (shared_var > 0) {
        usleep(100000);  // Force context switch
        shared_var--;    // Race exposed!
    }
}

// Use ThreadSanitizer (compiler flag)
// gcc -fsanitize=thread program.c -pthread
```

### 7.2 Debugging Deadlocks

```bash
# Find threads
ps -T -p PID

# Use gdb
gdb ./program
(gdb) run
^C  # When deadlocked
(gdb) info threads
(gdb) thread 1
(gdb) bt  # Backtrace - shows what it's waiting for
```

### 7.3 Adding Debug Prints

```c
#define DEBUG 1

#if DEBUG
#define debug_print(fmt, ...) \
    fprintf(stderr, "[%ld] %s:%d: " fmt "\n", \
            pthread_self(), __FILE__, __LINE__, ##__VA_ARGS__)
#else
#define debug_print(fmt, ...)
#endif

void* thread_func(void* arg) {
    debug_print("Thread started");
    pthread_mutex_lock(&mutex);
    debug_print("Lock acquired");
    // Work
    pthread_mutex_unlock(&mutex);
    debug_print("Lock released");
    return NULL;
}
```

### 7.4 Valgrind for Thread Errors

```bash
# Check for thread errors
valgrind --tool=helgrind ./program

# Check for memory leaks in threads
valgrind --leak-check=full ./program
```

---

## 8. Practice Problems

### Problem 1: Bank Account System

**Requirements:**
- Multiple threads deposit/withdraw simultaneously
- No race conditions
- Prevent negative balance

**Solution Template:**
```c
typedef struct {
    int balance;
    pthread_mutex_t lock;
} BankAccount;

void init_account(BankAccount* acc, int initial) {
    acc->balance = initial;
    pthread_mutex_init(&acc->lock, NULL);
}

int deposit(BankAccount* acc, int amount) {
    pthread_mutex_lock(&acc->lock);
    acc->balance += amount;
    int new_balance = acc->balance;
    pthread_mutex_unlock(&acc->lock);
    return new_balance;
}

int withdraw(BankAccount* acc, int amount) {
    pthread_mutex_lock(&acc->lock);
    
    if (acc->balance >= amount) {
        acc->balance -= amount;
        pthread_mutex_unlock(&acc->lock);
        return 1;  // Success
    }
    
    pthread_mutex_unlock(&acc->lock);
    return 0;  // Insufficient funds
}

int get_balance(BankAccount* acc) {
    pthread_mutex_lock(&acc->lock);
    int bal = acc->balance;
    pthread_mutex_unlock(&acc->lock);
    return bal;
}
```

---

### Problem 2: Thread Pool

**Requirements:**
- Fixed number of worker threads
- Task queue
- Workers pick tasks from queue

**Solution:**
```c
#define NUM_WORKERS 4
#define QUEUE_SIZE 100

typedef struct {
    void (*function)(void*);
    void* arg;
} Task;

typedef struct {
    Task tasks[QUEUE_SIZE];
    int head, tail, count;
    pthread_mutex_t lock;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
    int shutdown;
} ThreadPool;

void pool_init(ThreadPool* pool) {
    pool->head = pool->tail = pool->count = 0;
    pool->shutdown = 0;
    pthread_mutex_init(&pool->lock, NULL);
    pthread_cond_init(&pool->not_empty, NULL);
    pthread_cond_init(&pool->not_full, NULL);
}

void pool_add_task(ThreadPool* pool, void (*func)(void*), void* arg) {
    pthread_mutex_lock(&pool->lock);
    
    // Wait if queue full
    while (pool->count == QUEUE_SIZE && !pool->shutdown) {
        pthread_cond_wait(&pool->not_full, &pool->lock);
    }
    
    if (pool->shutdown) {
        pthread_mutex_unlock(&pool->lock);
        return;
    }
    
    // Add task
    pool->tasks[pool->tail].function = func;
    pool->tasks[pool->tail].arg = arg;
    pool->tail = (pool->tail + 1) % QUEUE_SIZE;
    pool->count++;
    
    pthread_cond_signal(&pool->not_empty);
    pthread_mutex_unlock(&pool->lock);
}

void* worker_thread(void* arg) {
    ThreadPool* pool = (ThreadPool*)arg;
    
    while (1) {
        pthread_mutex_lock(&pool->lock);
        
        // Wait for task
        while (pool->count == 0 && !pool->shutdown) {
            pthread_cond_wait(&pool->not_empty, &pool->lock);
        }
        
        if (pool->shutdown && pool->count == 0) {
            pthread_mutex_unlock(&pool->lock);
            break;
        }
        
        // Get task
        Task task = pool->tasks[pool->head];
        pool->head = (pool->head + 1) % QUEUE_SIZE;
        pool->count--;
        
        pthread_cond_signal(&pool->not_full);
        pthread_mutex_unlock(&pool->lock);
        
        // Execute task
        (task.function)(task.arg);
    }
    
    return NULL;
}

void pool_shutdown(ThreadPool* pool) {
    pthread_mutex_lock(&pool->lock);
    pool->shutdown = 1;
    pthread_cond_broadcast(&pool->not_empty);
    pthread_mutex_unlock(&pool->lock);
}
```

---

### Problem 3: Parking Lot System

**Requirements:**
- N parking spots
- Cars enter/exit
- Display available spots
- No overbooking

**Solution:**
```c
typedef struct {
    int total_spots;
    int available;
    sem_t spots;
    pthread_mutex_t lock;
} ParkingLot;

void parking_init(ParkingLot* lot, int spots) {
    lot->total_spots = spots;
    lot->available = spots;
    sem_init(&lot->spots, 0, spots);
    pthread_mutex_init(&lot->lock, NULL);
}

void* car_enter(void* arg) {
    ParkingLot* lot = (ParkingLot*)arg;
    int car_id = pthread_self() % 1000;
    
    printf("Car %d: Waiting for spot...\n", car_id);
    
    sem_wait(&lot->spots);  // Wait for available spot
    
    pthread_mutex_lock(&lot->lock);
    lot->available--;
    printf("Car %d: Parked! Available: %d/%d\n", 
           car_id, lot->available, lot->total_spots);
    pthread_mutex_unlock(&lot->lock);
    
    sleep(rand() % 5 + 1);  // Stay parked
    
    pthread_mutex_lock(&lot->lock);
    lot->available++;
    printf("Car %d: Leaving! Available: %d/%d\n", 
           car_id, lot->available, lot->total_spots);
    pthread_mutex_unlock(&lot->lock);
    
    sem_post(&lot->spots);  // Release spot
    
    return NULL;
}

int main() {
    ParkingLot lot;
    parking_init(&lot, 5);  // 5 spots
    
    pthread_t cars[20];
    
    // 20 cars trying to park in 5 spots
    for (int i = 0; i < 20; i++) {
        pthread_create(&cars[i], NULL, car_enter, &lot);
        usleep(100000);
    }
    
    for (int i = 0; i < 20; i++) {
        pthread_join(cars[i], NULL);
    }
    
    return 0;
}
```

---

### Problem 4: Print Server

**Requirements:**
- Multiple users submit print jobs
- One printer (sequential processing)
- Queue jobs if printer busy

**Solution:**
```c
typedef struct {
    char filename[100];
    int user_id;
    int pages;
} PrintJob;

typedef struct {
    PrintJob queue[50];
    int count;
    pthread_mutex_t lock;
    pthread_cond_t has_jobs;
    int running;
} PrintServer;

void server_init(PrintServer* server) {
    server->count = 0;
    server->running = 1;
    pthread_mutex_init(&server->lock, NULL);
    pthread_cond_init(&server->has_jobs, NULL);
}

void submit_job(PrintServer* server, PrintJob job) {
    pthread_mutex_lock(&server->lock);
    
    server->queue[server->count++] = job;
    printf("Job submitted: %s (%d pages) from user %d\n", 
           job.filename, job.pages, job.user_id);
    
    pthread_cond_signal(&server->has_jobs);
    pthread_mutex_unlock(&server->lock);
}

void* printer_thread(void* arg) {
    PrintServer* server = (PrintServer*)arg;
    
    while (1) {
        pthread_mutex_lock(&server->lock);
        
        while (server->count == 0 && server->running) {
            pthread_cond_wait(&server->has_jobs, &server->lock);
        }
        
        if (!server->running && server->count == 0) {
            pthread_mutex_unlock(&server->lock);
            break;
        }
        
        // Get job
        PrintJob job = server->queue[0];
        for (int i = 0; i < server->count - 1; i++) {
            server->queue[i] = server->queue[i + 1];
        }
        server->count--;
        
        pthread_mutex_unlock(&server->lock);
        
        // Print (simulate)
        printf("Printing: %s (%d pages)...\n", job.filename, job.pages);
        sleep(job.pages);  // 1 second per page
        printf("Completed: %s\n", job.filename);
    }
    
    return NULL;
}
```

---

## 9. Debugging Checklist

### Before Running:
- [ ] Compiled with `-pthread` flag?
- [ ] All mutexes initialized?
- [ ] All semaphores initialized?
- [ ] Condition variables initialized?

### During Development:
- [ ] Every lock has a matching unlock?
- [ ] Unlock happens on all code paths (including errors)?
- [ ] Using `while` (not `if`) with condition variables?
- [ ] Critical sections as small as possible?
- [ ] No deadlock potential (check lock ordering)?

### Testing:
- [ ] Test with multiple threads (stress test)
- [ ] Add sleep() to expose race conditions
- [ ] Run with Valgrind/Helgrind
- [ ] Check for memory leaks
- [ ] Monitor with ps/top during execution

### Common Error Messages:
```
"Segmentation fault"
→ Check: uninitialized mutex/semaphore, accessing freed memory

"Deadlock detected"  
→ Check: lock ordering, forgot to unlock

"Resource temporarily unavailable"
→ Check: too many threads, system limit reached

"Invalid argument"
→ Check: mutex/semaphore not initialized
```

---

## 10. Performance Tips

### Tip 1: Reduce Lock Contention
```c
// ✗ BAD: Lock for entire loop
pthread_mutex_lock(&lock);
for (int i = 0; i < 1000000; i++) {
    shared_data[i]++;
}
pthread_mutex_unlock(&lock);

// ✓ GOOD: Lock per operation (if necessary)
for (int i = 0; i < 1000000; i++) {
    pthread_mutex_lock(&lock);
    shared_data[i]++;
    pthread_mutex_unlock(&lock);
}

// ✓ BETTER: Use local variable
int local_data[1000000];
for (int i = 0; i < 1000000; i++) {
    local_data[i] = shared_data[i] + 1;
}
pthread_mutex_lock(&lock);
memcpy(shared_data, local_data, sizeof(local_data));
pthread_mutex_unlock(&lock);
```

### Tip 2: Use Read-Write Locks for Read-Heavy Workloads
```c
pthread_rwlock_t rwlock;
pthread_rwlock_init(&rwlock, NULL);

// Multiple readers can hold lock simultaneously
pthread_rwlock_rdlock(&rwlock);
read_data();
pthread_rwlock_unlock(&rwlock);

// Writer gets exclusive access
pthread_rwlock_wrlock(&rwlock);
write_data();
pthread_rwlock_unlock(&rwlock);
```

### Tip 3: Lock-Free Algorithms (Advanced)
```c
#include <stdatomic.h>

atomic_int counter = 0;

void increment() {
    atomic_fetch_add(&counter, 1);  // No lock needed!
}
```

---

## 11. Quick Reference Tables

### System Calls Summary

| Category | Function | Purpose | Return |
|----------|----------|---------|--------|
| **Process** | `fork()` | Create child | PID/0/-1 |
| | `wait(NULL)` | Wait any child | Child PID |
| | `waitpid(pid,&s,0)` | Wait specific | Child PID |
| | `exit(n)` | Terminate | No return |
| | `getpid()` | Get own PID | PID |
| | `getppid()` | Get parent PID | PID |
| **Thread** | `pthread_create` | Create thread | 0/error |
| | `pthread_join` | Wait thread | 0/error |
| | `pthread_exit` | Exit thread | No return |
| **Mutex** | `pthread_mutex_lock` | Acquire lock | 0/error |
| | `pthread_mutex_unlock` | Release lock | 0/error |
| | `pthread_mutex_trylock` | Try acquire | 0/EBUSY |
| **Semaphore** | `sem_init(&s,0,n)` | Initialize | 0/-1 |
| | `sem_wait(&s)` | P (decrement) | 0/-1 |
| | `sem_post(&s)` | V (increment) | 0/-1 |
| | `sem_destroy(&s)` | Cleanup | 0/-1 |
| **Condition** | `pthread_cond_wait` | Wait signal | 0/error |
| | `pthread_cond_signal` | Wake one | 0/error |
| | `pthread_cond_broadcast` | Wake all | 0/error |

---

### Decision Tree: Which Synchronization to Use?

```
Do you need to protect shared data?
│
├─ YES: Do multiple operations need to be atomic?
│   │
│   ├─ YES: Use MUTEX
│   │        Example: Updating account balance
│   │
│   └─ NO: Is it read-heavy with occasional writes?
│            └─ Use READ-WRITE LOCK
│
└─ NO: Do you need to limit concurrent access to N resources?
    │
    ├─ YES: Use COUNTING SEMAPHORE
    │        Example: Connection pool
    │
    └─ NO: Do threads need to wait for a condition?
             └─ Use CONDITION VARIABLE
                  Example: Wait for queue to have items
```

---

### Common Patterns Cheat Sheet

```c
// 1. SIMPLE MUTEX PROTECTION
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_lock(&lock);
shared_data++;
pthread_mutex_unlock(&lock);

// 2. SEMAPHORE RESOURCE POOL
sem_t pool;
sem_init(&pool, 0, N);  // N resources
sem_wait(&pool);        // Get resource
// Use resource
sem_post(&pool);        // Release

// 3. CONDITION VARIABLE WAIT
pthread_mutex_lock(&mutex);
while (!condition) {
    pthread_cond_wait(&cond, &mutex);
}
// Condition is true
pthread_mutex_unlock(&mutex);

// 4. PRODUCER-CONSUMER
// Producer:
sem_wait(&empty);
pthread_mutex_lock(&mutex);
add_to_buffer();
pthread_mutex_unlock(&mutex);
sem_post(&full);

// Consumer:
sem_wait(&full);
pthread_mutex_lock(&mutex);
remove_from_buffer();
pthread_mutex_unlock(&mutex);
sem_post(&empty);

// 5. SEQUENTIAL PROCESS EXECUTION
pid_t p1 = fork();
if (p1 == 0) { task1(); exit(0); }
waitpid(p1, NULL, 0);

pid_t p2 = fork();
if (p2 == 0) { task2(); exit(0); }
waitpid(p2, NULL, 0);
```

---

## 12. Interview Questions & Answers

### Q1: What's the difference between process and thread?

**Answer:**
- **Process:** Independent execution unit with own memory space
- **Thread:** Lightweight execution unit within a process, shares memory

**Key Differences:**
| Feature | Process | Thread |
|---------|---------|--------|
| Memory | Separate | Shared |
| Creation | fork() (expensive) | pthread_create() (cheap) |
| Communication | IPC (pipes, sockets) | Shared variables |
| Context switch | Slower | Faster |

---

### Q2: What is a race condition? Give an example.

**Answer:**
When multiple threads access shared data simultaneously and the result depends on execution order.

**Example:**
```c
// Race condition
if (tickets > 0) {     // Thread 1 checks: true
    // Context switch
                       // Thread 2 checks: true
    tickets--;         // Thread 1 decrements
    tickets--;         // Thread 2 decrements
                       // Result: tickets = -1 (OVERBOOKING!)
}
```

---

### Q3: Explain deadlock and how to prevent it.

**Answer:**

**Deadlock:** Two or more threads waiting for each other forever.

**Example:**
```
Thread 1: Holds A, wants B
Thread 2: Holds B, wants A
→ Both wait forever
```

**Prevention:**
1. **Lock ordering:** Always acquire locks in same order
2. **Timeout:** Use trylock with timeout
3. **Lock hierarchy:** Assign priorities to locks
4. **Avoid nested locks:** Acquire only one lock at a time

---

### Q4: When would you use semaphore instead of mutex?

**Answer:**

**Use Mutex when:**
- Protecting shared data
- Need ownership (same thread locks/unlocks)
- Binary state (locked/unlocked)

**Use Semaphore when:**
- Controlling access to N resources (counting)
- Signaling between threads (no ownership)
- Producer-consumer synchronization

**Example:**
```c
// Database connection pool with 10 connections
sem_t pool;
sem_init(&pool, 0, 10);  // 10 connections available

// Get connection
sem_wait(&pool);  // Decrement count
use_connection();
sem_post(&pool);  // Increment count
```

---

### Q5: What happens if you forget to unlock a mutex?

**Answer:**

**Immediate effect:** Other threads waiting for that mutex will block forever (deadlock)

**How to find:**
- Use Valgrind with helgrind
- Enable debug assertions
- Code review checklist

**Prevention:**
```c
// Always unlock on all paths
pthread_mutex_lock(&lock);
if (error) {
    pthread_mutex_unlock(&lock);  // Don't forget!
    return ERROR;
}
result = process();
pthread_mutex_unlock(&lock);
return result;
```

---

## 13. Real-World Applications

### Application 1: Web Server
- **Threads:** Handle multiple client requests
- **Mutex:** Protect access log file
- **Semaphore:** Limit concurrent connections
- **Condition Variable:** Thread pool task queue

### Application 2: Database System
- **Process:** Separate processes for queries
- **Mutex:** Protect table modifications
- **RW Lock:** Allow multiple readers
- **Semaphore:** Connection pool management

### Application 3: Video Game
- **Threads:** Rendering, physics, AI
- **Mutex:** Protect game state
- **Condition Variable:** Frame synchronization
- **Atomic Operations:** Score updates

---

## 14. Final Checklist for Solving Any Problem

### Step 1: Understand Requirements
- [ ] What needs to run concurrently?
- [ ] What data is shared?
- [ ] What's the execution order?
- [ ] What are the constraints?

### Step 2: Choose Approach
- [ ] Process or thread?
- [ ] Which synchronization mechanism?
- [ ] How many workers needed?

### Step 3: Design Solution
- [ ] Identify critical sections
- [ ] Plan lock acquisition order
- [ ] Handle edge cases (empty, full, errors)

### Step 4: Implement
- [ ] Initialize all synchronization objects
- [ ] Add locks around critical sections
- [ ] Unlock on all code paths
- [ ] Add error handling

### Step 5: Test
- [ ] Test with 1 thread (should work)
- [ ] Test with many threads (check race conditions)
- [ ] Add delays to expose timing issues
- [ ] Run with debugging tools

### Step 6: Optimize
- [ ] Minimize critical sections
- [ ] Reduce lock contention
- [ ] Profile for bottlenecks

---

## 15. Additional Resources

### Man Pages (Essential Reading)
```bash
man fork
man pthread_create
man pthread_mutex_lock
man sem_wait
man pthread_cond_wait
```

### Recommended Books
- "Operating System Concepts" - Silberschatz
- "The Linux Programming Interface" - Michael Kerrisk
- "Programming with POSIX Threads" - David R. Butenhof

### Online Resources
- POSIX Threads Tutorial: https://computing.llnl.gov/tutorials/pthreads/
- Linux man pages: https://man7.org/
- Stack Overflow: Search for specific issues

### Tools
- **Valgrind:** Memory and thread error detection
- **GDB:** Debugging deadlocks
- **Helgrind:** Thread error detector
- **strace:** Trace system calls

---

## Summary

You now have a complete reference for:

✅ **Process Management**
- fork(), wait(), waitpid()
- Zombie and orphan processes
- Process monitoring and control

✅ **Thread Synchronization**
- Mutex (mutual exclusion)
- Semaphore (resource counting)
- Condition variables (thread coordination)

✅ **Problem-Solving**
- Common patterns
- Classic problems
- Real-world applications

✅ **Best Practices**
- Avoiding deadlocks
- Preventing race conditions
- Performance optimization

✅ **Debugging**
- Tools and techniques
- Common errors
- Testing strategies

**Keep this guide handy and refer to specific sections when solving problems!**

---

**Author:** OS Expert  
**Version:** 2.0  
**Last Updated:** 2025

**Good luck with your OS practicals! 🚀**