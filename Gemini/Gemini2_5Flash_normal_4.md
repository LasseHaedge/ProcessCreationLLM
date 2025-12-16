I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



Creating a new process in a Unix-like system is primarily handled through the `fork()` and `exec()` family of system calls. In C, this involves creating a duplicate of your current process and then replacing that duplicate with the Python interpreter.

### The Core Mechanism: Fork and Exec

To run an external script, you follow a "spawn" pattern:

1.  **`fork()`**: Creates a child process. Both the parent and child continue execution from the point of the fork.
2.  **`execvp()`**: Replaces the child process's memory space with a new program (the Python interpreter).
3.  **`wait()`**: The parent process usually waits for the child to finish to avoid "zombie" processes.

-----

### Implementation Example

Here is a concise C program that launches a Python script named `calc.py`.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();

    if (pid < 0) {
        // Error occurred
        perror("Fork failed");
        return 1;
    } 
    else if (pid == 0) {
        // This is the Child Process
        printf("Child: Starting Python script...\n");
        
        // Arguments for execvp: 
        // 1. The command, 2. Array of arguments (must end with NULL)
        char *args[] = {"python3", "calc.py", NULL};
        
        execvp(args[0], args);

        // execvp only returns if there is an error
        perror("execvp failed");
        exit(1);
    } 
    else {
        // This is the Parent Process
        printf("Parent: Waiting for child (PID %d) to finish...\n", pid);
        
        int status;
        waitpid(pid, &status, 0); // Wait for the specific child process
        
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
| **`fork()`** | Returns `0` to the child and the **Child's PID** to the parent. This is how you tell them apart. |
| **`execvp()`** | The `p` stands for "path," meaning it will search your `$PATH` for `python3` automatically. |
| **`waitpid()`** | Ensures the parent doesn't finish before the child, allowing you to collect the child's exit code. |

### Important Considerations

  * **Permissions**: Ensure your Python script is readable. If you want to run it as `./calc.py` directly, ensure it has a shebang (`#!/usr/bin/env python3`) and execute permissions (`chmod +x calc.py`).
  * **Error Handling**: Always check if `pid < 0`. Systems can fail to fork if they run out of process slots or memory.
  * **Communication**: If you need to send data to the Python script or get data back (beyond a simple exit code), you would need to look into **Pipes** (`pipe()`) or **Shared Memory**.

Would you like me to show you how to set up a pipe so the C program can read the output of the Python script?
