I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



In a Unix-like environment, creating a new process to run an external script (like Python) involves a specific workflow: **Forking** a child process and then **Executing** the new program within that child.

### The `fork()` and `exec()` Workflow

To run a Python script from a C program, you don't just "open" the script; you replace a process with the Python interpreter. The standard pattern follows these steps:

1.  **`fork()`**: Creates a duplicate of your current process.
2.  **`execvp()`**: In the child process, this replaces the current memory image with the Python interpreter.
3.  **`wait()`**: The parent process stays alive to monitor the child and clean up resources when it finishes.

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
        // Fork failed
        perror("fork failed");
        return 1;
    } 
    else if (pid == 0) {
        // --- CHILD PROCESS ---
        printf("Child: Starting Python script...\n");

        // Arguments must be a NULL-terminated array of strings
        // "python3" is the command, "calc.py" is the argument
        char *args[] = {"python3", "calc.py", NULL};

        // execvp searches the PATH for python3
        execvp(args[0], args);

        // If execvp returns, it means an error occurred
        perror("execvp failed");
        exit(1);
    } 
    else {
        // --- PARENT PROCESS ---
        int status;
        printf("Parent: Waiting for child (PID %d) to finish...\n", pid);
        
        // Wait for the specific child process to finish
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
| **`fork()`** | Returns `0` to the child and the **Process ID (PID)** of the child to the parent. This is how the code "branches." |
| **`execvp()`** | Replaces the current process image. The `p` in `vp` stands for **PATH**, meaning it will automatically find where `python3` is installed (e.g., `/usr/bin/python3`). |
| **`waitpid()`** | Prevents "zombie processes." It ensures the parent collects the exit status of the child before continuing. |

### Important Considerations

  * **Error Handling**: Always check if `pid < 0`. This can happen if the system has reached its maximum process limit.
  * **Permissions**: Ensure your Python script is readable. If you want to run it as `./calc.py` directly, ensure it has a shebang line (e.g., `#!/usr/bin/env python3`) and execute permissions (`chmod +x calc.py`).
  * **Communication**: If you need to send data to the Python script or read its output back into C, you would need to use **pipes** (`pipe()`) before calling `fork()`.

Would you like me to show you how to set up a **pipe** so your C program can capture the output of the Python calculation?
