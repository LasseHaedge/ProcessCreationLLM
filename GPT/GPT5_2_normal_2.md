I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



On a Unix-like system, the standard and correct way to start a new process from C is to **fork** the current process and then **exec** a new program—in your case, the Python interpreter running a script.

Below is a structured, production-grade explanation with a minimal but complete example.

---

## Conceptual Overview

1. **`fork()`**

   * Creates a new process (child) as a copy of the calling process.
   * Returns:

     * `0` in the child
     * Child PID in the parent
     * `-1` on error

2. **`exec*()` family**

   * Replaces the current process image with a new program.
   * Common choice: `execvp()` or `execlp()`
   * If `exec` succeeds, it does **not return**.

3. **Parent process**

   * Optionally waits for the child using `wait()` or `waitpid()`

---

## Minimal Working Example

This example starts a Python script called `worker.py`.

### Python script (`worker.py`)

```python
#!/usr/bin/env python3

import time

result = sum(i*i for i in range(100000))
print(f"Result: {result}")
time.sleep(2)
```

Make it executable:

```sh
chmod +x worker.py
```

---

### C Program

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        /* Child process */
        execlp("python3", "python3", "./worker.py", NULL);

        /* Only reached if exec fails */
        perror("exec failed");
        exit(EXIT_FAILURE);
    } else {
        /* Parent process */
        printf("Started child process with PID %d\n", pid);

        /* Optional: wait for completion */
        waitpid(pid, NULL, 0);
        printf("Child process finished\n");
    }

    return 0;
}
```

Compile:

```sh
gcc -Wall -Wextra -o launcher launcher.c
```

Run:

```sh
./launcher
```

---

## Key Points

* **`fork()` creates the process**
* **`exec()` loads Python and your script**
* The Python process is fully independent once started
* The parent can:

  * wait for it
  * ignore it
  * monitor it
  * daemonize it

---

## Variants and Improvements

### 1. Pass Arguments to Python

```c
execlp("python3", "python3", "worker.py", "arg1", "arg2", NULL);
```

Access in Python via `sys.argv`.

---

### 2. Redirect Output (e.g., to a file)

```c
freopen("output.log", "w", stdout);
execlp("python3", "python3", "./worker.py", NULL);
```

---

### 3. Run in Background (No `waitpid`)

Simply omit the `waitpid()` call.

---

### 4. Use `posix_spawn()` (Modern Alternative)

For simple process creation with fewer side effects:

```c
#include <spawn.h>
extern char **environ;

posix_spawn(NULL, "/usr/bin/python3",
            NULL, NULL,
            (char *[]){"python3", "worker.py", NULL},
            environ);
```

This avoids `fork()` overhead and is often preferred on constrained systems.

---

## When *Not* to Use `system()`

Do **not** use:

```c
system("python3 worker.py");
```

* Harder to control
* Shell injection risk
* Poor error handling

---

## Summary

| Task                 | Mechanism           |
| -------------------- | ------------------- |
| Create process       | `fork()`            |
| Run Python           | `exec*()`           |
| Wait for completion  | `waitpid()`         |
| Pass args            | `exec*()` arguments |
| Background execution | No wait             |

If you want to:

* daemonize this process
* manage multiple workers
* communicate via pipes or shared memory

say so, and I can walk you through that architecture.
