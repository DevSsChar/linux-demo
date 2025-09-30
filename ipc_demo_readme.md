# Inter-Process Communication (IPC) Demo in C

This project demonstrates various IPC mechanisms in Linux/Unix using **Pipes**, **Named Pipes (FIFOs)**, and **client-server communication**. It is designed for learning and experimentation with inter-process communication, process coordination, and blocking/non-blocking behaviors.

---

## Features Implemented

1. **One-way Pipe (Parent -> Child)**
    - Demonstrates unidirectional communication using `pipe()`.
    - Parent sends a message to child, which reads and prints it.

2. **Two-way Pipe (Parent <-> Child)**
    - Demonstrates bidirectional communication using two pipes.
    - Parent and child exchange messages.

3. **FIFO (Named Pipe) Reader**
    - Demonstrates reading from a FIFO created using `mkfifo()`.
    - Reads a message sent by the FIFO writer.

4. **FIFO (Named Pipe) Writer**
    - Demonstrates writing to a FIFO.
    - Parent process writes a message into a named pipe.

5. **Client-Server Communication using FIFO**
    - Server waits for messages from client.
    - Client sends multiple messages to the server including a `quit` message to terminate.
    - Demonstrates communication between unrelated processes.

---

## How to Run

1. Compile the program:

```bash
gcc -o ipc_demo ipc_demo.c
```

2. Run the program:

```bash
./ipc_demo
```

3. Menu will appear:

```
Enter your choice
1. one way pipe
2. two way pipe
3. fifo reader
4. fifo writer
5. client
6. server
Click 7 to exit
```

4. Select the operation you want to perform.

**Important Note:**
- For client-server communication, run the **server first** (choice 6), then run the **client** (choice 5) to avoid blocking.

---

## Code

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <string.h>
#include <sys/stat.h>

void program1_oneway_pipe() {
    int pipefd[2];
    pid_t pid;
    char write_msg[]="hello from parent!!";
    char read_msg[100];

    if(pipe(pipefd)==-1) { perror("cant create pipe"); exit(1); }
    pid=fork();
    if(pid<0) { printf("Couldnt create child"); exit(1); }

    if(pid>0) {
        close(pipefd[0]);
        printf("Parent is sending %s\n",write_msg);
        write(pipefd[1],write_msg,strlen(write_msg)+1);
        close(pipefd[1]);
        wait(NULL);
    } else {
        close(pipefd[1]);
        read(pipefd[0],read_msg,sizeof(read_msg));
        printf("Child is reading %s\n",read_msg);
        close(pipefd[0]);
    }
}

void program2_twoway_pipe() {
    int pipe1[2],pipe2[2];
    pid_t pid;
    char write_msg[]="hello from parent \n";
    char write_msg1[]="hello from child \n";
    char read_msg[100];

    if(pipe(pipe1)==-1 || pipe(pipe2)==-1) { perror("Unable to create pipe"); exit(1); }
    pid=fork();
    if(pid<0) { perror("Child cant be created"); exit(1); }

    if(pid>0) {
        close(pipe1[0]); close(pipe2[1]);
        printf("parent is sending %s\n",write_msg);
        write(pipe1[1],write_msg,strlen(write_msg)+1);
        read(pipe2[0],read_msg,sizeof(read_msg));
        printf("parent has been given %s \n",read_msg);
        close(pipe1[1]); close(pipe2[0]);
    } else {
        close(pipe1[1]); close(pipe2[0]);
        read(pipe1[0],read_msg,sizeof(read_msg));
        printf("child has been given %s \n",read_msg);
        write(pipe2[1],write_msg1,strlen(write_msg1)+1);
        printf("child is sending %s\n",write_msg1);
        close(pipe1[0]); close(pipe2[1]);
    }
}

void program3_fifo_writer() {
    const char* file_path="/tmp/my_fifo";
    char write_msg[]="Hello from parent!!\n";
    mkfifo(file_path,0666);

    printf("Writer opening file to write:\n");
    int fd=open(file_path,O_WRONLY);
    printf("Writer writing %s:\n",write_msg);
    write(fd,write_msg,strlen(write_msg)+1);
    close(fd);
    printf("Writer is done writing\n");
}

void program3_fifo_reader() {
    const char* file_path="/tmp/my_fifo";
    char buffer[100];
    printf("Reader opening file to read\n");
    int fd=open(file_path,O_RDONLY);
    read(fd,buffer,sizeof(buffer));
    printf("Reader has read that: %s",buffer);
    close(fd);
    unlink(file_path);
}

void program5_fifo_server() {
    const char* fifo_path="/tmp/unrelated_file";
    char buffer[100];
    mkfifo(fifo_path,0666);
    printf("Server started:\n");
    while(1) {
        int fd=open(fifo_path,O_RDONLY);
        ssize_t bytes=read(fd,buffer,sizeof(buffer));
        if(bytes>0) {
            printf("Server received %s",buffer);
            if(strcmp(buffer,"quit")==0) { close(fd); break; }
        }
        close(fd);
    }
    unlink(fifo_path);
    printf("Server shutting down \n");
}

void program5_fifo_client() {
    const char* fifo_path="/tmp/unrelated_file";
    char message[][50]={"Hello from Client","This is message 2","Communication successful!","quit"};
    printf("Opening client\n");
    sleep(1);
    for(int i=0;i<4;i++) {
        int fd=open(fifo_path,O_WRONLY);
        printf("Client is sending: %s",message[i]);
        write(fd,message[i],strlen(message[i])+1);
        close(fd);
        sleep(1);
    }
    printf("[CLIENT] Done.\n");
}

int main() {
    int choice;
    do {
        printf("Enter your choice\n");
        printf("1. one way pipe\n2. two way pipe\n3. fifo reader\n4. fifo writer\n5. client\n6. server\nClick 7 to exit\n");
        scanf("%d",&choice);
        switch(choice) {
            case 1: printf("\n=== ONE-WAY PIPE ===\n"); program1_oneway_pipe(); break;
            case 2: printf("\n=== TWO-WAY PIPE ===\n"); program2_twoway_pipe(); break;
            case 3: printf("FIFO READER\n"); program3_fifo_reader(); break;
            case 4: printf("FIFO WRITER\n"); program3_fifo_writer(); break;
            case 5: printf("CLIENT \n"); program5_fifo_client(); break;
            case 6: printf("SERVER \n"); program5_fifo_server(); break;
            case 7: exit(0);
            default: printf("Invalid Choice \n");
        }
    } while(choice!=7);
}
```

---

## Notes

- **Blocking Behavior:**
  - FIFO `open()` can block if the other end is not open. Always run server first then client.
  - Menu-driven execution may require separate terminals for server and client.
- **Cleanup:**
  - `unlink()` is used to remove FIFOs after usage.
- **Permissions:**
  - FIFO permissions set to `0666` to allow read/write by all users.

---

This program serves as a practical demonstration of IPC mechanisms in C for Linux/Unix environments.

