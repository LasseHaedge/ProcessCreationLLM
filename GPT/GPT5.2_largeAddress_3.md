I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?


On Unix-like systems, the standard and correct way to create a new process from C—regardless of the parent’s address space size—is to use **`fork()`** followed by an **`exec*()`** call to replace the child’s address space with the Python interpreter running your script.

Modern kernels use **copy-on-write (CoW)** semantics, so a large address space in the parent does *not* mean the entire memory is duplicated when you fork. Only modified pages are copied, making this approach efficient.

---

## Canonical Process Creation Model

1. **`fork()`**
   Creates a child process.
2. **`exec()`**
   Replaces the child process image with Python.
3. **(Optional) `waitpid()`**
   Parent waits for completion or manages the child lifecycle.

---

## Minimal, Correct Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        // Child process
        execlp(
            "python3",           // Program to execute
            "python3",           // argv[0]
            "script.py",         // Script name
            "arg1", "arg2",      // Optional arguments
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
    }

    return EXIT_SUCCESS;
}
```

---

## Why This Works Well With Large Address Spaces

* **Copy-on-write** ensures memory is not duplicated unless modified.
* `exec()` immediately discards the child’s inherited memory and loads Python.
* The cost of `fork()` is mainly page table duplication, which is manageable.

---

## When to Use `posix_spawn()` Instead

If you:

* Are on systems with extremely large page tables
* Want slightly faster startup
* Don’t need complex child setup

Then `posix_spawn()` is a good alternative:

```c
#include <spawn.h>
#include <sys/wait.h>

extern char **environ;

int main(void)
{
    pid_t pid;
    char *argv[] = {"python3", "script.py", NULL};

    int rc = posix_spawnp(
        &pid,
        "python3",
        NULL,
        NULL,
        argv,
        environ
    );

    if (rc != 0) {
        perror("posix_spawnp");
        return EXIT_FAILURE;
    }

    waitpid(pid, NULL, 0);
    return EXIT_SUCCESS;
}
```

`posix_spawn()` avoids a traditional fork in some implementations and can be more memory-efficient.

---

## Important Notes

* Always call `_exit()` (not `exit()`) if `exec()` fails in the child.
* Use `execvp()` / `execlp()` if you want `$PATH` resolution.
* If you do not want the parent to block, omit `waitpid()` or use non-blocking waits.
* For high-frequency task execution, consider a **worker process pool** instead of repeated forks.

---

## Summary

| Requirement              | Recommended Approach            |
| ------------------------ | ------------------------------- |
| Standard Unix            | `fork()` + `exec()`             |
| Very large address space | Still `fork()` + `exec()` (CoW) |
| Minimal overhead         | `posix_spawn()`                 |
| Frequent tasks           | Persistent worker processes     |

If you want, I can also show:

* Redirecting stdin/stdout to the Python process
* Passing data via pipes or shared memory
* Managing many Python workers efficiently
