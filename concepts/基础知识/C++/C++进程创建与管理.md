# C++ 进程创建与管理

> 本文涵盖 C++ 中进程创建与管理的核心方式：`fork()+exec*()`、`std::system()`、`popen()/pclose()` 以及 `wait()/waitpid()`。

See also: [[POSIX进程控制]], [[POSIX进程派生]], [[IPC进程间通信]], [[C++多线程与并发]]

---

## 一、fork() + exec*()（POSIX）

```cpp
#include <iostream>
#include <unistd.h>
#include <sys/wait.h>
#include <cstring>

int main() {
    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }
    if (pid == 0) {
        execl("/bin/echo", "echo", "hello from child", nullptr);
        perror("execl");
        return 1;
    }
    std::cout << "parent: child pid = " << pid << std::endl;
    wait(nullptr);
    std::cout << "parent: child exited" << std::endl;
    return 0;
}
```

- `fork()`：复制当前进程，子进程从 `fork()` 返回处继续；父进程得到子进程 PID，子进程得到 0
- `execl(path, arg0, arg1, ..., nullptr)`：用新程序替换当前进程映像
- `exec` 族还有 `execv`、`execvp`、`execle` 等

[src: raw/ingested/2技术/cpp/C++多进程完整手册-二、进程创建与管理.md]

---

## 二、std::system()（C++ 标准）

```cpp
#include <cstdlib>
#include <iostream>

int main() {
    int ret = std::system("echo hello from system");
    std::cout << "exit status = " << WEXITSTATUS(ret) << std::endl;
    return 0;
}
```

- `std::system(command)` 通过 shell 执行命令，阻塞直到结束
- 返回值可用 `WEXITSTATUS` 等宏解析

[src: raw/ingested/2技术/cpp/C++多进程完整手册-二、进程创建与管理.md]

---

## 三、popen() / pclose()（进程 I/O）

```cpp
std::FILE* fp = popen("ls -l", "r");
// 读子进程 stdout
char buf[256];
while (std::fgets(buf, sizeof(buf), fp))
    std::cout << buf;
int status = pclose(fp);
```

- `popen("cmd", "r")`：创建子进程，返回可读流
- `popen("cmd", "w")`：可写流作为子进程 stdin

[src: raw/ingested/2技术/cpp/C++多进程完整手册-二、进程创建与管理.md]

---

## 四、wait() / waitpid()（进程等待）

```cpp
int status = 0;
pid_t w = waitpid(pid, &status, 0);
if (WIFEXITED(status))
    std::cout << "exited, code = " << WEXITSTATUS(status) << std::endl;
if (WIFSIGNALED(status))
    std::cout << "killed by signal " << WTERMSIG(status) << std::endl;
```

- `wait(nullptr)`：等待任意子进程
- `waitpid(pid, &status, WNOHANG)`：非阻塞轮询

[src: raw/ingested/2技术/cpp/C++多进程完整手册-二、进程创建与管理.md]

## Related Pages
- [[POSIX进程控制]]
- [[POSIX进程派生]]
- [[IPC进程间通信]]
- [[C++多线程与并发]]
