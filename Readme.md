# xv6 OS Modification for Additional System Calls and Utilities

This README provides step-by-step instructions on how to modify the original xv6 operating system to implement two new system calls (`ps` and `waitx`) and create two utility programs (`head` and `uniq`). Additionally, we will add functionality to track process statistics such as creation time, end time, and total time.

## Table of Contents

1. [Introduction](#introduction)
2. [System Call and Utility Overview](#overview)
3. [Modification Steps](#modification-steps)
   1. [Update `defs.h`](#update-defsh)
   2. [Update `syscall.c`](#update-syscallc)
   3. [Update `syscall.h`](#update-syscallh)
   4. [Update `sysproc.c`](#update-sysprocc)
   5. [Update `proc.h`](#update-proch)
   6. [Update `proc.c`](#update-procc)
   7. [Update `usys.S`](#update-usyss)
   8. [Update `user.h`](#update-userh)
   9. [Update `Makefile`](#update-makefile)
   10. [Create `head.c`](#create-headc)
   11. [Create `uniq.c`](#create-uniqc)
   12. [Create `test.c`](#create-testc)
   13. [Create `pstat.h`](#create-pstath)
   14. [Create `ps.c`](#create-psc)
4. [Building and Running](#building-and-running)
5. [Conclusion](#conclusion)

## 1. Introduction <a name="introduction"></a>

This project involves extending the xv6 operating system with two new system calls (`ps` and `waitx`) and creating two utility programs (`head` and `uniq`). Additionally, it tracks process statistics like creation time, end time, and total time.

## 2. System Call and Utility Overview <a name="overview"></a>

- **System Calls**:
  - `ps`: Displays process information including PID, state, name, creation time, end time, and total time.
  - `waitx`: Extends the functionality of `wait` to include process statistics.
- **Utilities**:
  - `head`: Displays the first N lines of a file.
  - `uniq`: Filters repeated lines from a file, displaying only unique lines.

## 3. Modification Steps <a name="modification-steps"></a>

### 3.1. Update `defs.h` <a name="update-defsh"></a>

Edit the `defs.h` file and add the following lines in the `proc` section:

```c
int ps(void);
int waitx(int, struct pstat *pstat);
```

### 3.2. Update `syscall.c` <a name="update-syscallc"></a>

In the `syscall.c` file, add the following lines below the predefined externs:

```c
extern int sys_ps(void);
extern int sys_waitx(void);
```

In the `syscall` array, add these lines:

```c
[SYS_ps]      sys_ps,
[SYS_waitx]   sys_waitx,
```

### 3.3. Update `syscall.h` <a name="update-syscallh"></a>

Edit the `syscall.h` file and add the below code in the `syscall` section:

```c
#define SYS_ps     22
#define SYS_waitx  23
```

### 3.4. Update `sysproc.c` <a name="update-sysprocc"></a>

In the `sysproc.c` file, add the following code:

```c
int sys_waitx(void) {
  int pid;
  struct pstat *pstat;
  if (argint(0, &pid) < 0 || argptr(1, (void*)&pstat, sizeof(*pstat)) < 0)
    return -1;
  return waitx(pid, pstat);
}

int sys_ps(void)
{
  return  ps();
}
```

### 3.5. Update `proc.h` <a name="update-proch"></a>

Edit the `proc.h` file and add the following lines:

```c
// New fields for extended proc struct
int ctime, etime, ttime;
```

### 3.6. Update `proc.c` <a name="update-procc"></a>

In the `proc.c` file, add the following lines as instructed:

```c
#include "pstat.h"


p->ctime = ticks;
p->etime = 0;

curproc->etime = ticks;
curproc->etime = ticks;

int waitx(int pid, struct pstat *pstat) {
  struct proc *p;
  int havekids;
  struct proc *curproc = myproc();
  acquire(&ptable.lock);
  for (;;) {
    havekids = 0;
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->parent != curproc)
        continue;
      havekids = 1;
      if (p->pid == pid && p->state == ZOMBIE) {
        // parent is waiting for child to exit.
        if (pstat != 0) {
          pstat->ctime = p->ctime;
          pstat->etime = p->etime;  //Filling up the process statistics structure
          pstat->ttime = p->etime - p->ctime; // For Calculating total time
        }
        kfree(p->kstack);
        p->kstack = 0;
        freevm(p->pgdir); // Free the resources allocated
        p->pid = 0;       // content between acquire and release is called critical section.
        p->parent = 0;
        p->name[0] = 0;
        p->killed = 0;
        p->state = UNUSED;

        // ptable => process table that stores list of processes which are running.

        release(&ptable.lock);
        return pid;
      }
    }
    curproc->etime = ticks;

    //ticks => Tracking process time

    if(!havekids || curproc->killed){
      release(&ptable.lock);
      return -1;
    }

    sleep(curproc, &ptable.lock);
  }
}

int ps(void)
{
    struct proc *p;

   // Interrupting the processor
    sti();

    acquire(&ptable.lock);
    // printing the process statistics using ps command
    cprintf("PID \t| State \t| Name \t| Creation Time \t| End Time \t| Total Time |\n");
    for (p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state == UNUSED) {
            continue;
        }
        p->etime = ticks;

        int ttime = p->etime - p->ctime;
        cprintf("%d \t| %s \t| %s \t| %d \t| %d \t| %d|\n",
                p->pid, p->state == SLEEPING ? "SLEEPING" : "RUNNING",
                p->name, p->ctime, p->etime, ttime);
    }
    release(&ptable.lock);

    return 0;
}
```

### 3.7. Update `usys.S` <a name="update-usyss"></a>

Edit the `usys.S` file and add the following lines:

```assembly
SYSCALL(ps)
SYSCALL(waitx)
```

### 3.8. Update `user.h` <a name="update-userh"></a>

Edit the `user.h` file and add the following lines in the system call section:

```c
int ps(void);
int waitx(int pid, struct pstat *pstat);
```

### 3.9. Update `Makefile` <a name="update-makefile"></a>

Edit the `Makefile` file and add the following lines in the user section:

```make
_ps\
_head\
_uniq\
_test\
```

### 3.10. Create `head.c` <a name="create-headc"></a>

Create a new file named `head.c` and add the following code:

```c
#include "types.h"
#include "stat.h"
#include "user.h"

#define MAX_LINE_LENGTH 1024
#define DEFAULT_N_LINES 10

void head(int fd, int n) {
    char line[MAX_LINE_LENGTH];
    int line_count = 0;

    while (1) {
        int bytesRead = read(fd, line, sizeof(line));
        if (bytesRead <= 0) {
            break;
        }

        for (int i = 0; i < bytesRead; i++) {
            if (line[i] == '\n') {
                line_count++;
                if (line_count > n) {
                    break;
                }
            }

            printf(1, "%c", line[i]);

            if (line_count >= n) {
                break;
            }
        }

        if (line_count >= n) {
            break;
        }
    }
}

int main(int argc, char *argv[]) {
    int n = DEFAULT_N_LINES;
    int fd = 0; // Initialize to standard input (0)

    if (argc > 1 && argv[1][0] == '-') {
        // Parse the number of lines from the command-line argument
        n = atoi(argv[1] + 1);

        if (n <= 0) {
            printf(2, "Usage: head [-N] [file]\n");
            exit();
        }

        // Open the file if provided
        if (argc > 2) {
            fd = open(argv[2], 0);
        }
    } else {
        // No option provided, use default number of lines
        if (argc > 1) {
            fd = open(argv[1], 0);
        }
    }

    if (fd < 0) {
        printf(2, "head: cannot open '%s'\n", argv[argc - 1]);
        exit();
    }

    head(fd, n);
    close(fd);
    exit();
}
```

### 3.11. Create `uniq.c` <a name="create-uniqc"></a>

Create a new file named `uniq.c` and add the following code:

```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "fcntl.h"

#define MAX_LINE_LENGTH 1024

void uniq(int input_fd) {
    char line[MAX_LINE_LENGTH];
    char prev_line[MAX_LINE_LENGTH] = "";  // Store the previous line

    while (1) {
        int n = read(input_fd, line, sizeof(line));

        if (n <= 0) {
            break;  // End of file or an error
        }

        line[n] = '\0';  // Null-terminate the line

        // If the current line is different from the previous line, print it
        if (strcmp(line, prev_line) != 0) {
            printf(1, "%s", line);
            strcpy(prev_line, line);  // Update the previous line
        }
    }
}

int main(int argc, char *argv[]) {
    int input_fd = 0;  // Default to standard input (file descriptor 0)

    if (argc > 1) {
        // Open the file if provided
        input_fd = open(argv[1], O_RDONLY);

        if (input_fd < 0) {
            printf(2, "uniq: cannot open %s\n", argv[1]);
            exit();
        }
    }

    uniq(input_fd);

    if (input_fd != 0) {
        close(input_fd);
    }

    exit();
}
```

### 3.12. Create `test.c` <a name="create-testc"></a>

Create a new file named `test.c` and add the following code:

```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "fcntl.h"

struct pstat{
int ctime;
int etime;
int ttime;
};
int main() {
  int c_pid_h, c_pid_u;
  struct pstat pstat1, pstat2;

  // To Fork a child process
  c_pid_h = fork();
  if (c_pid_h < 0) {
    printf(1,"fork failed\n");
    exit();
  }

  if (c_pid_h == 0) {

    char *args[] = {"head", "input.txt", 0};
    exec(args[0], args);
   printf(1,"exec head failed \n");
    exit();
  } else {
    // The below is for parent process
    if (waitx(c_pid_h, &pstat1) < 0) {
      printf(1,"waitx failed \n");
      exit();
    }
  }

  printf(1, "Process statistics for 'head input.txt':\n");
  printf(1, "  Creation time: %d\n", pstat1.ctime);
  printf(1, "  End time: %d\n", pstat1.etime);
  printf(1, "  Total time: %d\n", pstat1.ttime);


  c_pid_u = fork();
  if (c_pid_u < 0) {
   printf(1,"fork failed \n");
    exit();
  }

  if (c_pid_u == 0) {

    char *args[] = {"uniq", "example.txt", 0};
    exec(args[0], args);
    printf(1,"exec uniq failed\n");
    exit();
  } else {
    // This is the parent process
    if (waitx(c_pid_u, &pstat2) < 0) {
      printf(1,"waitx failed \n");
      exit();
    }
  }

  // Print process statistics for "uniq example.txt".
  printf(1, "\nProcess statistics for 'uniq example.txt':\n");
  printf(1, "  Creation time: %d\n", pstat2.ctime);
  printf(1, "  End time: %d\n", pstat2.etime);
  printf(1, "  Total time: %d\n", pstat2.ttime);

  exit();
  return 0;
}
```

### 3.13. Create `pstat.h` <a name="create-pstath"></a>

Create a new file named `pstat.h` and add the following code:

```c
struct pstat {
  int ctime;
  int etime;
  int ttime;
};
```

### 3.14. Create `ps.c` <a name="create-psc"></a>

Create a new file named `ps.c` and add the following code:

```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "fcntl.h"

int
main(int argc, char *argv[])
{
    // To call the ps system call
    ps();
    exit();
}
```

## 4. Building and Running <a name="building-and-running"></a>

1. Build xv6 with the new modifications. Follow the build instructions provided in the xv6 documentation or repository.

2. After building, make sure to include the newly created utility programs (`head`, `uniq`, `test`, and `ps`) in the `xv6.img` file using the `mkfs` command.

3. Boot xv6 in a virtual machine or emulator (e.g., QEMU).

4. You can now run the new utility programs from the xv6 shell. For example:
   - `head input.txt` will display the first 14 lines of the `input` file.
   - `uniq file.txt` will display unique lines from `file.txt`.
   - `test` will execute the test program, demonstrating the `ps` and `waitx` system calls.

## 5. Conclusion <a name="conclusion"></a>

You have successfully modified the xv6 operating system to include two new system calls (`ps` and `waitx`) and created two utility programs (`head` and `uniq`). These modifications extend the functionality of xv6 and provide additional system monitoring and text processing capabilities.
