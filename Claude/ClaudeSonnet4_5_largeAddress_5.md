I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



To create a new process from C on a Unix-like system, you'll use `fork()` and `exec()`. Here's how:

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
        
        // If exec fails, we'll reach here
        perror("execl failed");
        exit(1);
    }
    else {
        // Parent process
        printf("Parent: Created child process with PID %d\n", pid);
        
        // Wait for child to complete
        int status;
        waitpid(pid, &status, 0);
        
        if (WIFEXITED(status)) {
            printf("Child exited with status %d\n", WEXITSTATUS(status));
        }
    }
    
    return 0;
}
```

**Key points:**

1. **`fork()`** creates a copy of your process. It returns:
   - `0` in the child process
   - The child's PID in the parent process
   - `-1` on error

2. **`exec()` family** replaces the child's memory space with the new program (Python). Use `execl()`, `execv()`, `execvp()`, etc. depending on how you want to pass arguments. The `exec()` call doesn't return on success—the process becomes the new program.

3. **`waitpid()`** makes the parent wait for the child to finish, preventing zombie processes.

**Why this is efficient for your use case:** Even though `fork()` conceptually copies your large address space, modern Unix systems use **copy-on-write (COW)**. The child process initially shares memory pages with the parent. Since you immediately call `exec()`, which replaces the entire address space with Python, no actual copying of your large memory occurs.

If you need to pass arguments to the Python script:
```c
execl("/usr/bin/python3", "python3", "script.py", "arg1", "arg2", NULL);
```
