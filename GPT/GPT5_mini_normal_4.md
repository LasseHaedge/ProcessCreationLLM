I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



You want to start a new process from a C program on a Unix-like system to run a Python script. There are several ways to do this in C, depending on how much control you need over the child process. The most common approach is using `fork()` and `exec()` family functions. Here's a step-by-step explanation with an example.

---

### **1. Using `fork()` + `execlp()`**

This is the standard Unix way:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();  // Create a new process

    if (pid < 0) {
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        // Child process
        // Replace the child process with the Python script
        execlp("python3", "python3", "script.py", (char *)NULL);

        // If execlp returns, there was an error
        perror("execlp failed");
        exit(1);
    } else {
        // Parent process
        int status;
        waitpid(pid, &status, 0);  // Wait for child to finish
        if (WIFEXITED(status)) {
            printf("Child exited with status %d\n", WEXITSTATUS(status));
        }
    }

    return 0;
}
```

**Explanation:**

* `fork()` creates a new process.
* In the child process (`pid == 0`), `execlp()` replaces the process image with the Python interpreter running your script.
* The parent waits for the child to finish using `waitpid()`.

---

### **2. Using `system()` (simpler, less control)**

If you don’t need advanced control over the process:

```c
#include <stdlib.h>
#include <stdio.h>

int main() {
    int ret = system("python3 script.py");
    if (ret == -1) {
        perror("system failed");
    } else {
        printf("Python script finished with exit code %d\n", WEXITSTATUS(ret));
    }
    return 0;
}
```

**Notes:**

* `system()` is easier but less flexible.
* It runs the command in a shell, which can be a slight security risk if command strings include untrusted input.

---

### **3. Using `posix_spawn()` (modern alternative)**

`posix_spawn()` combines `fork()` + `exec()` efficiently and is sometimes preferred for embedded or performance-sensitive environments:

```c
#include <spawn.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>

extern char **environ;

int main() {
    pid_t pid;
    char *argv[] = {"python3", "script.py", NULL};

    if (posix_spawn(&pid, "/usr/bin/python3", NULL, NULL, argv, environ) != 0) {
        perror("posix_spawn failed");
        return 1;
    }

    int status;
    waitpid(pid, &status, 0);
    printf("Child exited with status %d\n", WEXITSTATUS(status));

    return 0;
}
```

---

### **Summary**

* Use `fork()` + `exec()` if you want full control over the child process.
* Use `system()` for quick-and-dirty execution.
* Use `posix_spawn()` if you want efficiency and simplicity, especially in modern Unix systems.

---

If you want, I can also show a version where your Python script **runs in the background** without blocking the C program, which is often useful for small calculations or daemon-like tasks.

Do you want that version?
