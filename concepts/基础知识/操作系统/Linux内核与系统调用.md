# Linux 内核与系统调用

> 本文涵盖 Linux 下内核与系统相关的系统调用 API：系统信息、内核模块、系统控制、挂载与文件系统、新挂载 API、句柄与路径、BPF、性能分析、NUMA 与内存策略及其他。

See also: [[Linux文件系统与路径操作]], [[C++POSIX文件操作]], [[Linux系统调用-文件IO]], [[POSIX进程控制]]

---

## 一、系统信息

```c
#include <sys/utsname.h>
#include <sys/sysinfo.h>

int uname(struct utsname *buf);
int sysinfo(struct sysinfo *info);
int syslog(int type, char *bufp, int len);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 二、内核模块

```c
#include <linux/module.h>

int init_module(void *module_image, unsigned long len, const char *param_values);
int finit_module(int fd, const char *param_values, int flags);  // [3.8]
int delete_module(const char *name, int flags);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 三、系统控制

```c
#include <unistd.h>
#include <sys/reboot.h>

int reboot(int cmd);
int sethostname(const char *name, size_t len);
int setdomainname(const char *name, size_t len);
int chroot(const char *path);
int acct(const char *filename);
int sync(void);
int pause(void);
int vhangup(void);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 四、挂载与文件系统

```c
#include <sys/mount.h>

int mount(const char *source, const char *target, const char *filesystemtype,
          unsigned long mountflags, const void *data);
int umount2(const char *target, int flags);
int swapon(const char *path, int swapflags);
int swapoff(const char *path);
int quotactl(int cmd, const char *special, int id, void *addr);
int quotactl_fd(int fd, int cmd, int id, void *addr);  // [5.14]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 五、新挂载 API `[Linux 5.2+]`

```c
#include <fcntl.h>

int open_tree(int dfd, const char *path, unsigned flags);
int move_mount(int from_dfd, const char *from_path, int to_dfd,
               const char *to_path, unsigned int flags);
int fsopen(const char *fs_name, unsigned int flags);
int fsconfig(int fd, unsigned int cmd, const char *key, const void *value, int aux);
int fsmount(int fs_fd, unsigned int flags, unsigned int ms_flags);
int fspick(int dfd, const char *path, unsigned flags);
int mount_setattr(int dfd, const char *path, unsigned int flags,
                  struct mount_attr *attr, size_t size);  // [5.12]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 六、句柄与路径

```c
#include <fcntl.h>

int name_to_handle_at(int dirfd, const char *pathname, struct file_handle *handle,
                      int *mount_id, int flags);  // [2.6.39]
int open_by_handle_at(int mount_fd, struct file_handle *handle, int flags);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 七、BPF `[Linux 3.18+]`

```c
#include <linux/bpf.h>

long bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 八、性能分析

```c
#include <linux/perf_event.h>

long perf_event_open(struct perf_event_attr *attr, pid_t pid, int cpu,
                     int group_fd, unsigned long flags);  // [2.6.31]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 九、NUMA 与内存策略 `[Linux 2.6.6+]`

```c
#include <numaif.h>

int mbind(void *addr, unsigned long len, int mode, const unsigned long *nodemask,
          unsigned long maxnode, unsigned flags);
int set_mempolicy(int mode, const unsigned long *nodemask, unsigned long maxnode);
int get_mempolicy(int *mode, unsigned long *nodemask, unsigned long maxnode,
                  unsigned long addr, unsigned long flags);
long migrate_pages(int pid, unsigned long maxnode, const unsigned long *old_nodes,
                   const unsigned long *new_nodes);
long move_pages(int pid, unsigned long nr_pages, const void **pages,
                const int *nodes, int *status, int flags);  // [2.6.18]
int set_mempolicy_home_node(unsigned long start, unsigned long len,
                            unsigned long home_node, unsigned long flags);  // [5.17]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

---

## 十、其他

```c
int personality(unsigned long persona);
int ioperm(unsigned long from, unsigned long num, int turn_on);
int iopl(int level);
int modify_ldt(int func, void *ptr, unsigned long bytecount);
int lookup_dcookie(uint64_t cookie, char *buf, size_t len);  // [2.6]
int cachestat(int fd, struct cachestat_range *cstat_range,
              struct cachestat *cstat, unsigned int flags);  // [6.5]
int kexec_load(unsigned long entry, unsigned long nr_segments,
               struct kexec_segment *segments, unsigned long flags);  // [2.6.13]
int kexec_file_load(int kernel_fd, int initrd_fd, unsigned long cmdline_len,
                    const char *cmdline, unsigned long flags);  // [3.17]
```

[src: raw/ingested/2技术/cpp/Linux c系统调用-十九、内核与系统.md]

## Related Pages
- [[Linux文件系统与路径操作]]
- [[C++POSIX文件操作]]
- [[Linux系统调用-文件IO]]
- [[POSIX进程控制]]
- [[内存管理]]
- [[进程内存区域与资源限制]]
