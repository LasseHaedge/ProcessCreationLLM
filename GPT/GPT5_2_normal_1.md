I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



On a Unix-like system, the canonical approach is to **create a new process using `fork()` and then replace that process image with the Python interpreter using one of the `exec*()` system calls**. This is the same mechanism shells use when you run external programs.

Below is a concise, production-appropriate explanation and example.

---

## High-level process model

1. **`fork()`**

   * Creates a child process.
   * The child is a near-exact copy of the parent.

2. **`exec*()`**

   * Replaces the child process with the Python interpreter running your script.
   * The process ID remains the same, but the program image changes.

3. **`wait()` / `waitpid()` (optional but recommended)**

   * The parent waits for the child to complete and can inspect its exit status.

---

## Minimal working example

### C program (`run_python.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {
        // fork failed
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // Child process
        execlp(
            "python3",          // executable searched in PATH
            "python3",          // argv[0]
            "script.py",        // Python script
            NULL
        );

        // Only reached if exec fails
        perror("execlp");
        exit(EXIT_FAILURE);
    } else {
        // Parent process
        int status;
        waitpid(pid, &status, 0);

        if (WIFEXITED(status)) {
            printf("Python script exited with status %d\n",
                   WEXITSTATUS(status));
        }
    }

    return 0;
}
```

### Python script (`script.py`)

```python
# Example calculation
result = sum(i * i for i in range(1000))
print(result)
```

---

## Compilation and execution

```bash
gcc -Wall -Wextra -o run_python run_python.c
./run_python
```

---

## Key notes and best practices

### Why `exec` instead of `system()`?

* `system()` spawns a shell, which is:

  * Slower
  * Harder to control
  * Risky if arguments are user-supplied
* `fork()` + `exec()` is:

  * Deterministic
  * Secure
  * Idiomatic Unix

### Choosing the right `exec*()` variant

* `execlp()` – convenient, uses `PATH`
* `execv()` – explicit path and argument array
* `execve()` – full control, including environment

Example with arguments:

```c
char *args[] = {"python3", "script.py", "arg1", "arg2", NULL};
execvp("python3", args);
```

---

## Capturing Python output (optional)

If you need to **read the calculation results back into C**, use:

* `pipe()`
* `dup2()`
* Redirect stdout in the child before `exec()`

This is a common pattern and can be added cleanly if needed.

---

## Summary

* Use `fork()` to create a process.
* Use `exec*()` to run the Python script.
* Optionally `waitpid()` to synchronize.
* This is the standard, correct Unix process model.

If you want:

* Bidirectional communication
* Long-running Python workers
* Background or daemonized execution

say so, and I can tailor the design accordingly.
