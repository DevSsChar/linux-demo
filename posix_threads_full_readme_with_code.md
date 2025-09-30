# POSIX Threads Demo Suite — README with Full Source

This document contains a README for a POSIX Threads demonstration project and includes the **exact full `main.c` source code** you provided (no omissions).

---

## Overview

This project demonstrates multithreading concepts using POSIX threads (`pthread`) in C. It includes examples for:

- Array summation (single vs multi-threaded)
- Producer-Consumer using semaphores and mutex
- Mutex demonstration (race vs safe increments)
- Matrix multiplication (single vs multi-threaded)
- Thread-safe file writing

Use this file as documentation and reference; the full source code follows below.

---

## How to compile

```bash
gcc -o posix_demo main.c -lpthread
```

Run:

```bash
./posix_demo
```

---

## Full Source (`main.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <semaphore.h>
#include <time.h>
#include <pthread.h>

#define ARRAY_SIZE 5000000
#define THREADS 4
#define buff 10
#define item 10
#define mat_size 400

typedef struct{
    int *arr;
    int start,end;
    long long sum;
}sumdata;

void* thread_sum(void *arg)
{
    sumdata *d=(sumdata*)arg;
    d->sum=0;
    for(int i=d->start;i<d->end;i++)
    {
        d->sum+=d->arr[i];
    }
    return NULL;
}

void matrixsum()
{
    int *arr=malloc(ARRAY_SIZE*sizeof(int));
    for(int i=0;i<ARRAY_SIZE;i++)arr[i]=i%100;
    
    clock_t start=clock();
    //for counting time
    long long s_sum=0;
    for(int i=0;i<ARRAY_SIZE;i++)s_sum+=arr[i];
    double ssum=(double)(clock()-start)/CLOCKS_PER_SEC;
    
    start=clock();
    pthread_t t[THREADS];
    //array of threads
    sumdata td[THREADS];
    //array of sum of threads
    int chunk=ARRAY_SIZE/THREADS;
    //amount of data to be done by each thread
    for(int i=0;i<THREADS;i++)
    {
        td[i].arr=arr;
        td[i].start=i*chunk;
        td[i].end=(i==THREADS-1)?ARRAY_SIZE:(i+1)*chunk;
        pthread_create(&t[i],NULL,thread_sum,&td[i]);
        //Create a new thread: store handle in t[i], run sum_thread(&td[i]).
        //Attributes NULL means default attributes
    }
    
    long long msum=0;
    for(int i=0;i<THREADS;i++)
    {
        pthread_join(t[i],NULL);
        //Wait for thread t[i] to finish. NULL means we ignore return pointer.
        msum+=td[i].sum;
    }
    
    double m_time=(double)(clock()-start)/CLOCKS_PER_SEC;
    
    printf("Single: sum=%lld and time=%.4fs and Multi: sum=%lld and time=%.4fs",s_sum,ssum,msum,m_time);
    printf("\n");
    free(arr);
}

int buf[buff],in=0,out=0;
sem_t empty,full;
pthread_mutex_t buf_mtx;

void* producer(void *arg)
{
    for(int i=0;i<item;i++)
    {
        int item1=rand()%100;
    sem_wait(&empty);
    pthread_mutex_lock(&buf_mtx);
    buf[in]=item1;
    printf("item:%d produced at %d\n",item1,in);
    in=(in+1)%buff;
    pthread_mutex_unlock(&buf_mtx);
    sem_post(&full);
    usleep(100000);
    }
    return NULL;
}
void* consumer(void *arg)
{
    for(int i=0;i<item;i++)
    {
        sem_wait(&full);
        pthread_mutex_lock(&buf_mtx);
        int item1=buf[out];
        printf("Item:%d consumed at %d",item1,out);
        out=(out+1)%buff;
        pthread_mutex_unlock(&buf_mtx);
        sem_post(&empty);
        usleep(100000);
    }
    return NULL;
}
void prodcon()
{
    printf("Producer Consumer \n");
    sem_init(&empty,0,buff);
    //Initialize empty semaphore with initial value BUF_SIZE (all slots free)
    //. 0 indicates not shared between processes.
    sem_init(&full,0,0);
    pthread_mutex_init(&buf_mtx,NULL);
    
    pthread_t p,c;
    pthread_create(&p,NULL,producer,NULL);
    pthread_create(&c,NULL,consumer,NULL);
    pthread_join(p,NULL);
    pthread_join(c,NULL);
    
    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&buf_mtx);
    printf("Completed!\n");
}

int counter=0;
pthread_mutex_t cnt_mtx;

void* inc_unsafe(void *arg)
{
    for(int i=0;i<100000;i++)counter++;
    return NULL;
}
void* inc_safe(void* arg)
{
    for(int i=0;i<100000;i++)
    {
        pthread_mutex_lock(&cnt_mtx);
        counter++;
        pthread_mutex_unlock(&cnt_mtx);
    }
    return NULL;
}
void mutex()
{
    printf("MUTEX\n");
    counter=0;
    pthread_t t1[10];
    for(int i=0;i<10;i++)pthread_create(&t1[i],NULL,inc_unsafe,NULL);
    for(int i=0;i<10;i++)pthread_join(t1[i],NULL);
    printf("UNSAFE INCREMENT:%d\n",counter);
    
    counter=0;
    pthread_mutex_init(&cnt_mtx,NULL);
    pthread_t t2[10];
    for(int i=0;i<10;i++)pthread_create(&t2[i],NULL,inc_safe,NULL);
    for(int i=0;i<10;i++)pthread_join(t2[i],NULL);
    printf("SAFE INCREMENT:%d\n",counter);
}

typedef struct{
   int **A,**B,**C;
   //A, B, C are int** (array-of-pointers rows); start/end row indices; size is matrix dimension.
   int start,end,size;
}matdata;

void* multiply(void* arg)
{
    matdata* d=(matdata*)arg;
    for(int i=d->start;i<d->end;i++)
    {
        for(int j=0;j<d->size;j++)
        {
            d->C[i][j]=0;
            for(int k=0;k<d->size;k++)
            {
                d->C[i][j]+=d->A[i][k]+d->B[k][j];
            }
        }
    }
}

void mulmatrix()
{
    int **A=malloc(mat_size*sizeof(int*));
    //Allocate arrays of int* pointers for rows of four matrices: A, B,
    //C1 (single-thread result), C2 (multi-thread result). Each A[i] will later be malloc'd to hold the row.
    int **B=malloc(mat_size*sizeof(int*));
    int **C1=malloc(mat_size*sizeof(int*));
    int **C2=malloc(mat_size*sizeof(int*));
    
    for(int i=0;i<mat_size;i++)
    {
        A[i]=malloc(mat_size*sizeof(int));
        B[i]=malloc(mat_size*sizeof(int));
        C1[i]=malloc(mat_size*sizeof(int));
        C2[i]=malloc(mat_size*sizeof(int));
        for(int j=0;j<mat_size;j++)
        {
            A[i][j]=rand()%10;
            B[i][j]=rand()%10;
        }
    }
    
    clock_t start=clock();
    for(int i=0;i<mat_size;i++)
    {
        for(int j=0;j<mat_size;j++)
        {
            C1[i][j]=0;
            for(int k=0;k<mat_size;k++)
            {
                C1[i][j]+=A[i][k]+B[k][j];
            }
        }
    }
    double ssum=(double)(clock()-start)/CLOCKS_PER_SEC;
    
    start=clock();
    pthread_t t[THREADS];
    matdata td[THREADS];
    int row=mat_size/THREADS;
    
    for(int i=0;i<THREADS;i++)
    {
        td[i].A=A; td[i].B=B; td[i].C=C2;
        td[i].size=mat_size;
        td[i].start=i*row;
        td[i].end=(i==THREADS-1)?mat_size:row*(i+1);
        pthread_create(&t[i],NULL,multiply,&td[i]);
    }
    
    for(int i=0;i<THREADS;i++)
    {
        pthread_join(t[i],NULL);
    }
    double msum=(double)(clock()-start)/CLOCKS_PER_SEC;
    printf("Single: time=%.4fs and Multi: time=%.4fs",ssum,msum);
    printf("\n");
}

pthread_mutex_t file_mtx;
FILE* file;

void* write_file(void* arg)
{
    int id=*((int*)arg);
    for(int i=0;i<5;i++)
    {
        pthread_mutex_lock(&file_mtx);
        fprintf(file,"Thread:%d has written %d",id,i+1);
        fflush(file);
        printf("Thread:%d has written %d",id,i+1);
        pthread_mutex_unlock(&file_mtx);
        usleep(10000);
    }
    return NULL;
}

void writing()
{
    printf("WRITING \n");
    file=fopen("output.txt","w");
    if(!file)
    {
        perror("Cant open file");
        return;
    }
    pthread_mutex_init(&file_mtx,NULL);
    pthread_t t[5];
    int ids[5];
    
    for(int i=0;i<5;i++)
    {
        ids[i]=i+1;
        pthread_create(&t[i],NULL,write_file,&ids[i]);
    }
    for(int i=0;i<5;i++)
    {
        pthread_join(t[i],NULL);
    }
    printf("DONE WRITING \n");
    fclose(file);
    pthread_mutex_destroy(&file_mtx);
}
int main() {
    srand(time(NULL));
    
    printf("╔═══════════════════════════════════════╗\n");
    printf("║   POSIX THREADS DEMO SUITE            ║\n");
    printf("╚═══════════════════════════════════════╝\n");
    
    int choice;
    do {
        printf("\n1. Array Sum\n2. Producer-Consumer\n3. Mutex Demo\n");
        printf("4. Matrix Multiplication\n5. File Writing\n6. Run All\n0. Exit\n");
        printf("Choice: ");
        scanf("%d", &choice);
        
        switch(choice) {
            case 1: matrixsum(); break;
            case 2: prodcon(); break;
            case 3: mutex(); break;
            case 4: mulmatrix(); break;
            case 5: writing(); break;
            case 0: printf("Exit\n"); break;
            default: printf("Invalid!\n");
        }
    } while(choice != 0);
    
    return 0;
}
```

---

If you want any edits, formatting changes, or additional notes added to this README (for example sample input/output blocks, compilation flags, or troubleshooting tips), tell me exactly what to add and I will update the canvas file. 

