I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



On a Unix-like system, the canonical approach is:

1. **Create a new process** using `fork()`.
2. **Replace the child process image** with the Python interpreter running your script using one of the `exec*()` family calls.
3. Optionally **synchronize** with the child using `wait()` / `waitpid()`.

Below is a concise, production-appropriate explanation with a concrete example.

---

## Conceptual Flow

1. Your C program calls `fork()`.

   * Parent continues executing.
   * Child is a near-identical copy.
2. In the **child**, call `execve()` / `execvp()` to run:

   ```
   python3 your_script.py
   ```
3. In the **parent**, optionally wait for completion or continue independently.

---

## Minimal Working Example

### Python script (`worker.py`)

```python
#!/usr/bin/env python3

import sys

def main():
    x = 5
    y = 7
    print(f"Result: {x + y}")

if __name__ == "__main__":
    main()
```

Make it executable:

```sh
chmod +x worker.py
```

---

### C Program (`launcher.c`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {
        // Fork failed
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        // Child process
        char *argv[] = {
            "python3",
            "worker.py",
            NULL
        };

        execvp("python3", argv);

        // execvp only returns on error
        perror("execvp");
        _exit(EXIT_FAILURE);
    }

    // Parent process
    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status)) {
        printf("Child exited with status %d\n", WEXITSTATUS(status));
    }

    return EXIT_SUCCESS;
}
```

Compile and run:

```sh
gcc launcher.c -o launcher
./launcher
```

---

## Key System Calls Explained

### `fork()`

* Creates a new process.
* Returns:

  * `0` in the child
  * Child PID in the parent
  * `-1` on error

### `execvp()`

* Replaces the current process image.
* Uses `PATH` lookup (preferred over `execv()`).
* Does **not** return on success.

### `waitpid()`

* Allows the parent to wait for the child.
* Prevents zombie processes.

---

## Variations You May Consider

### Run Python in the Background (No Wait)

Simply omit `waitpid()` in the parent.

### Pass Arguments to Python

```c
char *argv[] = {
    "python3",
    "worker.py",
    "42",
    "13",
    NULL
};
```

### Capture Python Output

Use `pipe()`, `dup2()`, and `read()` to redirect stdout/stderr.

### Avoid `fork()` Overhead

If you only want Python logic, embedding Python via the **CPython C API** may be more appropriate, though more complex.

---

## Best Practices

* Always check return values of `fork()` and `exec*()`
* Use `_exit()` in the child on failure
* Prefer `execvp()` unless you need absolute paths
* Use `waitpid()` to avoid zombies

---

If you want, I can also show:

* Redirecting stdout/stderr to your C program
* Bidirectional communication via pipes
* Using `posix_spawn()` instead of `fork()`
* Embedding Python directly into a C application
