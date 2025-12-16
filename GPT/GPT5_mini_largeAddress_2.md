I got a process with a large address space. Now I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



Got it. You want to start a new process from a C program on a Unix-like system, running a Python script, while your original process has a large address space. There are a few ways to do this in C. Let me walk you through the standard approach using `fork()` and `exec()`.

---

### 1. Using `fork()` and `execlp()` (or other `exec*` functions)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();  // Create a new process

    if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // Child process: run the Python script
        execlp("python3", "python3", "/path/to/script.py", (char *)NULL);

        // If execlp returns, there was an error
        perror("execlp failed");
        exit(EXIT_FAILURE);
    } else {
        // Parent process: optionally wait for the child to finish
        int status;
        waitpid(pid, &status, 0);
        if (WIFEXITED(status)) {
            printf("Child exited with code %d\n", WEXITSTATUS(status));
        }
    }

    return 0;
}
```

**Explanation:**

* `fork()` creates a new process that is almost an exact copy of the parent, including memory.

  * On modern Unix systems, `fork()` uses **copy-on-write**, so the large memory of the parent is **not immediately duplicated**, which is efficient.
* `execlp()` replaces the child process image with a new program—in this case, `python3 script.py`.
* The parent can either continue execution or wait for the child using `waitpid()`.

---

### 2. Using `posix_spawn()` (more efficient for very large parents)

When the parent process has a **very large address space**, `fork()` can still be heavy because the kernel needs to set up page tables, even with copy-on-write. In that case, `posix_spawn()` is preferred:

```c
#include <stdio.h>
#include <stdlib.h>
#include <spawn.h>
#include <sys/wait.h>

extern char **environ;

int main() {
    pid_t pid;
    char *argv[] = {"python3", "/path/to/script.py", NULL};

    if (posix_spawn(&pid, "/usr/bin/python3", NULL, NULL, argv, environ) != 0) {
        perror("posix_spawn failed");
        exit(EXIT_FAILURE);
    }

    int status;
    waitpid(pid, &status, 0);
    if (WIFEXITED(status)) {
        printf("Child exited with code %d\n", WEXITSTATUS(status));
    }

    return 0;
}
```

**Advantages:**

* `posix_spawn()` avoids duplicating the parent’s address space entirely.
* It’s more memory-efficient for very large processes.

---

### 3. Notes

* Make sure the Python executable path is correct (`which python3`).
* Pass arguments as a `char *argv[]` array for `exec` or `posix_spawn`.
* If the Python script needs its own environment variables, you can modify `environ` or pass a custom `envp`.

---

If your parent process’s memory usage is very large, **`posix_spawn()` is safer** because it avoids potential `fork()` failures due to memory pressure.

---

If you want, I can also show a **version that runs the Python script completely in the background without blocking the parent**, so the original process keeps running without waiting. That’s often useful for lightweight task runners. Do you want me to show that?
