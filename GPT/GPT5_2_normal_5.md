I want to create a new process that is running a python script doing some minor calculation tasks. How would I do that in a Unix (like) system using C?



You want to launch a Python script as a separate process from a C program on a Unix-like system. In C on Unix, the typical approach is to use **`fork()`** and **`exec()`**, or a higher-level wrapper like **`system()`**. Here's a detailed explanation with examples.

---

### **1. Using `fork()` and `exec`**

This is the standard and flexible way to create a new process:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();  // Create a new process

    if (pid < 0) {
        // fork failed
        perror("fork failed");
        return 1;
    }

    if (pid == 0) {
        // Child process
        char *python = "/usr/bin/python3";   // Path to Python interpreter
        char *script = "/path/to/script.py"; // Path to your Python script
        char *args[] = {python, script, NULL};

        execv(python, args);  // Replace child process with Python
        // execv only returns if there's an error
        perror("execv failed");
        exit(1);
    } else {
        // Parent process
        printf("Child process PID: %d\n", pid);
        int status;
        waitpid(pid, &status, 0);  // Wait for child to finish
        printf("Child exited with status %d\n", WEXITSTATUS(status));
    }

    return 0;
}
```

**Explanation:**

* `fork()` creates a copy of the current process.
* `execv()` replaces the child process with the Python interpreter running your script.
* `waitpid()` lets the parent wait for the child to finish and collect its exit status.

---

### **2. Using `system()` (simpler, less flexible)**

If you don’t need control over the child process, `system()` is easier:

```c
#include <stdlib.h>
#include <stdio.h>

int main() {
    int ret = system("python3 /path/to/script.py");
    if (ret == -1) {
        perror("system failed");
    } else {
        printf("Script finished with return code %d\n", ret);
    }
    return 0;
}
```

**Pros:**

* Very simple.
* Runs a shell command.

**Cons:**

* Starts a shell process (slightly less efficient).
* Harder to capture output or control the child process.

---

### **3. Using `popen()` to capture output**

If your Python script produces output you want to read in C:

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    FILE *fp = popen("python3 /path/to/script.py", "r");
    if (fp == NULL) {
        perror("popen failed");
        return 1;
    }

    char buffer[128];
    while (fgets(buffer, sizeof(buffer), fp) != NULL) {
        printf("Python output: %s", buffer);
    }

    int status = pclose(fp);
    printf("Script exited with status %d\n", WEXITSTATUS(status));
    return 0;
}
```

---

### **Recommendation**

* Use **`fork()` + `exec`** if you want full control (e.g., different environment variables, I/O redirection, etc.).
* Use **`system()`** for simple scripts.
* Use **`popen()`** if you need to capture output.

---

If you want, I can give you a **ready-to-use C function `run_python_script(const char* script_path)`** that runs a Python script asynchronously and returns immediately, without blocking the parent process. That’s often what’s needed for minor calculation tasks.

Do you want me to do that?
