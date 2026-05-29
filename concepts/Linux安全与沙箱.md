# Linux 安全与沙箱

> 本文涵盖 Linux 下安全与沙箱相关的系统调用 API：seccomp、getrandom、landlock。

See also: [[Linux内核与系统调用]], [[Linux文件系统与路径操作]], [[C++POSIX文件操作]]

---

## 一、seccomp — 安全计算模式 `[Linux]`

**API说明：**
```c
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <sys/prctl.h>

// 通过prctl设置seccomp
int prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);        // [Linux 2.6.12+]
int prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, prog);  // [Linux 3.5+]

// 通过seccomp系统调用
int seccomp(unsigned int operation, unsigned int flags, void *args); // [Linux 3.17+]
// operation: SECCOMP_SET_MODE_STRICT | SECCOMP_SET_MODE_FILTER | SECCOMP_GET_ACTION_AVAIL
```

**示例：限制系统调用**
```c
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/audit.h>
#include <sys/syscall.h>
#include <stdio.h>
#include <unistd.h>

void seccomp_strict_example() {
    printf("Before seccomp: can use printf\n");

    // 启用严格模式（只允许 read, write, exit, sigreturn）
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT) == -1) {
        perror("prctl");
        return;
    }

    // write仍然可用
    const char msg[] = "After seccomp: write still works\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);

    // 调用其他系统调用会导致进程被SIGKILL杀死
    _exit(0);
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十五、安全与沙箱.md]

---

## 二、getrandom — 安全随机数 `[Linux 3.17+]`

**API说明：**
```c
#include <sys/random.h>

ssize_t getrandom(void *buf, size_t buflen, unsigned int flags); // [Linux 3.17+]
// flags: GRND_RANDOM (使用/dev/random) | GRND_NONBLOCK
// 注：glibc 2.25+ 提供包装函数
```

**示例：**
```c
#include <sys/random.h>
#include <stdio.h>
#include <stdint.h>

void getrandom_example() {
    uint64_t random_val;
    ssize_t n = getrandom(&random_val, sizeof(random_val), 0);
    if (n == sizeof(random_val)) {
        printf("Random uint64: %lu\n", random_val);
    }

    unsigned char buf[32];
    getrandom(buf, sizeof(buf), 0);
    printf("Random bytes: ");
    for (int i = 0; i < 32; i++) printf("%02x", buf[i]);
    printf("\n");
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十五、安全与沙箱.md]

---

## 三、landlock — 访问控制 `[Linux 5.13+]`

**API说明：**
```c
#include <linux/landlock.h>
#include <sys/syscall.h>

int landlock_create_ruleset(const struct landlock_ruleset_attr *attr,
                            size_t size, __u32 flags);
int landlock_add_rule(int ruleset_fd, enum landlock_rule_type rule_type,
                      const void *rule_attr, __u32 flags);
int landlock_restrict_self(int ruleset_fd, __u32 flags);
```

**示例：限制文件系统访问**
```c
#include <linux/landlock.h>
#include <sys/syscall.h>
#include <sys/prctl.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

void landlock_example() {
    struct landlock_ruleset_attr attr = {
        .handled_access_fs =
            LANDLOCK_ACCESS_FS_READ_FILE |
            LANDLOCK_ACCESS_FS_WRITE_FILE |
            LANDLOCK_ACCESS_FS_EXECUTE
    };

    int ruleset_fd = syscall(SYS_landlock_create_ruleset, &attr, sizeof(attr), 0);
    if (ruleset_fd < 0) { perror("landlock_create_ruleset"); return; }

    // 允许读取 /tmp
    int path_fd = open("/tmp", O_PATH | O_CLOEXEC);
    struct landlock_path_beneath_attr path_beneath = {
        .allowed_access = LANDLOCK_ACCESS_FS_READ_FILE | LANDLOCK_ACCESS_FS_WRITE_FILE,
        .parent_fd = path_fd
    };
    syscall(SYS_landlock_add_rule, ruleset_fd,
            LANDLOCK_RULE_PATH_BENEATH, &path_beneath, 0);
    close(path_fd);

    // 应用限制
    prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
    syscall(SYS_landlock_restrict_self, ruleset_fd, 0);
    close(ruleset_fd);

    printf("Landlock restrictions applied. Only /tmp is accessible.\n");
}
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十五、安全与沙箱.md]

## Related Pages
- [[Linux内核与系统调用]]
- [[Linux文件系统与路径操作]]
- [[C++POSIX文件操作]]
- [[IPC进程间通信]]
