# Linux perf 到火焰图：全流程与细节

## 一、整体流程概览

从性能采样到最终可视化，完整链路如下：

```
perf record  →  perf.data  →  perf script  →  折叠栈  →  flamegraph.pl  →  SVG
  (采样)        (二进制)       (文本化)       (stackcollapse)   (生成图)     (交互式)
```

每一步都有大量细节，下面逐一拆解。

---

## 二、perf 是什么

`perf` 是 Linux 内核自带的性能分析工具，源码位于内核树 `tools/perf/`。它利用内核的 **perf_events** 子系统，通过硬件 PMU（Performance Monitoring Unit）和软件计数器来采集性能数据。

### 2.1 架构总览

```
用户态                          内核态                        硬件
┌──────────┐    syscall     ┌──────────────────┐      ┌──────────┐
│ perf 命令 │ ──────────── → │  perf_events 子系统 │ ← ── │ PMU 硬件  │
│          │  perf_event_  │                  │      │ (计数器)  │
│          │  open()       │  ring buffer      │      └──────────┘
│          │ ← ────────── │  (per-CPU)        │
│  mmap    │  读取采样数据  │                  │
│ perf.data│               └──────────────────┘
└──────────┘
```

### 2.2 安装

```bash
# Debian/Ubuntu
sudo apt install linux-tools-common linux-tools-$(uname -r)

# RHEL/CentOS
sudo yum install perf

# 验证
perf version
```

!!! warning "内核版本必须匹配"
    `perf` 工具版本必须与当前运行的内核版本一致。如果内核升级后没有更新 `linux-tools` 包，`perf` 会报错。

---

## 三、perf record — 采样阶段

### 3.1 基本用法

```bash
# 对指定命令采样
perf record -g ./myapp

# 对运行中的进程采样 30 秒
perf record -g -p <pid> -- sleep 30

# 全系统采样
perf record -g -a -- sleep 10
```

`-g` 是关键参数，表示 **记录调用栈**（call graph），没有它就无法生成火焰图。

### 3.2 采样原理

`perf record` 默认使用 **CPU cycles** 事件进行采样。核心机制：

```
1. perf_event_open() 系统调用
   → 内核为目标进程/CPU 创建一个 perf_event
   → 配置 PMU 硬件计数器，设定 overflow 阈值（采样周期）

2. CPU 执行指令时，硬件计数器递增
   → 计数器溢出时触发 PMI（Performance Monitoring Interrupt）
   → 内核 NMI handler 被调用

3. NMI handler 中：
   → 记录当前 RIP（指令指针）
   → 回溯调用栈（根据 -g 的方式：fp / dwarf / lbr）
   → 将采样数据写入 per-CPU 的 ring buffer

4. perf 用户态进程通过 mmap 读取 ring buffer
   → 写入 perf.data 文件
```

### 3.3 采样频率 vs 采样周期

```bash
# 按频率采样（默认 4000 Hz，即每秒 ~4000 个样本）
perf record -F 99 -g ./myapp      # 99 Hz，常用值（避开 100 Hz 时钟对齐）

# 按事件次数采样（每 N 次事件采一次）
perf record -c 100000 -g ./myapp  # 每 10 万个 cycles 采一次
```

!!! tip "为什么推荐 99 Hz 而不是 100 Hz"
    如果采样频率恰好是 timer 中断频率的倍数（如 100 Hz 对 100 Hz HZ），每次采样都会落在相同的代码路径上（timer 处理），造成 **lockstep 偏差**。使用 99 Hz 这样的奇数可以避免。

### 3.4 调用栈回溯方式（`-g` 的模式）

这是影响火焰图质量最关键的选项：

| 方式 | 参数 | 原理 | 优点 | 缺点 |
|------|------|------|------|------|
| Frame Pointer | `--call-graph fp` | 沿 RBP 链回溯 | 开销最低 | 需 `-fno-omit-frame-pointer` 编译；现代编译器默认省略 FP |
| DWARF | `--call-graph dwarf` | 记录栈顶 N 字节，事后用 DWARF 信息回溯 | 不需要 FP；最准确 | 采样数据量大（默认拷贝 8192 字节栈） |
| LBR | `--call-graph lbr` | 利用 CPU 的 Last Branch Record 硬件 | 开销极低；不需要 FP | 栈深度有限（通常 8-32 层）；需要 Intel Haswell+ |

```bash
# Frame Pointer 模式（需要编译时保留 FP）
perf record --call-graph fp -p <pid> -- sleep 30

# DWARF 模式（推荐通用方案）
perf record --call-graph dwarf -p <pid> -- sleep 30

# DWARF 模式 + 指定栈拷贝大小
perf record --call-graph dwarf,16384 -p <pid> -- sleep 30

# LBR 模式（Intel CPU 推荐）
perf record --call-graph lbr -p <pid> -- sleep 30
```

!!! note "Frame Pointer 是最常见的坑"
    如果火焰图中出现大量 `[unknown]` 或调用栈只有 1-2 层，几乎一定是因为二进制没有 frame pointer。解决方案：

    - 用 `-fno-omit-frame-pointer` 重新编译
    - 改用 `--call-graph dwarf`
    - 改用 `--call-graph lbr`（Intel CPU）

### 3.5 常用事件类型

```bash
# 查看所有支持的事件
perf list

# CPU cycles（默认）— 找 CPU 热点
perf record -e cycles -g ./myapp

# 指令数
perf record -e instructions -g ./myapp

# cache miss — 找内存瓶颈
perf record -e cache-misses -g ./myapp

# page fault — 找内存分配热点
perf record -e page-faults -g ./myapp

# 软件事件 — 上下文切换
perf record -e context-switches -g -a -- sleep 10

# 多事件同时采样
perf record -e cycles,cache-misses -g ./myapp
```

### 3.6 perf.data 文件格式

`perf record` 的输出是二进制文件 `perf.data`，内部结构：

```
┌─────────────────────┐
│      File Header    │  magic, 版本, 数据偏移, 属性偏移
├─────────────────────┤
│   Attribute Section │  采样配置（事件类型、频率、调用栈模式等）
├─────────────────────┤
│    Data Section     │  采样记录流（按时间排序）
│  ┌────────────────┐ │
│  │ PERF_RECORD_    │ │
│  │ SAMPLE          │ │  → IP, TID, 时间戳, 调用栈, CPU, 权重...
│  ├────────────────┤ │
│  │ PERF_RECORD_    │ │
│  │ MMAP2           │ │  → 进程内存映射（用于地址→符号解析）
│  ├────────────────┤ │
│  │ PERF_RECORD_    │ │
│  │ COMM            │ │  → 进程/线程命名
│  ├────────────────┤ │
│  │ ...             │ │
│  └────────────────┘ │
├─────────────────────┤
│  Feature Section    │  构建 ID、主机名、CPU 拓扑、cmdline 等元数据
└─────────────────────┘
```

关键记录类型：

| 记录类型 | 作用 |
|----------|------|
| `PERF_RECORD_SAMPLE` | 核心采样数据：IP、调用栈、时间戳 |
| `PERF_RECORD_MMAP2` | 进程加载的 ELF 文件映射，用于符号解析 |
| `PERF_RECORD_COMM` | 进程/线程名变更 |
| `PERF_RECORD_FORK` / `EXIT` | 进程创建/退出 |
| `PERF_RECORD_SWITCH` | 上下文切换（如果启用） |

---

## 四、perf script — 文本化采样数据

### 4.1 基本用法

```bash
# 将 perf.data 转为人类可读的文本
perf script > out.perf
```

输出格式示例：

```
myapp  12345 [003] 12345.678901:     250000 cycles:
        aaaaaaaa1234 compute_hash+0x14 (/usr/bin/myapp)
        aaaaaaaa5678 process_request+0x88 (/usr/bin/myapp)
        aaaaaaaa9abc handle_connection+0x120 (/usr/bin/myapp)
        aaaaaaaade00 main+0x200 (/usr/bin/myapp)
        7f1234567890 __libc_start_main+0xf0 (/usr/lib/libc.so.6)
```

每个采样块包含：

```
进程名  TID [CPU] 时间戳:  事件计数 事件名:
        地址 符号+偏移 (DSO 路径)      ← 调用栈（从栈顶到栈底）
        地址 符号+偏移 (DSO 路径)
        ...
```

### 4.2 符号解析

`perf script` 需要将采样中的原始地址解析为函数名。解析依赖：

```
地址 → 查 MMAP2 记录确定所属 DSO → 查 DSO 的符号表/DWARF → 函数名
```

**符号来源优先级：**

1. 二进制文件自身的 `.symtab` / `.dynsym` 段
2. 独立的 debuginfo 文件（`/usr/lib/debug/...`）
3. Build ID 对应的 debuginfo（`~/.debug/` 或 debuginfod）
4. `/proc/<pid>/maps`（仅用于确定 DSO，不含符号）
5. 内核符号：`/proc/kallsyms`

```bash
# 安装 debuginfo（用于解析系统库）
# Debian/Ubuntu
sudo apt install libc6-dbg

# RHEL/CentOS
sudo debuginfo-install glibc

# 使用 debuginfod（自动下载 debuginfo，推荐）
export DEBUGINFOD_URLS="https://debuginfod.elfutils.org/"
```

!!! warning "符号丢失的常见原因"
    - **二进制被 `strip` 过**：没有 `.symtab`，需要安装对应的 `-dbg` / `-debuginfo` 包
    - **JIT 编译的代码**（Java、Node.js、Python）：地址不在任何 DSO 中，需要特殊处理（见第八节）
    - **内核符号缺失**：确保 `/proc/kallsyms` 可读（`kernel.kptr_restrict=0`）
    - **采样时和分析时的二进制不同**：Build ID 不匹配

### 4.3 有用的过滤选项

```bash
# 只输出指定进程名
perf script --comm myapp

# 只输出指定 TID
perf script --tid 12345

# 只输出指定时间范围
perf script --time 10%/5     # 第 5 个 10% 时间段

# 自定义输出字段
perf script -F comm,tid,time,event,ip,sym,dso,trace
```

---

## 五、栈折叠 — stackcollapse

### 5.1 为什么需要折叠

`perf script` 的输出是**每个采样一块多行文本**，而火焰图工具需要的输入格式是**每行一个唯一调用栈 + 出现次数**。`stackcollapse-perf.pl` 完成这个转换。

### 5.2 折叠过程

输入（`perf script` 输出）：

```
myapp 12345 cycles:
    compute_hash
    process_request
    main

myapp 12345 cycles:
    compute_hash
    process_request
    main

myapp 12345 cycles:
    read_config
    main
```

经过 `stackcollapse-perf.pl`：

```
myapp;main;process_request;compute_hash 2
myapp;main;read_config 1
```

**转换规则：**

1. 将调用栈从底到顶翻转（栈底在左，栈顶在右），用 `;` 连接
2. 相同的完整调用栈合并，计数累加
3. 进程名作为栈的最左端（root）

### 5.3 使用方法

```bash
# 基本用法
perf script | stackcollapse-perf.pl > out.folded

# 包含进程名（默认包含）
perf script | stackcollapse-perf.pl --all > out.folded

# 只包含内核栈
perf script | stackcollapse-perf.pl --kernel > out.folded

# 只包含用户态栈
perf script | stackcollapse-perf.pl --jit > out.folded

# 内联函数展开（如果有 DWARF 信息）
perf script | stackcollapse-perf.pl --inline > out.folded
```

!!! tip "也可以用 perf 内置功能跳过这一步"
    ```bash
    # perf 较新版本支持直接输出折叠格式
    perf script -s stackcollapse.py
    # 或使用 --flamegraph（部分版本）
    ```

---

## 六、flamegraph.pl — 生成火焰图

### 6.1 获取工具

```bash
git clone https://github.com/brendangregg/FlameGraph.git
cd FlameGraph
# 主要脚本：
#   stackcollapse-perf.pl  — 折叠 perf 输出
#   flamegraph.pl          — 生成 SVG
```

### 6.2 完整命令链

```bash
# 标准流程（三步管道）
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > flamegraph.svg

# 一步到位（从 perf.data 到 SVG）
perf script -i perf.data | ./stackcollapse-perf.pl | ./flamegraph.pl > flamegraph.svg
```

### 6.3 flamegraph.pl 的工作原理

```
读取折叠栈文本
│
├── 1. 解析每行：按 `;` 分割栈帧，获取计数
│
├── 2. 构建栈帧树（trie）
│     root
│     ├── main (3)
│     │   ├── process_request (2)
│     │   │   └── compute_hash (2)
│     │   └── read_config (1)
│
├── 3. 计算每个栈帧的宽度
│     宽度 = 该帧（含所有子帧）的采样总数 / 全局采样总数 × 图宽
│
├── 4. 深度优先遍历，为每个栈帧生成 SVG 矩形
│     x 坐标：同一层级的帧按字母序排列（不是按时间！）
│     y 坐标：栈深度（从下到上）
│     宽度：按采样比例
│     颜色：根据函数名 hash 或类型分配
│
└── 5. 输出 SVG（内嵌 JavaScript 实现交互）
```

### 6.4 常用参数

```bash
# 设置标题
flamegraph.pl --title "MyApp CPU Profile" > out.svg

# 设置颜色方案
flamegraph.pl --color java    # Java 配色（绿=Java，黄=C/C++，红=内核）
flamegraph.pl --color mem     # 内存配色
flamegraph.pl --color io      # I/O 配色

# 反转火焰图（icicle graph，从上往下看）
flamegraph.pl --inverted > out.svg

# 设置最小显示宽度（过滤太窄的帧）
flamegraph.pl --minwidth 0.5 > out.svg

# 设置图片宽度
flamegraph.pl --width 1600 > out.svg

# 计数类型标签（显示在详情中）
flamegraph.pl --countname "samples" > out.svg

# 反转栈顺序（适合查看 "谁调用了这个函数"）
flamegraph.pl --reverse > out.svg
```

### 6.5 如何阅读火焰图

```
                    ┌──────┐
                    │ func_d│  ← 采样最多命中的叶子函数（CPU 热点）
              ┌─────┴──────┴─────┐
              │     func_b       │
        ┌─────┴──────────────────┴───────┐
        │           func_a               │
   ┌────┴────────────────────────────────┴────┐
   │                main                      │
   └──────────────────────────────────────────┘
```

**核心规则：**

| 维度 | 含义 |
|------|------|
| **Y 轴**（高度） | 调用栈深度。底部是栈底（`main`），顶部是栈顶（实际执行的函数） |
| **X 轴**（宽度） | 采样数量的比例。**不是时间线**，同层帧按字母序排列 |
| **帧宽度** | 该函数（含其调用的所有子函数）占 CPU 时间的比例 |
| **帧颜色** | 无特殊含义（随机或按类型），用于视觉区分 |

**关注点：**

- **顶部的宽帧**（"plateaus"）= 直接消耗 CPU 最多的函数
- **底部的宽帧** = 调用路径入口，本身不一定慢，但其调用的子树占比大
- **窄帧** = 不值得优化（占比太小）

**交互功能（SVG 内嵌）：**

- **点击** 某帧 → 放大该子树（zoom）
- **鼠标悬停** → 显示函数名、采样数和百分比
- **Ctrl+F / 搜索** → 高亮匹配的函数，显示累计占比

---

## 七、完整实战示例

### 7.1 一次完整的 CPU profiling

```bash
# 1. 采样（99 Hz，DWARF 回溯，30 秒）
sudo perf record -F 99 --call-graph dwarf -p $(pgrep myapp) -- sleep 30

# 2. 查看采样统计
perf report --stdio | head -40

# 3. 生成火焰图
perf script | stackcollapse-perf.pl | flamegraph.pl \
  --title "myapp CPU flame graph" \
  --subtitle "30s sample at 99Hz" \
  > cpu_flamegraph.svg

# 4. 在浏览器中打开
xdg-open cpu_flamegraph.svg   # Linux
# open cpu_flamegraph.svg     # macOS
```

### 7.2 Off-CPU 分析（等待时间分析）

常规火焰图只能看到 **on-CPU** 的热点。如果程序慢是因为等 I/O、锁、sleep，需要 off-CPU 分析：

```bash
# 记录调度事件
sudo perf record -e sched:sched_switch \
  --call-graph dwarf -p $(pgrep myapp) -- sleep 30

# 生成 off-CPU 火焰图（需要特殊的 stackcollapse 脚本处理时间差）
perf script | stackcollapse-perf.pl | flamegraph.pl \
  --title "Off-CPU flame graph" \
  --color io \
  --countname "us" \
  > offcpu_flamegraph.svg
```

!!! note "更完善的 Off-CPU 方案"
    Brendan Gregg 的 [bcc/BPF 工具集](https://github.com/iovisor/bcc) 中的 `offcputime` 提供了更精确的 off-CPU 分析，它直接在内核中计算阻塞时间，不会像 `sched_switch` 事件那样产生大量数据。

### 7.3 差分火焰图（对比两次采样）

```bash
# 优化前采样
perf record -F 99 --call-graph dwarf -o before.data -p <pid> -- sleep 30
perf script -i before.data | stackcollapse-perf.pl > before.folded

# 优化后采样
perf record -F 99 --call-graph dwarf -o after.data -p <pid> -- sleep 30
perf script -i after.data | stackcollapse-perf.pl > after.folded

# 生成差分火焰图（红=增加，蓝=减少）
difffolded.pl before.folded after.folded | flamegraph.pl \
  --title "Differential flame graph" \
  --negate \
  > diff_flamegraph.svg
```

---

## 八、特殊场景处理

### 8.1 Java 应用

Java 代码由 JIT 编译，地址不在任何 ELF 文件中，perf 默认无法解析符号。

```bash
# 方法 1：使用 perf-map-agent（JDK 8-11）
# 让 JVM 生成 /tmp/perf-<pid>.map 符号映射文件
java -XX:+PreserveFramePointers -jar myapp.jar &
# 采样
perf record -F 99 --call-graph fp -p $(pgrep java) -- sleep 30
# 生成 map 文件
create-java-perf-map.sh $(pgrep java)
# 生成火焰图
perf script | stackcollapse-perf.pl | flamegraph.pl --color java > java_flame.svg

# 方法 2：async-profiler（推荐，JDK 8+，无需 FP）
./profiler.sh -d 30 -f flamegraph.html <pid>
```

!!! tip "async-profiler"
    [async-profiler](https://github.com/async-profiler/async-profiler) 是 Java 火焰图的最佳工具。它同时采样 Java 栈和 native 栈，不需要 `-XX:+PreserveFramePointers`，也不需要 `perf`。直接输出火焰图 HTML。

### 8.2 Node.js

```bash
# 启动时启用 perf 支持
node --perf-basic-prof myapp.js &

# 这会生成 /tmp/perf-<pid>.map
# 然后正常用 perf record + 火焰图流程

# 或使用 0x（更简单）
npx 0x myapp.js
```

### 8.3 Python

```bash
# 使用 py-spy（推荐，无需修改代码）
pip install py-spy
py-spy record -o profile.svg --pid <pid>

# 使用 perf + Python 符号支持
python3 -X perf myapp.py &   # Python 3.12+ 内置支持
perf record -F 99 --call-graph dwarf -p <pid> -- sleep 30
```

### 8.4 内核态分析

```bash
# 采样内核态（需要 root）
sudo perf record -F 99 -g -a -- sleep 10

# 确保内核符号可读
sudo sysctl kernel.kptr_restrict=0
sudo sysctl kernel.perf_event_paranoid=-1

# 生成火焰图（区分内核/用户态颜色）
sudo perf script | stackcollapse-perf.pl | flamegraph.pl \
  --color kernel \
  --title "Kernel CPU flame graph" \
  > kernel_flame.svg
```

### 8.5 容器内采样

```bash
# 方案 1：从宿主机采样容器进程
# 找到容器内进程在宿主机的 PID
PID=$(docker inspect --format '{{.State.Pid}}' <container>)
sudo perf record -F 99 --call-graph dwarf -p $PID -- sleep 30

# 方案 2：容器内运行 perf（需要特权）
docker run --privileged --pid=host ...
# 或
docker run --cap-add SYS_ADMIN --cap-add SYS_PTRACE ...
```

!!! warning "容器内符号解析"
    在宿主机采样时，perf 需要访问容器内的二进制文件来解析符号。可以用 `--symfs` 指定根文件系统路径：
    ```bash
    perf script --symfs /proc/<pid>/root/
    ```

---

## 九、perf_event_open 系统调用详解

这是 perf 的核心系统调用，理解它有助于理解整个采样机制。

```c
int perf_event_open(
    struct perf_event_attr *attr,  // 事件配置
    pid_t pid,                      // 目标进程（-1 = 所有进程）
    int cpu,                        // 目标 CPU（-1 = 所有 CPU）
    int group_fd,                   // 事件组（-1 = 独立）
    unsigned long flags             // PERF_FLAG_FD_CLOEXEC 等
);
```

`perf_event_attr` 中的关键字段：

| 字段 | 含义 |
|------|------|
| `type` | 事件类型：`PERF_TYPE_HARDWARE`、`PERF_TYPE_SOFTWARE`、`PERF_TYPE_TRACEPOINT` |
| `config` | 具体事件：`PERF_COUNT_HW_CPU_CYCLES`、`PERF_COUNT_HW_CACHE_MISSES` |
| `sample_period` / `sample_freq` | 采样周期或频率 |
| `sample_type` | 每个样本包含什么：`PERF_SAMPLE_IP | PERF_SAMPLE_CALLCHAIN | PERF_SAMPLE_TID | PERF_SAMPLE_TIME` |
| `read_format` | 读取计数器时的格式 |
| `freq` | 1 = 使用频率模式，0 = 使用周期模式 |

Ring buffer 交互：

```
perf_event_open() 返回 fd
    → mmap(fd, ...) 映射 ring buffer 到用户态
    → 内核写入采样数据到 ring buffer
    → 用户态 perf 进程通过 poll(fd) 等待通知
    → 读取 ring buffer 中的 perf_event_header + 数据
    → 写入 perf.data
```

---

## 十、权限与安全

### 10.1 `perf_event_paranoid`

```bash
cat /proc/sys/kernel/perf_event_paranoid
```

| 值 | 权限 |
|----|------|
| `-1` | 不限制（允许所有用户访问所有事件） |
| `0` | 允许非 root 访问非内核态事件 |
| `1` | 允许非 root 进行受限的用户态采样（默认） |
| `2` | 只允许非 root 使用用户态软件事件 |
| `3+` | 非 root 完全禁止（部分发行版，如 Debian） |

```bash
# 临时放宽（用于调试）
sudo sysctl kernel.perf_event_paranoid=-1

# 永久设置
echo 'kernel.perf_event_paranoid=-1' | sudo tee /etc/sysctl.d/99-perf.conf
sudo sysctl --system
```

### 10.2 CAP_PERFMON

Linux 5.8+ 引入了 `CAP_PERFMON` capability，可以更细粒度地授权：

```bash
# 给 perf 二进制添加 capability（比 setuid root 安全）
sudo setcap cap_perfmon,cap_sys_ptrace+ep $(which perf)
```

---

## 十一、常见问题排查

### 问题 1：火焰图中大量 `[unknown]`

```bash
# 检查是否有符号
nm /path/to/binary | head
readelf -s /path/to/binary | head

# 如果被 strip 了，安装 debuginfo 包
# 或重新编译带 -g

# 如果是共享库缺符号
perf script 2>&1 | grep "no symbols"
# 安装对应的 -dbg 包
```

### 问题 2：调用栈只有 1-2 层

```bash
# 几乎一定是 frame pointer 缺失
# 方案 1：重新编译
CFLAGS="-fno-omit-frame-pointer" make

# 方案 2：改用 DWARF
perf record --call-graph dwarf ...

# 方案 3：改用 LBR（Intel）
perf record --call-graph lbr ...
```

### 问题 3：`perf record` 报权限错误

```bash
# 检查 paranoid 级别
cat /proc/sys/kernel/perf_event_paranoid

# 快速解决：用 sudo
sudo perf record ...

# 长期解决：降低 paranoid 或授权 CAP_PERFMON
```

### 问题 4：perf.data 文件过大

```bash
# 降低采样频率
perf record -F 49 ...    # 49 Hz 足够多数场景

# 缩短采样时间
perf record ... -- sleep 10

# DWARF 模式减小栈拷贝
perf record --call-graph dwarf,4096 ...    # 默认 8192

# 采样后查看大小
ls -lh perf.data
```

### 问题 5：火焰图是空的或只有一条栈

```bash
# 检查采样数
perf report --stdio | head -5
# 如果 "Samples: 0" → 目标进程在采样期间可能没有 CPU 活动

# 检查事件是否正确
perf stat -e cycles -p <pid> -- sleep 5
# 如果 cycles = 0，进程确实没有在跑
```

---

## 十二、进阶工具与替代方案

| 工具 | 特点 |
|------|------|
| [bpftrace](https://github.com/bpftrace/bpftrace) | 用 BPF 做自定义 profiling，灵活强大 |
| [async-profiler](https://github.com/async-profiler/async-profiler) | Java 最佳 profiling 工具 |
| [py-spy](https://github.com/benfred/py-spy) | Python 零侵入 profiler |
| [pprof](https://github.com/google/pprof) | Go 内置 profiler，支持火焰图 |
| [speedscope](https://www.speedscope.app/) | 现代火焰图查看器，支持多种格式 |
| [Firefox Profiler](https://profiler.firefox.com/) | 可导入 perf 数据的 Web 查看器 |
| `perf report --tui` | perf 自带的 TUI 分析界面 |

---

## 十三、速查：从零到火焰图

```bash
# 完整的一键脚本
PID=<your_pid>
DURATION=30
FREQ=99

# 采样
sudo perf record -F $FREQ --call-graph dwarf -p $PID -- sleep $DURATION

# 生成火焰图
perf script | \
  path/to/FlameGraph/stackcollapse-perf.pl | \
  path/to/FlameGraph/flamegraph.pl --title "CPU Profile" > flame.svg

echo "Done. Open flame.svg in a browser."
```
