I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



Creating a new process in a Unix-like environment (Linux, macOS, etc.) involves the fundamental concept of **forking**. Since you want to run an external Python script, you will combine `fork()` with the `exec` family of functions.

Here is the breakdown of how to achieve this.

### 1\. The Core Workflow

To run a separate program from a C parent process, you follow a "Fork-Exec" pattern:

1.  **`fork()`**: Creates a duplicate of the current process.
2.  **`execvp()`**: Replaces the duplicate process with the Python interpreter and your script.
3.  **`wait()`**: (Optional) The parent process waits for the Python script to finish.

-----

### 2\. Example Implementation

This code demonstrates how to spawn a child process to run a script named `calc.py`.

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
        // --- Child Process ---
        printf("Child: Starting Python script...\n");
        
        // Arguments: "python3", "script_name", NULL
        char *args[] = {"python3", "calc.py", NULL};
        
        // execvp replaces the current process image with Python
        execvp(args[0], args);

        // If execvp returns, it means an error occurred
        perror("execvp failed");
        exit(1);
    } 
    else {
        // --- Parent Process ---
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

### 3\. Key Functions Explained

  * **`fork()`**: This is unique because it returns twice. It returns `0` to the new child process and the **Process ID (PID)** of the child to the parent.
  * **`execvp(file, argv)`**: This looks for the executable `python3` in your system's `PATH`. The `argv` array must be NULL-terminated. Once called, the C code in the child process stops, and the Python code takes over its memory space.
  * **`waitpid()`**: Without this, the child process might become a "zombie" process if it finishes before the parent. It also allows the parent to collect the exit code from your Python script.

### 4\. Important Considerations

  * **Pathing**: If `calc.py` is not in the same directory where you run the C executable, provide the full path (e.g., `/home/user/scripts/calc.py`).
  * **Permissions**: Ensure the Python script is readable. You don't necessarily need "execute" permissions on the `.py` file if you are calling `python3` as the primary executable.
  * **Performance**: Forking is very fast in Unix due to "Copy-on-Write" (CoW) memory management, making it an efficient way to handle sub-tasks.

-----

Would you like me to show you how to pass specific calculation data from the C program to the Python script using **pipes**?
