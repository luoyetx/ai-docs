# Linux Coredump 全过程与细节

## 一、什么是 Coredump

当进程收到某些信号（如 SIGSEGV、SIGABRT）异常终止时，内核将进程的内存映像写入磁盘文件，即 core dump 文件。它是进程死亡时刻的"快照"，用于事后调试。

---

## 二、触发条件

以下信号的默认动作是 **终止进程 + 产生 core**：

| 信号 | 编号 | 典型原因 |
|------|------|---------|
| SIGSEGV | 11 | 非法内存访问 |
| SIGABRT | 6 | `abort()` 调用 |
| SIGFPE | 8 | 算术异常（如除零） |
| SIGBUS | 7 | 总线错误（对齐、映射问题） |
| SIGILL | 4 | 非法指令 |
| SIGQUIT | 3 | 用户按 `Ctrl+\` |
| SIGSYS | 31 | 非法系统调用 |
| SIGTRAP | 5 | 断点/调试陷阱 |

!!! note "不是所有致命信号都产生 core"
    SIGKILL（9）和 SIGTERM（15）的默认动作是终止进程但**不**产生 core dump。只有上表中 default action 为 "core" 的信号才会触发。

---

## 三、内核处理流程

整个过程发生在内核的信号处理路径中：

```
用户态异常（如访问非法地址）
  → CPU 触发异常（如 page fault）
  → 内核 do_page_fault()
  → force_sig(SIGSEGV)
  → get_signal() 处理挂起信号
  → sig_kernel_coredump(signr) 为 true
  → do_coredump()
```

### 3.1 `do_coredump()` 主要步骤

源码位于 `fs/coredump.c`，核心流程如下：

```
do_coredump(const kernel_siginfo_t *siginfo)
│
├── 1. 检查 RLIMIT_CORE
│     ulimit -c 为 0 → 直接返回，不产生 core
│
├── 2. 检查 dumpable 标志
│     /proc/sys/fs/suid_dumpable
│     - 0: SUID 程序不 dump（默认，安全）
│     - 1: dump（不安全，可能泄露特权进程内存）
│     - 2: dump 到 pipe handler（suidsafe）
│
├── 3. 确定 core 文件路径
│     读取 /proc/sys/kernel/core_pattern
│     - 普通路径: 如 "core.%p"
│     - 管道模式: 以 "|" 开头
│
├── 4. 冻结线程组
│     - zap_threads(): 冻结同一线程组的所有线程
│     - 等待所有线程停止，保证内存快照一致性
│
├── 5. 打开/创建输出（文件或管道）
│     - 文件模式: open + truncate
│     - 管道模式: fork 子进程执行外部 handler
│
├── 6. 调用二进制格式的 core_dump 回调
│     对 ELF 格式 → elf_core_dump()
│
└── 7. 清理
      通知父进程（SIGCHLD），wait status 包含 core 标志位
      父进程可通过 WCOREDUMP(status) 宏判断
```

### 3.2 `elf_core_dump()` — 生成 ELF core 文件

源码位于 `fs/binfmt_elf.c`。Core 文件本质上是一个 ELF 文件：

```
┌─────────────────────────┐
│      ELF Header         │  e_type = ET_CORE
├─────────────────────────┤
│   Program Headers       │
│  ┌────────────────────┐ │
│  │ PT_NOTE             │ │  ← 进程/线程元数据
│  │ PT_LOAD (vma 1)     │ │  ← 各内存段
│  │ PT_LOAD (vma 2)     │ │
│  │ ...                 │ │
│  └────────────────────┘ │
├─────────────────────────┤
│     NOTE segment        │
│  - NT_PRSTATUS (每线程) │  ← 寄存器状态 (rip, rsp, rax...)
│  - NT_PRPSINFO          │  ← 进程名、PID、状态
│  - NT_SIGINFO           │  ← 触发 core 的信号信息
│  - NT_AUXV              │  ← auxiliary vector
│  - NT_FILE              │  ← 文件映射列表 (/proc/pid/maps)
│  - NT_FPREGSET          │  ← 浮点寄存器
│  - NT_X86_XSTATE        │  ← 扩展状态 (AVX/SSE)
├─────────────────────────┤
│     LOAD segments       │
│  各 VMA 的内存内容       │  ← 堆、栈、mmap 区域、数据段...
└─────────────────────────┘
```

关键细节：

```
elf_core_dump()
│
├── 遍历所有 VMA (vm_area_struct)
│     决定哪些 VMA 需要 dump（见 coredump_filter）
│
├── 为每个线程收集寄存器状态
│     通过 task_pt_regs() 获取
│     包括通用寄存器、FP/SIMD 寄存器
│
├── 写入 ELF header
├── 写入 program headers
├── 写入 NOTE segment
├── 逐页写入各 LOAD segment 的内存内容
│     ⚠ 这是最耗时的部分
│     对于大进程（几十 GB）可能非常慢
│
└── 完成
```

---

## 四、控制参数详解

### 4.1 RLIMIT_CORE（`ulimit -c`）

控制 core 文件的最大字节数。为 0 则不产生 core。

```bash
ulimit -c 0          # 禁用 coredump
ulimit -c unlimited  # 不限制大小
```

!!! warning "ulimit 只影响当前 shell 及其子进程"
    要全局生效，需修改 `/etc/security/limits.conf`：
    ```
    *  soft  core  unlimited
    *  hard  core  unlimited
    ```

### 4.2 `core_pattern`

```bash
# 查看当前配置
cat /proc/sys/kernel/core_pattern
```

**文件模式** — 支持的占位符：

| 占位符 | 含义 |
|--------|------|
| `%p` | PID |
| `%u` | UID |
| `%g` | GID |
| `%s` | 信号编号 |
| `%t` | UNIX 时间戳 |
| `%h` | 主机名 |
| `%e` | 可执行文件名（前 16 字节） |
| `%E` | 可执行文件完整路径（`/` 替换为 `!`） |

```bash
# 示例：core 文件带进程名、PID 和时间戳
echo "core.%e.%p.%t" > /proc/sys/kernel/core_pattern

# 指定目录集中存放
echo "/var/coredumps/core.%e.%p.%t" > /proc/sys/kernel/core_pattern
```

**管道模式** — 以 `|` 开头，将 core 数据通过 stdin 传给外部程序：

```bash
echo "|/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %h" \
  > /proc/sys/kernel/core_pattern
```

### 4.3 `coredump_filter`

每个进程有独立的 coredump_filter，控制哪些类型的内存区域被 dump：

```bash
cat /proc/<pid>/coredump_filter   # 默认值 0x33
```

| Bit | 内存类型 | 默认 |
|-----|---------|------|
| 0 | anonymous private（堆、栈） | dump |
| 1 | anonymous shared | dump |
| 2 | file-backed private（代码段） | 不 dump |
| 3 | file-backed shared | 不 dump |
| 4 | ELF headers | dump |
| 5 | hugetlb private | dump |
| 6 | hugetlb shared | 不 dump |
| 7 | DAX private | 不 dump |

```bash
# dump 所有类型（调试时使用）
echo 0x7f > /proc/<pid>/coredump_filter
```

!!! tip "在代码中控制"
    ```c
    // 排除大型 mmap 区域，减小 core 文件体积
    madvise(ptr, size, MADV_DONTDUMP);

    // 重新包含
    madvise(ptr, size, MADV_DODUMP);
    ```

### 4.4 `suid_dumpable`

```bash
cat /proc/sys/fs/suid_dumpable
```

| 值 | 行为 |
|----|------|
| 0 | setuid/setgid 程序不产生 core（默认，安全） |
| 1 | 产生 core（不安全，可能泄露特权进程内存） |
| 2 | 产生 core 但只能通过管道模式处理（suidsafe） |

---

## 五、systemd-coredump（现代发行版）

大多数现代 Linux 发行版默认使用 `systemd-coredump` 作为管道处理器。

### 工作流程

```
内核 do_coredump()
  → 通过 pipe 将 core 数据发送给 systemd-coredump
  → systemd-coredump 压缩（lz4/zstd）并存储
  → 记录到 systemd journal
  → 存储位置: /var/lib/systemd/coredump/
```

### 配置

`/etc/systemd/coredump.conf`：

```ini
[Coredump]
Storage=external        # external（存文件）| journal（嵌入日志）| none
Compress=yes            # 使用压缩
MaxUse=1G              # 最大磁盘占用
ProcessSizeMax=2G      # 单个 core 最大大小
ExternalSizeMax=2G     # 外部存储单文件大小限制
```

### `coredumpctl` 常用命令

```bash
# 列出所有 core dump
coredumpctl list

# 查看最近一次 core dump 信息
coredumpctl info

# 查看指定进程的 core dump
coredumpctl info <PID>

# 直接启动 gdb 调试
coredumpctl debug <PID>

# 导出 core 文件
coredumpctl dump <PID> -o /tmp/core.file

# 按可执行文件名过滤
coredumpctl list myapp
```

---

## 六、实际调试流程

### 6.1 准备工作

```bash
# 1. 确保 coredump 开启
ulimit -c unlimited

# 2. 编译时带调试信息
gcc -g -O0 buggy.c -o buggy

# 3. 确认 core_pattern 设置
cat /proc/sys/kernel/core_pattern
```

### 6.2 用 GDB 分析 core 文件

```bash
# 加载 core 文件
gdb ./buggy core.12345
```

常用 GDB 命令：

```gdb
(gdb) bt                 # 查看完整调用栈（最重要）
(gdb) bt full            # 调用栈 + 局部变量
(gdb) info threads       # 查看所有线程
(gdb) thread 2           # 切换到线程 2
(gdb) thread apply all bt # 打印所有线程的调用栈

(gdb) info reg           # 查看寄存器
(gdb) x/10i $rip         # 查看崩溃处的汇编指令
(gdb) print var          # 查看变量值
(gdb) info proc mappings # 查看内存映射

(gdb) frame 3            # 切换到第 3 层栈帧
(gdb) info locals        # 查看当前帧的局部变量
(gdb) info args          # 查看当前帧的函数参数
```

### 6.3 示例：分析一个空指针解引用

```c
// buggy.c
#include <stdio.h>
#include <stdlib.h>

struct Node {
    int value;
    struct Node *next;
};

void process(struct Node *node) {
    printf("value = %d\n", node->next->value);  // next 为 NULL 时 crash
}

int main() {
    struct Node n = {42, NULL};
    process(&n);
    return 0;
}
```

```bash
$ gcc -g -O0 buggy.c -o buggy
$ ./buggy
Segmentation fault (core dumped)

$ gdb ./buggy core.xxxxx
(gdb) bt
#0  0x0000555555555169 in process (node=0x7fffffffde10) at buggy.c:10
#1  0x00005555555551a0 in main () at buggy.c:15

(gdb) frame 0
(gdb) print node->next
$1 = (struct Node *) 0x0          # 确认 next 为 NULL

(gdb) info reg rax
rax  0x0                          # 解引用了空指针
```

---

## 七、大型进程的 Coredump 优化

对于内存占用很大的进程（如数据库、搜索引擎），直接 dump 全量内存会非常慢且占磁盘。

### 7.1 减小 core 文件体积

```bash
# 只 dump 匿名内存（堆、栈），跳过 file-backed 映射
echo 0x33 > /proc/<pid>/coredump_filter

# 在代码中排除大型索引文件的 mmap 区域
madvise(index_ptr, index_size, MADV_DONTDUMP);
```

### 7.2 使用 `gcore` 在线生成 core

不需要让进程崩溃也能获取 core dump：

```bash
# 暂停进程并生成 core（进程不会终止）
gcore <pid>

# GDB 内部
(gdb) attach <pid>
(gdb) generate-core-file /tmp/mycore
(gdb) detach
```

### 7.3 使用 `google-coredumper`（非阻塞方式）

Google 开源的库，可以在进程运行时 fork 一个子进程来写 core，主进程几乎不停顿：

```c
#include <google/coredumper.h>
WriteCoreDump("core.manual");  // fork + dump，主进程继续运行
```

---

## 八、常见问题排查

**没有生成 core 文件？** 按以下顺序逐步检查：

```bash
# 1. 检查 ulimit
ulimit -c
# 如果为 0，设置 ulimit -c unlimited

# 2. 检查 core_pattern
cat /proc/sys/kernel/core_pattern
# 如果以 | 开头，说明走管道模式，用 coredumpctl 查看

# 3. 检查目标目录的写权限
# core_pattern 指向的目录是否存在、是否可写

# 4. 检查 suid_dumpable（对 setuid 程序）
cat /proc/sys/fs/suid_dumpable

# 5. 检查磁盘空间
df -h

# 6. 检查进程是否自行捕获了信号
# 如果进程注册了 signal(SIGSEGV, handler)，内核不会走 coredump 路径

# 7. 检查 cgroup/容器限制
# Docker 默认可能限制了 core dump
# docker run --ulimit core=-1 ...

# 8. 检查 AppArmor/SELinux 是否阻止了写入
dmesg | grep -i denied
```

**core 文件被截断？**

- 检查 `RLIMIT_CORE` 是否设置了上限
- 检查 `systemd-coredump` 的 `ProcessSizeMax` 配置
- 检查磁盘剩余空间

**GDB 提示 "not in executable format"？**

- 确保 GDB 的参数顺序正确：`gdb <executable> <core-file>`
- 确保可执行文件版本与 core dump 匹配（同一次编译产物）

---

## 九、Coredump 与安全

Core dump 包含进程的完整内存，可能包含：

- 密码、密钥、token
- 用户隐私数据
- 加密中间状态

**安全建议：**

- 生产环境慎重开启，或仅 dump 到安全存储
- 设置 `suid_dumpable=0`（默认）
- 使用 `prctl(PR_SET_DUMPABLE, 0)` 在代码中禁止特定进程的 core dump
- 限制 core 文件的存储目录权限（`0700`）
- 定期清理旧的 core 文件
