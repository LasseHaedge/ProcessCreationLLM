I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?


On Unix-like systems, the canonical and correct way to start a new process from C—regardless of the parent’s address space size—is to use **`fork()` (or `posix_spawn()`) followed by an `exec*()` call**. The child process will replace its address space with the Python interpreter and your script.

Because modern Unix kernels use **copy-on-write (CoW)**, a large parent address space does **not** get physically copied when calling `fork()`; only modified pages are duplicated. For your use case (launching a lightweight Python script), this is safe and standard.

Below are the practical options, from most traditional to most efficient.

---

## Option 1: `fork()` + `exec()` (Most Common)

### Key points

* `fork()` creates a child process.
* `exec()` replaces the child’s memory with Python.
* No actual memory duplication unless pages are written.
* Works everywhere.

### Example

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // Child process
        execlp("python3", "python3", "/path/to/script.py", NULL);

        // Only reached if exec fails
        perror("exec failed");
        exit(EXIT_FAILURE);
    } else {
        // Parent process
        wait(NULL);  // Optional
    }

    return 0;
}
```

### When to use

* General-purpose process creation
* Maximum portability
* Simple logic

---

## Option 2: `posix_spawn()` (Preferred for Large Address Spaces)

If your concern is **very large virtual memory usage**, `posix_spawn()` is often better. It avoids creating a full child process image in user space and is typically implemented inside the kernel.

### Advantages

* Lower overhead than `fork()` for large processes
* Safer in multithreaded programs
* Designed specifically for “run another program”

### Example

```c
#include <spawn.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>

extern char **environ;

int main(void) {
    pid_t pid;
    char *argv[] = {
        "python3",
        "/path/to/script.py",
        NULL
    };

    int status = posix_spawnp(
        &pid,
        "python3",
        NULL,
        NULL,
        argv,
        environ
    );

    if (status != 0) {
        perror("posix_spawn failed");
        exit(EXIT_FAILURE);
    }

    waitpid(pid, NULL, 0);
    return 0;
}
```

### When to use

* Parent process has a huge address space
* Multithreaded applications
* Performance-sensitive systems

---

## Option 3: `vfork()` (Not Recommended)

`vfork()` shares the parent’s address space until `exec()` is called.

### Why to avoid it

* Extremely fragile
* Undefined behavior if child touches memory
* Superseded by `posix_spawn()`

Only use it if you fully understand the constraints.

---

## Notes on Address Space Concerns

* **Linux, BSD, macOS** all use copy-on-write for `fork()`
* Large address space ≠ large physical memory copy
* The cost is mainly **page table duplication**, not RAM
* `posix_spawn()` avoids even that cost

---

## Recommendation Summary

| Scenario               | Recommendation      |
| ---------------------- | ------------------- |
| Simple, portable code  | `fork()` + `exec()` |
| Huge memory footprint  | `posix_spawn()`     |
| Multithreaded parent   | `posix_spawn()`     |
| Legacy/low-level hacks | Avoid `vfork()`     |

---

If you want to:

* Pass arguments or environment variables
* Redirect stdin/stdout
* Run Python embedded instead of as a process
* Avoid blocking the parent

I can provide tailored examples for each case.
