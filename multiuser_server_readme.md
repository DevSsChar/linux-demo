# Multi-User Server Simulation using `fork()` in Linux

This project demonstrates how a **multi-user web server** handles multiple concurrent client requests by creating separate processes. It uses the `fork()` system call in Linux to simulate **process creation**, **depth tracking**, and **resource management**.  

Additionally, a shell script is provided to automatically terminate all child processes after a specified delay.

---

## 📌 Features

- Simulates a server that spawns processes for client requests.  
- Tracks **process depth** (hierarchical parent-child relationship).  
- Demonstrates **process IDs (PID)** and **parent process IDs (PPID)**.  
- Visualizes the process tree using Linux commands.  
- Includes a **shell script** to kill all child processes after a delay.  

---

## 🖥️ C Program: Server Simulation with Depth Tracking

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

void create_processes(int depth, int max_depth) {
    if (depth > max_depth) return;

    pid_t pid = fork();

    if (pid < 0) {
        perror("Fork failed");
        exit(1);
    }

    if (pid == 0) {  
        // Child process
        printf("Child Process: PID=%d, PPID=%d, Depth=%d\n", getpid(), getppid(), depth);

        // Simulate some "work"
        sleep(3);

        // Create the next child if depth allows
        create_processes(depth + 1, max_depth);

        exit(0);
    } else {
        // Parent waits for child
        wait(NULL);
    }
}

int main() {
    int clients;

    printf("Enter number of client requests (process depth): ");
    scanf("%d", &clients);

    printf("Parent Process: PID=%d\n", getpid());

    // Start from depth 1 (child level)
    create_processes(1, clients);

    return 0;
}
```

---

## ⚙️ How it Works

1. The program asks for the **number of clients** (process depth).  
2. Each `fork()` call creates a **new child process**.  
3. Each child prints:
   - Its **PID** (Process ID).  
   - Its **PPID** (Parent Process ID).  
   - Its **depth** level in the hierarchy.  
4. Each child simulates processing time using `sleep(3)`.  
5. Use the following commands to view the process tree:
   ```bash
   ps -f --forest
   ```
   or
   ```bash
   pstree -p
   ```

---

## 📊 Example Output

If user enters `3`:

```
Parent Process: PID=12345
Child Process: PID=12346, PPID=12345, Depth=1
Child Process: PID=12347, PPID=12346, Depth=2
Child Process: PID=12348, PPID=12347, Depth=3
```

---

## 🔪 Shell Script: Kill Child Processes After a Delay

Save as **`kill_children.sh`**:

```bash
#!/bin/bash

# Usage: ./kill_children.sh <parent_pid> <delay_seconds>

PARENT_PID=$1
DELAY=$2

echo "Waiting for $DELAY seconds before killing children of PID $PARENT_PID..."
sleep $DELAY

# Kill all child processes of the parent
pkill -P $PARENT_PID

echo "All child processes of $PARENT_PID have been killed."
```

---

## 🚀 How to Run

1. Compile and run the C program:
   ```bash
   gcc server_sim.c -o server_sim
   ./server_sim
   ```

2. Note the **parent process ID** (printed as output).  
   Example:
   ```
   Parent Process: PID=12345
   ```

3. Run the shell script to kill child processes after a delay (e.g., 5 seconds):
   ```bash
   ./kill_children.sh 12345 5
   ```

---

## 🎯 Learning Outcomes

- Understand **process creation** using `fork()`.  
- Observe **parent-child relationships** in Linux processes.  
- Track **depth levels** in a process hierarchy.  
- Learn how to **manage and terminate processes** using shell scripts.  