I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?


On a Unix-like system, the canonical and correct way to start a new process from C—regardless of the parent’s address-space size—is to use **`fork()` (or `posix_spawn()`), followed by an `exec*()` family call**. The child process will then execute the Python interpreter with your script.

Below is a practical, production-appropriate explanation with concrete code examples and trade-offs.

---

## 1. The Standard Model: `fork()` + `exec()`

### Why this works even with a large address space

Modern Unix kernels (Linux, *BSD, macOS) implement **copy-on-write (CoW)** for `fork()`:

* The child initially **shares** the parent’s memory pages.
* Pages are copied **only if modified**.
* If you immediately call `exec()`, **almost no memory is duplicated**.

Therefore, a large address space is *not* a blocker in practice.

---

### Minimal Example: Run a Python Script

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
        perror("fork");
        return EXIT_FAILURE;
    }

    if (pid == 0) {
        // Child process
        execlp(
            "python3",          // executable (searched in PATH)
            "python3",          // argv[0]
            "/path/to/script.py",
            "arg1",
            "arg2",
            (char *)NULL
        );

        // Only reached if exec fails
        perror("execlp");
        _exit(127);
    }

    // Parent process
    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status)) {
        printf("Python exited with code %d\n", WEXITSTATUS(status));
    }

    return EXIT_SUCCESS;
}
```

### Key points

* `fork()` creates the child
* `execlp()` replaces the child’s memory with Python
* `_exit()` avoids running parent cleanup handlers in the child
* `waitpid()` synchronizes (optional)

---

## 2. Preferred for Large Processes: `posix_spawn()`

If your process is **very large**, multi-threaded, or you want to avoid even the temporary CoW cost, **`posix_spawn()`** is often superior.

### Advantages

* No full `fork()` semantics
* Safer in multi-threaded programs
* Lower overhead on some platforms

### Example

```c
#include <spawn.h>
#include <sys/wait.h>
#include <stdio.h>
#include <stdlib.h>

extern char **environ;

int main(void)
{
    pid_t pid;
    char *argv[] = {
        "python3",
        "/path/to/script.py",
        "arg1",
        NULL
    };

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

    int status;
    waitpid(pid, &status, 0);

    return EXIT_SUCCESS;
}
```

### When to choose `posix_spawn()`

* Parent has **huge memory mappings**
* Parent is **multi-threaded**
* You do not need complex child-setup logic before `exec`

---

## 3. Avoiding Common Mistakes

### Do **NOT**

* `system("python3 script.py")` → shell injection, poor control
* Write your own ELF loader
* Worry about copying the address space if you `exec()` immediately

### Use `_exit()` in the child after exec failure

This avoids:

* Double flushing stdio buffers
* Running parent atexit handlers

---

## 4. Redirecting I/O (Common Requirement)

Example: capture Python output

```c
int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
dup2(fd, STDOUT_FILENO);
dup2(fd, STDERR_FILENO);
close(fd);
```

Do this **in the child before `exec()`**.

---

## 5. Summary Recommendation

| Scenario                             | Best Approach                |
| ------------------------------------ | ---------------------------- |
| Simple process                       | `fork()` + `exec()`          |
| Very large or multi-threaded process | `posix_spawn()`              |
| Need environment control             | `execve()` / `posix_spawn()` |
| Performance-critical                 | `posix_spawn()`              |

---

If you want, I can also provide:

* A **thread-safe** variant
* An example with **pipes for IPC**
* A comparison of **Linux vs macOS behavior**
* Error-handling patterns suitable for daemons

Just specify your constraints.
