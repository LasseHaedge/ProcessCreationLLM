I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



Creating a new process in a Unix-like system is primarily handled through the `fork()` and `exec()` family of system calls. In your case, you'll use `fork()` to create a duplicate process and then `execvp()` (or a similar variant) to transform that child process into a Python interpreter running your script.

### The Core Mechanism: Fork and Exec

The workflow follows a standard pattern:

1.  **`fork()`**: Creates a copy of the current process.
2.  **Child Process**: Uses `exec` to replace its memory space with the Python interpreter.
3.  **Parent Process**: Continues running and usually waits for the child to finish using `wait()`.

### Implementation Example

Here is a clean implementation. This C program spawns a process to run a Python script named `calc.py`.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        // Error occurred
        perror("fork failed");
        return 1;
    } 
    else if (pid == 0) {
        // This is the Child Process
        printf("Child: Launching Python script...\n");

        // Arguments for the execution: 
        // 1. The command ("python3")
        // 2. The script name ("calc.py")
        // 3. NULL terminator (required)
        char *args[] = {"python3", "calc.py", NULL};

        execvp(args[0], args);

        // If execvp returns, it means an error occurred
        perror("execvp failed");
        exit(1);
    } 
    else {
        // This is the Parent Process
        int status;
        printf("Parent: Waiting for child (PID %d) to finish...\n", pid);
        
        // Wait for the specific child process to complete
        waitpid(pid, &status, 0);

        if (WIFEXITED(status)) {
            printf("Parent: Child exited with status %d\n", WEXITSTATUS(status));
        }
    }

    return 0;
}
```

-----

### Key Functions Explained

| Function | Purpose |
| :--- | :--- |
| **`fork()`** | Returns `0` to the child and the **Process ID** of the child to the parent. |
| **`execvp()`** | Replaces the current process image. The `p` version searches your `PATH`, so you don't need the full path to the Python binary. |
| **`waitpid()`** | Prevents "zombie processes" by ensuring the parent collects the exit status of the child. |

### Important Considerations

  * **Error Handling**: Always check if `pid < 0`. This happens if the system is out of memory or has reached the process limit.
  * **The "Replace" Nature of Exec**: Remember that `execvp` never returns if it is successful. Anything you write in the code after `execvp` will only execute if the Python script failed to start.
  * **Environment**: If your Python script requires a specific virtual environment, you may need to provide the full path to the `python` binary inside that environment's `bin` folder.

Would you like me to show you how to set up **pipes** so the C program can read the output of the Python script directly?
