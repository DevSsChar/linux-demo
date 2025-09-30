# Linux Process System Calls Demonstration in C

This program demonstrates various Linux process-related system calls and concepts using the C programming language. It covers `fork()`, `wait()` and `waitpid()`, the exec family (`execl`, `execlp`, `execv`, `execvp`), file operations (`open()`, `read()`, `write()`, `close()`), input/output redirection with `dup`/`dup2`, and concepts of orphans, zombies, and recursive process creation.

## Features Demonstrated

1. **Fork Demo** - Basic creation of a child process using `fork()`. Logs process IDs and demonstrates parent-child relationship.
2. **Wait & Waitpid** - Synchronizing parent and child processes using `wait()` and `waitpid()`.
3. **Zombie Demo** - Shows how a terminated child becomes a zombie until parent calls `wait()`.
4. **Orphan Demo** - Demonstrates how a child process gets adopted by `init` when parent exits early.
5. **Sequential Fork** - Creating multiple children sequentially.
6. **Recursive Fork** - Demonstrates creation of a process tree recursively.
7. **Exec Family** - Demonstrates executing new programs using `execl`, `execlp`, `execv`, and `execvp`.
8. **File Operations** - Demonstrates reading and writing to files using low-level system calls.
9. **Dup/Dup2** - Demonstrates redirection of standard output to a file.
10. **I/O Redirect** - Redirects both stdin and stdout using `dup2` and runs `cat` to read from input file and write to output file.

## Usage

1. Compile the program:
```bash
gcc main.c -o process_demo
```

2. Run the program:
```bash
./process_demo
```

3. Follow the menu options to test each demonstration.

4. Monitor processes in real-time using:
```bash
ps aux | grep process_demo
```

5. Check output logs in `child_log.txt` and other output files.

## Code Explanation

- **Logging:** `log_message(const char *msg)` logs messages to `child_log.txt` with timestamps, PID, and PPID.
- **Print Separator:** `print_separator()` is used to separate outputs for clarity.
- **Press Enter Prompt:** `press_enter()` waits for user input to continue.
- **Menu Driven:** `menu()` displays choices for the user to run different demos.
- **Fork Demos:** Demonstrates creation of single, sequential, and recursive child processes.
- **Wait/Waitpid:** Synchronization of parent with children.
- **Zombie and Orphan:** Illustrates special process states.
- **Exec Family:** Runs external programs in child processes.
- **File Operations & Dup/Dup2:** Shows reading, writing, and redirecting output to files.
- **I/O Redirect Demo:** Redirects stdin and stdout to demonstrate file-based input/output with a child process.

---

### Full Source Code

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <fcntl.h>
#include <string.h>
#include <signal.h>
#include <time.h>

#define LOG_FILE "child_log.txt"
#define OUTPUT_FILE "output.txt"
#define INPUT_FILE "input.txt"

void log_message(const char* msg)
{
    int fd=open(LOG_FILE, O_WRONLY | O_CREAT | O_APPEND, 0644);
    //open returns file descriptor fd, o_wronly->write only, o_creat->create,0644-> read write permission
    if(fd<0)
    {
        printf("failed to open file");
        return;
    }
    time_t now=time(NULL);//create present time
    char *timestamp=ctime(&now);//create a string, introduces newline by default
    timestamp[strlen(timestamp)-1]='\0';//remove newline
    
    char buffer[512];
    snprintf(buffer,sizeof(buffer),"[%s] pid=%d ppid=%d %s\n",timestamp,getpid(),getppid(),msg);
    write(fd,buffer,strlen(buffer));
    close(fd);
    
    printf("%s",buffer);//print msg to screen
}
void print_seperator(const char *title) {
    printf("\n");
    printf("========================================\n");
    printf("  %s\n", title);
    printf("========================================\n");
}

void press_enter() {
    printf("\n[Press Enter to continue...]\n");
    getchar();
}

//main functions

void demo_fork()
{
    print_seperator("1. FORK DEMO/n");
    pid_t pid=fork();
    log_message("FORK DEMO:");
    if(pid<0)
    {
        printf("failed to create child");
        return;
    }
    if(pid==0)
    {
        char buffer[100];
        snprintf(buffer, sizeof(buffer), "child with pid:%d created by parent of pid:%d", getpid(), getppid());
        log_message(buffer);

        sleep(1);
        log_message("child is exiting");
        exit(0);
    }else{
        log_message("parent waits for child");
        wait(NULL);
        log_message("parent done working");
    }
}

void demo_wait_waitpid()
{
    print_seperator("2. wait & waitpid \n");
    log_message("wait and wait pid");
    
    pid_t pids[3];//take 3 children
    for(int i=0;i<3;i++)
    {
        pids[i]=fork();
        if(pids[i]==0)
        {
            char msg[100];
            snprintf(msg,sizeof(msg),"child %d working for %d seconds\n",i+1,3-i);
            log_message(msg);
            sleep(3-i);
            
            snprintf(msg,sizeof(msg),"child %d exiting with code %d\n",i+1,10+i);
            log_message(msg);
            exit(i+10);
        }
    }
    
    printf("\n parent: waiting for child to exit of pid:%d",pids[2]);
    int status;
    waitpid(pids[2],&status,0);
    //blocks until child is done
    
    if(WIFEXITED(status))
    {
        printf("child exited with status %d\n",WEXITSTATUS(status));
    }
    printf("waiting for remaining children \n");
    while(wait(&status)>0)
    {
        if(WIFEXITED(status))
        {
          printf("a child exited with status %d\n",WEXITSTATUS(status));
        }
    }
    log_message("all child exited");
}
void demo_zombie()
{
    print_seperator("3. ZOMBIE DEMO\n");
    printf("ZOMBIE STARTS\n");
    
    if(fork()==0)
    {
        log_message("ZOMBIE GONE");
        exit(0);
    }
    log_message("parent sleeping for 10s, zombie there");
    sleep(10);
    wait(NULL);
    log_message("zombie cleaned");
}
void orphan()
{
    print_seperator("4. ORPHAN DEMO \n");
    printf("Orphan demo\n");
    if(fork()==0)
    {
        sleep(2);
        log_message("child adopted by init");
        exit(0);
    }
    log_message("child is orphaned rn");
    exit(0);
}
void sequentialfork()
{
    print_seperator("5. SEQUENTIALFORK\n");
    printf("SEQUENTIALFORK starts\n");
    for(int i=1;i<=3;i++)
    {
        if(fork()==0)
        {
            char msg[100];
            snprintf(msg,sizeof(msg),"child process created with %d\n",getpid());
            log_message(msg);
            exit(i);
        }
    }
    for(int i=0;i<3;i++)wait(NULL);
    log_message("are child done SEQUENTIALFORK");
}
void recurse(int depth)
{
    if(depth>2)return;
    char msg[50];
    snprintf(msg,sizeof(msg),"tree child of %d pid\n",getpid());
    log_message(msg);
    
    for(int i=0;i<2;i++)
    {
        if(fork()==0)
        {
            recurse(depth+1);
            exit(0);
        }
    }
    wait(NULL);
    wait(NULL);
}
void recursivefork()
{
    print_seperator("6. RECURSIVE FORK\n");
    printf("recursive fork tree:\n");
    recurse(0);
    log_message("Fork tree done");
}
void exec_family()
{
    print_seperator("7. EXEC FAMILY \n");
    printf("exec family\n");
    log_message("exec family commands");
    
    if(fork()==0)//create child
    {
        execl("/bin/echo","echo","execl works",NULL);//(full_path,program_name,arg_print,end->NULL)
        exit(1);//child not done then exit 1
    }
    wait(NULL);//parent waits for child
    
    if(fork()==0)
    {
        execlp("echo","echo","execlp works",NULL);//path,rest like execl
        exit(1);
    }
    wait(NULL);
    
    if(fork()==0)
    {
        char *arg[]={"echo","execv works",NULL};//program_name,message,done
        execv("/bin/echo",arg);//need full path here,list of arg
        exit(1);
    }
    wait(NULL);
    
    if(fork()==0)
    {
        char *arg[]={"echo","execvp works",NULL};//program_name,message,done
        execvp("echo",arg);//doesnt need full path here,list of arg
        exit(1);
    }
    wait(NULL);
}
void file_operations()
{
    print_seperator("8. FILE OPERATIONS \n");
    printf("file operations\n");
    
    int fd=open("file.txt",O_WRONLY | O_CREAT | O_APPEND,0644);
    write(fd,"hello from write",16);
    close(fd);
    
    fd=open("file.txt",O_RDONLY);
    char buff[100];
    int n=read(fd,buff,100);
    buff[n]='\0';
    printf("%s",buff);
    close(fd);
    log_message("file opr done");
}
void redirection()
{
    print_seperator("9. DUP/DUP2\n");
    printf("io redirection with dup \n");
    if(fork()==0)
    {
        int fd=open("dupp.text",O_WRONLY | O_CREAT | O_APPEND,0644);
    dup2(fd,1);
    close(fd);
    
    printf("This goes to file dupp");
    printf("this too");
    
    exit(1);
    }
    wait(NULL);
    log_message("dup done");
}
void demo10() {
    printf("\n=== 10. I/O REDIRECT ===\n");
    
    // Create input file
    int fd = open("input.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    write(fd, "Line 1\nLine 2\nLine 3\n", 21);
    close(fd);
    
    if (fork() == 0) {
        // Redirect stdin
        int in = open("input.txt", O_RDONLY);
        dup2(in, 0);
        close(in);
        
        // Redirect stdout
        int out = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
        dup2(out, 1);
        close(out);
        
        // Cat reads stdin, writes stdout
        execlp("cat", "cat", NULL);
        exit(1);
    }
    wait(NULL);
    printf("Check output.txt\n");
}
void menu()
{
    printf("MENU\n");
    printf("1. FORK DEMO/n");
    printf("2. wait & waitpid \n");
    printf("3. ZOMBIE DEMO\n");
    printf("4. ORPHAN DEMO \n");
    printf("5. SEQUENTIALFORK\n");
    printf("6. RECURSIVE FORK\n");
    printf("7. EXEC FAMILY \n");
    printf("8. FILE OPERATIONS \n");
    printf("9. DUP/DUP2\n");
    printf("10. I/O REDIRECT\n");
    printf("0.  Exit\n");
    printf("Monitor: ps aux | grep compact_demo\n");
    printf("Choice: ");
}
int main()
{
    printf("START \n");
    log_message("program started");
    
    int choice;
    
    while(1)
    {
        menu();
        scanf("%d",&choice);
        getchar();
        
        switch(choice) {
            case 1: demo_fork(); break;
            case 2: demo_wait_waitpid(); break;
            case 3: demo_zombie(); break;
            case 4: orphan(); return 0; // Orphan exits parent
            case 5: sequentialfork(); break;
            case 6: recursivefork(); break;
            case 7: exec_family(); break;
            case 8: file_operations(); break;
            case 9: redirection(); break;
            case 10: demo10(); break;
            case 0:
                log_message("Exiting");
                printf("Check: demo.log, *.txt files\n");
                return 0;
            default: printf("Invalid\n");
        }
        printf("\n[Press Enter]");
        getchar();
    }
}
```

---

### Notes

- Make sure to **compile with gcc** on Linux.
- Use `ps` or `top` to **observe child processes** and their states (zombie/orphan).  
- Check `child_log.txt` to see **timestamped logs of child processes**.
- Files like `file.txt`, `output.txt`, and `dupp.txt` are created to demonstrate **I/O operations and redirection**.
- This program is **menu-driven** for step-by-step execution of all demos.

