I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



To create a new process from C that runs a Python script on a Unix-like system, you'll use the `fork()` and `exec()` family of functions. Here's how to do it:

## Basic Approach

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid == -1) {
        // Fork failed
        perror("fork failed");
        return 1;
    }
    else if (pid == 0) {
        // Child process
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
            printf("Child exited with status %d\n", WEXITSTATUS(status));
        }
    }
    
    return 0;
}
```

## Key Points

**Why this is efficient for your large address space:**
- `fork()` uses copy-on-write semantics, so the child process shares the parent's memory pages until either modifies them
- `exec()` immediately replaces the child's address space with the Python interpreter, so even less memory is actually copied
- This means creating the new process is quite cheap despite your large parent process

**Common variations:**

If you need to pass arguments to the Python script:
```c
execl("/usr/bin/python3", "python3", "script.py", "arg1", "arg2", NULL);
```

If you want to use the system's PATH to find Python:
```c
execlp("python3", "python3", "script.py", NULL);
```

If you have arguments in an array:
```c
char *args[] = {"python3", "script.py", "arg1", NULL};
execvp("python3", args);
```

You can also use `posix_spawn()` as a more modern alternative that combines fork+exec in one call and can be more efficient in some cases.

Would you like me to show you a more complete example with error handling, or how to capture the Python script's output?
