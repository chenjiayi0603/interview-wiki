# 避免 CPU 频率缩放

**问题**：CPU 频率动态调整会导致性能不稳定。

## 解决方案

### 1. 设置 CPU 为性能模式（Linux）

```bash
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### 2. 禁用 CPU 节能特性

```bash
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo  # Intel
echo 0 | sudo tee /proc/sys/kernel/sched_rt_runtime_us  # 实时调度
```

### 3. 使用 cpupower 工具

```bash
sudo cpupower frequency-set -g performance
sudo cpupower set -b 0  # 禁用 C-states
```

[src: raw/ingested/2技术/性能优化/低延迟-低延迟c++系统分析-二、CPU-优化技术-二、CPU-优化技术.md]