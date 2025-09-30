# Scheduling Simulator — README

## Overview

This C program is a CPU scheduling simulator that implements four common scheduling algorithms:

- **FCFS (First-Come, First-Served)**
- **SJF (Shortest Job First) — non-preemptive**
- **Round Robin (RR)**
- **Priority Scheduling (non-preemptive)**

Each process has the following attributes:
- `pid` — process id (assigned by the simulator)
- `arrival` — arrival time
- `burst` — CPU burst time
- `priority` — smaller value = higher priority

The simulator computes for each process:
- `ct` — completion time
- `tat` — turnaround time (`ct - arrival`)
- `wt` — waiting time (`tat - burst`)

This README explains how to compile and run the program, the input format, and a short description of each algorithm implemented.

---

## Files
- `scheduling_simulator.c` — main source file (contains all algorithm implementations and a menu-driven `main`).
- Output files are not required; results are printed to stdout.

---

## Compile

```bash
gcc scheduling_simulator.c -o scheduler
```

No special libraries are required — compile with a standard Linux `gcc` toolchain.

---

## Run

```bash
./scheduler
```

The program will prompt for the number of processes, then for each process: arrival time, burst time, and priority. It will also ask for the time quantum used by Round Robin.

### Input format (interactive)
```
Enter number of processes: <n>
For each process i from 1..n enter:
  <arrival> <burst> <priority>
Enter time quantum: <quantum>
```

Example:
```
Enter number of processes: 3
P1: 0 5 2
P2: 1 3 1
P3: 2 8 3
Enter time quantum: 2
```

---

## Output

For each scheduling algorithm the program prints a table with:
```
PID AT BT CT TAT WT
```
and prints average waiting time and average turnaround time at the end of each table.

A typical output block will look like:
```
-------- FCFS --------
PID AT  BT  CT  TAT WT
P1  0   5   5   5   0
P2  1   3   8   7   4
P3  2   8   16  14  6
Average WT: 3.33  Average TAT: 8.67
```

---

## Algorithm notes

### FCFS
- Processes are served in the order of arrival.
- If the CPU is idle (no process has arrived yet), time jumps to the next arrival.

### SJF (non-preemptive)
- At each step the scheduler picks the process with the smallest burst among the arrived and not-yet-completed processes.
- Non-preemptive: once a process starts, it runs to completion.

### Round Robin (RR)
- Uses a fixed time quantum; processes are served in a ready queue in FIFO order.
- If the ready queue is empty the simulator advances time to the next arrival.
- When a process's remaining time becomes 0 it finishes and completion time is recorded.

### Priority (non-preemptive)
- The scheduler chooses the ready process with the smallest `priority` value.
- Non-preemptive: running process finishes before the scheduler picks the next one.

---

## Assumptions & Limitations
- All scheduling implementations are **non-preemptive**, except Round Robin which is time-sliced.
- Arrival times, burst times, priorities and quantum are integers and non-negative.
- The program uses simple in-memory arrays and is intended for educational/demo use; it does not scale to thousands of processes.
- The simulator assumes processes are indexed starting at 1 (PID assigned as input order). If arrival times are not sorted, FCFS uses the given order; SJF and Priority choose among arrived processes at runtime.

---

## Full Source

Below is the full source code. Save it as `scheduling_simulator.c` and compile with the `gcc` command above.

```c
#include <stdio.h>
#include <limits.h>

typedef struct{
  int pid,arrival,burst,priority;
  int ct,tat,wt,rm;
}process;

void printResults(process p[],int n,char * name)
{
    printf("-------- %s --------\n",name);
    float avgwt=0,avgtat=0;
    printf("PID AT BT CT TAT WT\n");
    for(int i=0;i<n;i++)
    {
        printf("P%-2d %-3d %-3d %-3d %-3d %-4d\n",p[i].pid,p[i].arrival,p[i].burst,p[i].ct,p[i].tat,p[i].wt);
        avgwt+=p[i].wt;
        avgtat+=p[i].tat;
    }
    printf("Average WT:%.2f Average BT:%.2f\n",avgwt/n,avgtat/n);
}

void fcfs(process p[],int n)
{
    int time=0;
    for(int i=0;i<n;i++)
    {
        if(time<p[i].arrival)time=p[i].arrival;
        p[i].ct=time+p[i].burst;
        p[i].tat=p[i].ct-p[i].arrival;
        p[i].wt=p[i].tat-p[i].burst;
        time=p[i].ct;
    }
    printResults(p,n,"FCFS");
}

void sjf(process p[],int n)
{
    int done[n],time=0,completed=0;
    for(int i=0;i<n;i++)
    {
        done[i]=0;
    }
    while(completed<n)
    {
        int idx=-1,minburst=INT_MAX;
        for(int i=0;i<n;i++)
        {
            if(!done[i] && p[i].arrival<=time && p[i].burst<minburst)
            {
                minburst=p[i].burst;
                idx=i;
            }
        }
        if(idx==-1){
            time++; 
            continue;
        }
        p[idx].ct=time+p[idx].burst;
        p[idx].tat=p[idx].ct-p[idx].arrival;
        p[idx].wt=p[idx].tat-p[idx].burst;
        time=p[idx].ct;
        completed++;
        done[idx]=1;
    }
    printResults(p,n,"SJF");
}
void roundrobin(process p[],int n,int quantum)
{
    int inq[n],q[100],front=0,rear=0,done=0,time=0;
    for(int i=0;i<n;i++)
    {
        p[i].rm=p[i].burst;
        inq[i]=0;
    }
    q[rear++]=0;
    inq[0]=1;
    while(done<n)
    {
        if(front==rear)
        {
            time++; continue;
        }
        int i=q[front++];
        int exec=p[i].rm<quantum?p[i].rm:quantum;
        p[i].rm-=exec;
        time+=exec;
        
        for(int j=0;j<n;j++)
        {
            if(!inq[j] && p[j].arrival<=time && p[j].rm>0)
            {
                q[rear++]=j;
                inq[j]=1;
            }
        }
        
        if(p[i].rm==0)
        {
            p[i].ct=time;
            p[i].tat=p[i].ct-p[i].arrival;
            p[i].wt=p[i].tat-p[i].burst;
            done++;
        }else{
            q[rear++]=i;
        }
    }
    printResults(p,n,"ROUNDROBIN");
}
void priority(process p[],int n)
{
    int done[n],time=0,completed=0;
    for(int i=0;i<n;i++)
    {
        done[i]=0;
    }
    while(completed<n)
    {
        int idx=-1,maxprio=INT_MAX;
        for(int i=0;i<n;i++)
        {
            if(!done[i] && p[i].arrival<=time && p[i].priority<maxprio)
            {
                maxprio=p[i].priority;
                idx=i;
            }
        }
        if(idx==-1){
            time++; 
            continue;
        }
        p[idx].ct=time+p[idx].burst;
        p[idx].tat=p[idx].ct-p[idx].arrival;
        p[idx].wt=p[idx].tat-p[idx].burst;
        time=p[idx].ct;
        completed++;
        done[idx]=1;
    }
    printResults(p,n,"PRIORITY");
}
int main()
{
    int n;
    printf("Enter number of processes:");
    scanf("%d",&n);
    
    process p[n],temp[n];
    
    printf("Enter details of process:\n");
    for(int i=0;i<n;i++)
    {
        p[i].pid=i+1;
        printf("P%d: ",p[i].pid);
        scanf("%d %d %d",&p[i].arrival,&p[i].burst,&p[i].priority);
    }
    int quantum;
    printf("Enter time quantum:");
    scanf("%d",&quantum);
    
    for(int i=0;i<n;i++)temp[i]=p[i];
    fcfs(temp,n);
    
    for(int i=0;i<n;i++)temp[i]=p[i];
    sjf(temp,n);
    
    for(int i=0;i<n;i++)temp[i]=p[i];
    roundrobin(temp,n,quantum);
    
    for(int i=0;i<n;i++)temp[i]=p[i];
    priority(temp,n);
    
    return 0;
}
```

---

## Tips & Next Steps
- Try different arrival orders to see how SJF and Priority choose processes at runtime.
- Extend the simulator to include **preemptive SJF (SRTF)** and **preemptive priority**.
- Add Gantt chart printing to visualize the schedule timeline.

---

If you want, I can:
- Add sample input/output blocks for a specific test case.
- Add a Makefile and test scripts.
- Convert this into a lab worksheet with step-by-step tasks and expected outputs.

Tell me which you'd like next!
