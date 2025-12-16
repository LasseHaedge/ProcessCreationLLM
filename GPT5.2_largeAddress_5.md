I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?


On Unix-like systems, the canonical and safest way to start a new process that runs a Python script from a C program is to **fork a child process** and then **replace its address space with the Python interpreter** using one of the `exec*()` family of functions.

This approach is specifically designed to avoid duplicating a large address space unnecessarily and is well-supported by the kernel via **copy-on-write (COW)** semantics.

---

## Recommended Process Model

### High-level sequence

1. **`fork()`**

   * Creates a child process.
   * Memory pages are *not copied immediately*; they are shared until modified (copy-on-write).

2. **`execve()` / `execvp()`**

   * Replaces the child’s entire address space with the Python interpreter.
   * The large parent address space is discarded in the child.

This is efficient even if the parent process has a very large virtual memory footprint.

---

## Minimal Example in C

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        // Child process
        execlp("python3",
               "python3",
               "/path/to/script.py",
               (char *)NULL);

        // Only reached if exec fails
        perror("exec failed");
        _exit(EXIT_FAILURE);
    }

    // Parent process
    waitpid(pid, NULL, 0);
    return EXIT_SUCCESS;
}
```

---

## Why This Is Efficient with Large Address Spaces

* `fork()` **does not copy memory eagerly**
* Linux, BSD, and macOS use **copy-on-write**
* After `exec()`, the child discards the inherited mappings entirely
* Result: minimal overhead regardless of parent size

In practice, the cost is proportional to the number of page tables, *not* the size of memory actually used.

---

## Alternative: `posix_spawn()` (When Appropriate)

If you want to avoid `fork()` semantics altogether (e.g., in memory-constrained environments or very large processes), consider:

```c
#include <spawn.h>
#include <sys/wait.h>

extern char **environ;

posix_spawnp(
    &pid,
    "python3",
    NULL,
    NULL,
    (char *const[]){"python3", "/path/to/script.py", NULL},
    environ
);
```

### Notes on `posix_spawn()`

* Often implemented as `vfork()` + `exec()` internally
* Lower overhead in some libc implementations
* Fewer foot-guns (signal handlers, mutex state)
* More restrictive than `fork()`

On **Linux/glibc**, `fork()` is usually fast enough unless you are forking extremely frequently.

---

## Important Caveats

### After `fork()` (before `exec()`), do **not**:

* Allocate memory
* Use non–async-signal-safe functions
* Touch large data structures
* Call Python APIs

Keep the child path minimal.

---

## Passing Arguments or Environment

Arguments:

```c
execlp("python3", "python3", "script.py", "arg1", "arg2", NULL);
```

Environment:

```c
execle("python3", "python3", "script.py", NULL, envp);
```

---

## Summary

| Requirement                 | Solution               |
| --------------------------- | ---------------------- |
| Large parent address space  | `fork()` + `exec()`    |
| Minimal memory duplication  | Copy-on-write          |
| Simple, portable, idiomatic | `fork()` + `exec*()`   |
| Avoid fork in special cases | `posix_spawn()`        |
| Run Python script from C    | `exec("python3", ...)` |

If you need **bidirectional IPC**, **job control**, or **parallel execution**, the design may change slightly—clarify if that applies.
