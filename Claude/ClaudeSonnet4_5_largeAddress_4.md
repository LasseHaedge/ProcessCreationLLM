I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



To create a new process from C on a Unix-like system that runs a Python script, you'll use the `fork()` and `exec()` family of system calls. Here's how to do it:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid < 0) {
        // Fork failed
        perror("fork failed");
        return 1;
    }
    else if (pid == 0) {
        // Child process
        // Execute Python script
        execl("/usr/bin/python3", "python3", "script.py", NULL);
        
        // If execl returns, it failed
        perror("execl failed");
        exit(1);
    }
    else {
        // Parent process
        printf("Parent: created child with PID %d\n", pid);
        
        // Wait for child to complete (optional)
        int status;
        waitpid(pid, &status, 0);
        
        if (WIFEXITED(status)) {
            printf("Child exited with status %d\n", WEXITHEXIT_STATUS(status));
        }
    }
    
    return 0;
}
```

**Key points:**

- **`fork()`** creates a new process by duplicating the calling process. It returns 0 in the child, the child's PID in the parent, or -1 on error.

- **`execl()`** replaces the child process's memory space with the Python interpreter. This is perfect for your use case since you want a small Python script, not a copy of your large address space.

- The child process gets a fresh, small address space for Python—it doesn't inherit your large memory footprint after `exec()`.

**Alternative using `execv()` with arguments:**

```c
char *args[] = {
    "python3",
    "script.py",
    "arg1",
    "arg2",
    NULL
};
execv("/usr/bin/python3", args);
```

**If you don't need to wait for the child:**

Simply omit the `waitpid()` call. The child will run independently, and the OS will clean it up when it exits.

**Finding Python's path:**

If Python isn't at `/usr/bin/python3`, use `which python3` to find it, or use `execlp()` which searches the PATH:

```c
execlp("python3", "python3", "script.py", NULL);
```

This approach is efficient because `exec()` replaces the process image entirely, so your large parent process's address space isn't duplicated in memory for the child.
