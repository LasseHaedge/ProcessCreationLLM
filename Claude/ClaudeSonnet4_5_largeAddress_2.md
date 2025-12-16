I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



To create a new process on a Unix-like system using C, you'll use the `fork()` and `exec()` family of functions. Here's how to do it:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid;
    int status;
    
    // Fork a new process
    pid = fork();
    
    if (pid < 0) {
        // Fork failed
        perror("fork failed");
        return 1;
    }
    else if (pid == 0) {
        // Child process
        // Execute the Python script
        execl("/usr/bin/python3", "python3", "script.py", NULL);
        
        // If execl returns, it failed
        perror("execl failed");
        exit(1);
    }
    else {
        // Parent process
        printf("Parent: Created child process with PID %d\n", pid);
        
        // Wait for the child to complete (optional)
        waitpid(pid, &status, 0);
        
        if (WIFEXITED(status)) {
            printf("Child exited with status %d\n", WEXIFEXIT STATUS(status));
        }
    }
    
    return 0;
}
```

**Key points:**

1. **`fork()`** creates a copy of the current process. It returns:
   - `0` in the child process
   - The child's PID in the parent process
   - `-1` on error

2. **`exec()` family** replaces the child's memory space with the new program (Python script). Common variants:
   - `execl()` - list arguments individually
   - `execv()` - pass arguments as array
   - `execlp()` - searches PATH for the executable
   - `execvp()` - combines both features

3. **Copy-on-write optimization**: Modern Unix systems use copy-on-write for `fork()`, so even though your parent has a large address space, the child won't immediately duplicate all that memory. Only when either process writes to memory pages will they be copied.

**Alternative with `execlp()` (searches PATH):**

```c
execlp("python3", "python3", "script.py", NULL);
```

**Passing arguments to the Python script:**

```c
execl("/usr/bin/python3", "python3", "script.py", "arg1", "arg2", NULL);
```

**If you don't want to wait for the child:**

Simply remove the `waitpid()` call. The child will run independently, and the parent can continue its work.
