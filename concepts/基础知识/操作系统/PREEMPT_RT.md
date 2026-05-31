# PREEMPT_RT

**PREEMPT_RT**（Real-Time Preemption Patch，实时抢占补丁）是针对主线 Linux 内核开发的一套增强补丁，它的目标是在不改变 Linux 主体架构的基础上，将原本只有有限范围抢占性的 Linux 内核，变为**几乎全内核可抢占**、高确定性、低延迟的实时操作系统内核。

**RT 补丁**就是PREEMPT_RT的通俗说法，即“Real-Time Patch”。它包含一系列技术改造，主要包括：

- 将大量自旋锁（spinlock）替换为可睡眠的实时互斥锁（rt_mutex），以大幅缩短内核中“不可抢占区”时长。
- 将绝大部分硬件中断（hardirq）和软中断（softirq）**线程化**（即以内核线程形式运行），从而能够用普通优先级、亲和性机制灵活调度和隔离中断处理。
- 提供更细粒度的内核抢占路径，相比 vanilla Linux（未打补丁版本）在负载下的最大调度延迟大幅降低。

**PREEMPT_RT 的核心作用：**
- 显著降低实时线程的“最坏情况调度延迟”——常见从毫秒级降至几十、几百微秒，甚至更低（依赖硬件和配置）。
- 更高的调度可控性，适合高频交易、工业自动化、机器人等软/硬实时领域。
- 即使不是极端实时要求，也能改善低延迟、高并发服务器应用中的突发抖动问题。

**应用方式：**
- 官方主线内核已有部分 PREEMPT_RT 补丁合并，最完整功能需选用专门的 RT 内核分支或发行版（如 Ubuntu Studio、RedHat/CentOS RT 变种等）。
- 选择内核编译选项 `CONFIG_PREEMPT_RT`、“Fully Preemptible Kernel (RT)”即可。

简而言之，**PREEMPT_RT 或 RT 补丁让 Linux “更像实时操作系统”，为 C++ 高性能与实时场景提供坚实基础。**

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-二、内核调度模型与-PREEMPT_RT.md]

## PREEMPT_RT 安装方法

### 1. 使用发行版自带 RT 内核（推荐）

主流 Linux 发行版（如 Ubuntu、Debian、CentOS、Red Hat 等）通常提供预编译好的 RT（实时）内核包，可以直接安装：

- **Ubuntu/Debian**（通过 apt 安装预编译 RT 内核）:
    ```bash
    sudo apt update
    sudo apt install linux-image-rt-amd64
    # 查询已装的 RT 内核
    dpkg -l | grep linux-image
    # 设置 grub 默认启动内核，或重启进入对应 RT 内核
    sudo reboot
    ```

- **Red Hat/CentOS/Rocky/Alma**（需启用 Real Time Repo）:
    ```bash
    sudo yum install kernel-rt kernel-rt-devel
    sudo grub2-set-default 0   # 或手动选择 RT kernel 为默认
    sudo reboot
    ```

**安装后重启，并通过 `uname -a` 或 `cat /proc/version` 验证内核是否为 RT 版本**。

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-二、内核调度模型与-PREEMPT_RT.md]

### 2. 手动编译 PREEMPT_RT 补丁内核

如官方源无你平台合适的 RT 内核，可手动 patch+build：

1. **下载内核源码和 PREEMPT_RT 补丁**

    - 到 https://mirrors.edge.kernel.org/pub/linux/kernel/ 下载对应版本的 Linux 内核源码。
    - 到 https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/ 下载对应版本的 RT 补丁（patch-X.YY.Z-rtXX.patch.xz）。

2. **打补丁**

    ```bash
    tar xf linux-X.YY.Z.tar.xz
    cd linux-X.YY.Z
    xz -d patch-X.YY.Z-rtXX.patch.xz
    # 这个命令的作用：将 RT 实时补丁应用到内核源码树
    patch -p1 < ../patch-X.YY.Z-rtXX.patch
    ```

3. **配置并编译 RT 内核**

    ```bash
    # 这是什么: 这是内核的图形化配置菜单命令，用于配置各项内核参数，包括选择"Fully Preemptible Kernel (RT)"等选项
    make menuconfig
    # 设置 Preemption Model → Fully Preemptible Kernel (RT)
    # 其它可选优化: 禁用不需要的驱动, 开启 NUMA/Cgroups 配置等

    # 编译内核和模块（自动根据 CPU 核数加速编译）
    make -j$(nproc)
    #  编译的是整个内核（包括内核镜像、内核模块等），最终会生成 vmlinuz、System.map、和内核模块等文件
    sudo make modules_install
    # 这里执行的是整个内核（kernel）和相关模块的安装，而不是单独安装某个“插件”。
    # 原因：实时补丁（PREEMPT_RT）属于内核源码级别的修改，必须完整编译和部署整个内核，而不是以插件形式动态加载。
    # 为什么不能只安装模块（sudo make modules_install），而必须执行 sudo make install？
    # 因为实时补丁（PREEMPT_RT）修改了内核的核心调度机制，只编译和安装模块无法让系统运行于新的实时内核。
    # 必须将新的内核镜像（vmlinuz-xxx-rt）完整安装到 /boot，并更新引导，否则系统依旧启动老内核，仅换模块没有效果。
    sudo make install
    ```

4. **更新 grub 并重启**

    ```bash
    sudo update-grub    # 或 grub2-mkconfig -o /boot/grub2/grub.cfg
    sudo reboot
    ```

5. **启动并确认 RT 内核**

    ```bash
    uname -a   # 应该包含 'PREEMPT RT'
    ```

上面的示例执行 `uname -a`，发现显示为：

```
Linux win-chen 6.6.87.2-microsoft-standard-WSL2 #1 SMP PREEMPT_DYNAMIC Thu Jun  5 18:30:46 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
```

分析：

- 该内核是 WSL2 虚拟环境中的 Linux，并非标准物理机环境。
- 抢占模型显示为 `PREEMPT_DYNAMIC`，说明此内核支持动态切换抢占模式，但**不是 PREEMPT_RT 实时内核**（RT 内核通常字段为 `PREEMPT_RT`）。
- 如果目的是验证是否成功切换到 RT 实时内核，应在上述输出中看到 `PREEMPT_RT` 字样。

因此，这个输出证明当前系统并未加载 RT 实时补丁的内核，而是一个常规（或仅有动态抢占能力）的内核环境。

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-二、内核调度模型与-PREEMPT_RT.md]

### 3. 验证

预期看到如下类似内容：

```bash
$ uname -a
Linux hostname 5.10.107-rt62 #1 SMP PREEMPT_RT ...
```
或
```bash
$ cat /proc/version
... PREEMPT_RT ...
```

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-二、内核调度模型与-PREEMPT_RT.md]

### 4. 常见问题和建议

- RT 内核版本需与补丁严格对应，否则打 patch 会报错。
- 生产环境建议优先用官方发行版支持的预构建包，兼容性更好。
- RT 内核下部分特性和硬件驱动兼容性与通用内核略有不同，升级/替换时需做好测试。
- 查看当前内核抢占级别：`zcat /proc/config.gz | grep PREEMPT`。

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-二、内核调度模型与-PREEMPT_RT.md]

## 2.1 抢占模型

文档中给出的 Linux 抢占模型从弱到强：

```text
PREEMPT_NONE       → 仅自愿抢占，延迟最大
PREEMPT_VOLUNTARY  → 在安全点自愿抢占
PREEMPT            → 多数代码可抢占（桌面/服务器常见）
PREEMPT_RT         → 几乎全可抢占 + 中断线程化（RT 补丁）
```

**调度优化要点：**

- 若要做软实时 / 低延迟系统，至少使用 `PREEMPT`，更推荐打 **PREEMPT_RT**。
- PREEMPT_RT 将大量 **spinlock → 可睡眠的 rt_mutex**，缩短不可抢占区。
- 将大多数硬中断处理线程化，使之受调度器管理，可以设定优先级和亲和性。

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-二、内核调度模型与-PREEMPT_RT.md]

## 2.2 PREEMPT_RT 的调度收益

- 最大调度延迟通常可从 **数十毫秒 → 数百微秒甚至更低**（视负载和硬件而定）。
- 对实时线程来说，内核路径被拆成更多可抢占小片段，**高优先级线程更容易及时抢占**。
- 中断被线程化后，可以：  
  - 统一使用 SCHED_FIFO/SCHED_RR 管理中断线程；  
  - 配合 `smp_affinity` 把中断集中在“非实时核”或专用“中断核”。

[src: raw/ingested/2技术/性能优化/调度优化-调度优化-总结-二、内核调度模型与-PREEMPT_RT.md]