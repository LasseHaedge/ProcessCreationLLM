I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



In a Unix-like environment, creating a new process to run an external script (like Python) from a C program involves a three-step workflow: **forking**, **executing**, and **waiting**.

The standard way to achieve this is by using the `fork()` and `exec()` family of system calls.

-----

### The Process Lifecycle

When you want to run a Python script from C, the operating system doesn't just "start" a new process from scratch. Instead, it copies the current process and then replaces that copy with the new program.

1.  **`fork()`**: This creates a "child" process that is an exact duplicate of your C program (the "parent").
2.  **`execvp()`**: This replaces the child process's memory and code with the Python interpreter and your script.
3.  **`wait()`**: This tells the parent process to pause until the child finishes its calculations.

-----

### Practical Code Example

Here is a concise implementation. This code assumes you have a script named `calc.py` in the same directory.

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
        // --- Inside Child Process ---
        printf("Child: Starting Python script...\n");
        
        // Arguments: "python3", "script_name", and NULL to terminate the list
        char *args[] = {"python3", "calc.py", NULL};
        
        // execvp searches for 'python3' in your PATH
        execvp(args[0], args);

        // If execvp returns, it means an error occurred
        perror("execvp failed");
        exit(1);
    } 
    else {
        // --- Inside Parent Process ---
        int status;
        printf("Parent: Waiting for child (PID %d) to finish...\n", pid);
        
        waitpid(pid, &status, 0);
        
        if (WIFEXITED(status)) {
            printf("Parent: Child finished with exit code %d\n", WEXITSTATUS(status));
        }
    }

    return 0;
}
```

### Key Components Explained

  * **`pid_t pid`**: The return value of `fork()`. In the parent, it's the ID of the child. In the child, it is `0`. This is how the code "knows" which branch it is in.
  * **`execvp(file, argv)`**:
      * The `p` stands for **PATH**, meaning it will look for `python3` automatically in your system folders (like `/usr/bin/`).
      * Once called, the C code in the child process stops existing; it becomes the Python process.
  * **`waitpid()`**: This is crucial to prevent "zombie processes." It ensures the parent cleans up the child's resources after it finishes.

-----

### A Simpler Alternative: `system()`

If you don't need fine-grained control (like capturing the process ID or redirecting specific data streams), you can use the `system()` function:

```c
system("python3 calc.py");
```

While much easier, `system()` is generally considered less secure and less flexible because it invokes a shell (`/bin/sh`) to run the command, which adds overhead and potential security risks if you are using user-provided strings.

**Would you like me to show you how to pass specific numeric arguments from your C code into the Python script's calculation?**
