I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



On Unix-like systems, creating a new process to run a Python script from C follows a well-established **fork–exec** pattern. The parent process creates a child, and the child replaces its process image with the Python interpreter executing your script.

Below is the canonical and production-appropriate approach.

---

## High-Level Flow

1. **fork()**
   Creates a child process.
2. **exec*** (in the child)
   Replaces the child with the Python interpreter.
3. **waitpid()** (in the parent)
   Optionally waits for the child to complete and collects its exit status.

---

## Minimal Working Example

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
        execlp(
            "python3",          // Executable (searched in PATH)
            "python3",          // argv[0]
            "script.py",        // Script to run
            "arg1",             // Optional script arguments
            "arg2",
            (char *)NULL
        );

        // Only reached if exec fails
        perror("execlp");
        _exit(EXIT_FAILURE);
    }

    // Parent process
    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status)) {
        printf("Child exited with status %d\n", WEXITSTATUS(status));
    } else {
        printf("Child terminated abnormally\n");
    }

    return EXIT_SUCCESS;
}
```

---

## Key Points Explained

### `fork()`

* Returns:

  * `< 0` on error
  * `0` in the child
  * Child PID in the parent
* Child inherits file descriptors, environment, and working directory.

### `execlp()`

* Replaces the current process image.
* `execlp` searches `PATH`, which is typically desirable for `python3`.
* If `exec` succeeds, it **never returns**.

Other valid variants:

* `execv()` — pass arguments as an array
* `execve()` — full control over environment
* `posix_spawn()` — often preferable for simple use cases (see below)

---

## Using `posix_spawn()` (Modern Alternative)

If you do not need complex pre-exec logic, `posix_spawn()` is cleaner and avoids `fork()` pitfalls:

```c
#include <spawn.h>
#include <sys/wait.h>
#include <stdio.h>

extern char **environ;

int main(void) {
    pid_t pid;
    char *argv[] = {
        "python3",
        "script.py",
        NULL
    };

    if (posix_spawnp(&pid, "python3", NULL, NULL, argv, environ) != 0) {
        perror("posix_spawnp");
        return 1;
    }

    waitpid(pid, NULL, 0);
    return 0;
}
```

**Advantages**

* Lower overhead
* Safer in multi-threaded programs
* Recommended by POSIX for straightforward process creation

---

## Capturing Output (Optional)

If you want to read results from the Python script:

* Use `pipe()` before `fork()`
* Redirect `stdout`/`stderr` in the child via `dup2()`
* Read from the pipe in the parent

If that is relevant, I can provide a concrete example.

---

## When This Approach Is Appropriate

This design is suitable when:

* The Python work is independent
* Execution time is non-trivial
* You want process isolation
* Python dependencies should remain separate from your C binary

For extremely frequent or very small calculations, embedding Python (via the Python C API) may be more appropriate, but that has a significantly higher integration cost.

---

## Summary

* Use **`fork()` + `exec()`** for classic Unix process creation
* Prefer **`posix_spawn()`** for simple, modern code
* Python runs as a fully separate process
* Parent can monitor, wait, and capture results as needed

If you want to:

* Pass structured data
* Capture numeric results
* Run multiple scripts concurrently
* Manage long-running workers

…say so, and I will tailor the design accordingly.
