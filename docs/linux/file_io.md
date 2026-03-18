# Linux 文件 I/O 系统性总结

## 一、I/O 路径全景

```
用户程序
   │
   ├── buffered I/O (stdio)     fread/fwrite → 用户态缓冲区 → syscall
   │
   ├── standard syscall I/O     read/write   → page cache → 磁盘
   │
   ├── mmap                     虚拟地址映射 → page cache → 磁盘
   │
   ├── direct I/O (O_DIRECT)    read/write   → 绕过 page cache → 磁盘
   │
   └── io_uring / AIO           异步提交，完成后通知

内核层
   ├── page cache（文件页缓存）
   ├── block layer（I/O 调度）
   └── 设备驱动
```

---

## 二、Buffered I/O（用户态缓冲，stdio）

### 原理

`FILE *` 接口在用户态维护一个缓冲区（默认 8KB），`fread/fwrite` 先操作这块缓冲，攒够一批再调 `read/write` syscall。

```c
FILE *f = fopen("file.txt", "r");
char buf[1024];
fread(buf, 1, 1024, f);
fclose(f);
```

### 优点
- 减少 syscall 次数，适合大量小 I/O
- 提供 `fgets`/`fprintf` 等格式化接口

### 缺点
- 多一层缓冲，数据经过：用户态 buffer → page cache → 磁盘，延迟略高
- 线程不安全（需要加锁或用 `flockfile`）
- 缓冲区大小固定，不适合大块随机读

### 适用场景
- 日志写入、配置文件读取、文本处理
- 需要行缓冲或格式化输出的场景

---

## 三、Standard syscall I/O（内核缓冲，read/write）

### 原理

直接调用 `read/write` syscall，数据经过内核 **page cache**。写入时先写入 page cache（dirty page），由内核定期或手动 `fsync` 刷盘。

```c
int fd = open("file", O_RDONLY);
char buf[4096];
ssize_t n = read(fd, buf, sizeof(buf));
close(fd);
```

### Page Cache 行为
- **读**：先查 page cache，命中则直接返回；未命中触发 page fault，从磁盘读入 page cache 再返回
- **写**：写入 page cache 即返回，标记为 dirty；内核后台 writeback 刷盘
- **Readahead**：顺序读时内核自动预读后续页（默认 128KB），提升顺序读吞吐

```c
// 控制 readahead 行为
posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);  // 顺序读，加大预读
posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);      // 随机读，关闭预读
posix_fadvise(fd, 0, 0, POSIX_FADV_DONTNEED);    // 读完通知内核可以回收
```

### 优点
- 简单直接
- page cache 透明加速重复读
- 顺序写有 writeback 聚合，实际吞吐高

### 缺点
- 数据要在用户 buf 和 page cache 之间 copy（一次拷贝）
- page cache 占用内存，大文件可能 evict 热数据
- `write` 返回不代表落盘，需要 `fsync` 保证持久性

### 适用场景
- 通用文件读写
- 数据库 WAL 日志（结合 `fsync`）
- 大多数应用程序的默认选择

---

## 四、mmap（内存映射）

### 原理

将文件映射到进程虚拟地址空间，访问内存即访问文件。底层仍通过 page cache，但不需要 `read/write` 拷贝——直接访问 page cache 中的物理页。

```c
int fd = open("file", O_RDONLY);
struct stat st;
fstat(fd, &st);
void *p = mmap(NULL, st.st_size, PROT_READ, MAP_SHARED, fd, 0);
// 直接访问 p[offset]，page fault 时内核自动从磁盘读入
munmap(p, st.st_size);
```

### 关键 flags

| Flag | 含义 |
|------|------|
| `MAP_SHARED` | 映射到 page cache，写操作直接修改 page cache（对其他进程可见）|
| `MAP_PRIVATE` | 写时复制（COW），修改不影响原文件 |
| `MAP_POPULATE` | mmap 时立即预读所有页，避免运行时 page fault |
| `MAP_LOCKED` | 等同 mlock，禁止页被 swap 出去 |
| `MAP_HUGETLB` | 使用大页（2MB/1GB），需要系统预留 hugepage |
| `MAP_ANONYMOUS` | 匿名映射，不关联文件（用于内存分配）|

### madvise 调优

```c
madvise(p, size, MADV_SEQUENTIAL);   // 顺序访问，内核加大预读
madvise(p, size, MADV_RANDOM);       // 随机访问，关闭预读
madvise(p, size, MADV_WILLNEED);     // 预热，提示内核预读
madvise(p, size, MADV_DONTNEED);     // 用完，提示内核可以回收
madvise(p, size, MADV_HUGEPAGE);     // 启用透明大页（THP）
madvise(p, size, MADV_DONTDUMP);     // 排除在 core dump 之外（大索引文件常用）
```

### 零拷贝优势

```
read/write:  磁盘 → page cache → 用户 buf     （1次拷贝）
mmap:        磁盘 → page cache               （0次拷贝，直接访问）
```

### mmap vs read 的选择边界

- **文件较小（< 几十 MB）**：`read` 通常更快，mmap 的 VMA 管理开销不可忽略
- **大文件随机读**：mmap 更好，避免每次 `read` syscall + 拷贝
- **多进程共享**：`MAP_SHARED` 共享同一份 page cache，节省物理内存
- **需要就地修改文件**：mmap + `MAP_SHARED` + `PROT_WRITE` 最简单

### 大页（Huge Pages）与 TLB

TLB（Translation Lookaside Buffer）是页表缓存，容量有限（通常 1500 个 entry）。

| 页大小 | 1GB 文件所需 TLB entry |
|--------|----------------------|
| 4KB    | 262,144              |
| 2MB    | 512                  |
| 1GB    | 1                    |

对大文件的随机读，TLB miss 是主要瓶颈之一，大页可以显著改善。

**两种大页方式：**

1. **显式大页（HugeTLB）**：`MAP_HUGETLB`，需要管理员预留（`/proc/sys/vm/nr_hugepages`），不灵活
2. **透明大页（THP）**：`MADV_HUGEPAGE`，内核自动在后台将连续 4KB 页合并成 2MB 大页，对匿名内存效果好，对 file-backed mmap 效果略差（page cache 分配更碎片化）

### 适用场景
- 大型只读索引文件（搜索引擎、数据库索引）
- 数据库 buffer pool（如 InnoDB 用 mmap 管理共享内存）
- 需要多进程共享同一文件内容
- 实现简单的持久化 key-value store

---

## 五、Direct I/O（O_DIRECT）

### 原理

绕过 page cache，数据直接在用户 buf 和磁盘设备之间传输。

```c
// 注意：buffer、offset、length 都必须按 512B 或 4096B 对齐
int fd = open("file", O_RDONLY | O_DIRECT);

void *buf;
posix_memalign(&buf, 4096, 4096 * 1024);  // 对齐分配
ssize_t n = read(fd, buf, 4096 * 1024);   // offset 和 length 也必须对齐
```

### 对齐要求

内核要求（以 Linux 为例）：
- buffer 地址：按 `logical_block_size` 对齐（通常 512B，SSD 可能 4096B）
- 读写长度：按 `logical_block_size` 对齐
- 文件偏移：按 `logical_block_size` 对齐

不满足对齐会返回 `EINVAL`。

### 优点
- 完全避免 page cache，不污染缓存（适合一次性大文件读取）
- 写入延迟更可预测（没有 writeback 的不确定性）
- 数据库可以自己管理 buffer pool，不需要 OS 帮忙缓存

### 缺点
- 使用复杂，必须处理对齐
- 小 I/O 性能差（没有 readahead 聚合）
- 不适合随机小读，每次都是真实磁盘 I/O

### 适用场景
- 数据库自管理 buffer pool（PostgreSQL、MySQL InnoDB 可选开启）
- 大文件备份/复制（不希望污染 page cache）
- 需要精确控制 I/O 时序的场景
- 与匿名 mmap 结合做大文件预热（Vespa 的做法）

---

## 六、异步 I/O

### 6.1 POSIX AIO（不推荐）

```c
struct aiocb cb = {
    .aio_fildes = fd,
    .aio_buf = buf,
    .aio_nbytes = size,
    .aio_offset = 0,
};
aio_read(&cb);
// ... 做其他事 ...
aio_suspend(&cb, 1, NULL);  // 等待完成
```

Linux 的 POSIX AIO 实现用线程模拟，性能差，不建议使用。

### 6.2 io_uring（推荐，Linux 5.1+）

```c
struct io_uring ring;
io_uring_queue_init(32, &ring, 0);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, size, offset);
io_uring_sqe_set_data(sqe, user_data);

io_uring_submit(&ring);

struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
// cqe->res 是返回值
io_uring_cqe_seen(&ring, cqe);
```

**io_uring 的核心优势：**
- 共享内存环形队列，批量提交减少 syscall 次数
- 支持 `SQPOLL` 模式：内核线程轮询 SQ，零 syscall 提交
- 支持 fixed buffers / fixed files，减少内核态地址转换开销
- 真正的内核原生异步，不依赖线程
- 支持链式操作（`IOSQE_IO_LINK`），如 read 完自动 write

### io_uring 适用场景
- 高并发网络 + 文件 I/O 混合（如高性能 web server）
- 需要批量发起大量 I/O 请求
- 数据库存储引擎的底层 I/O 层
- 替代 epoll + 线程池的架构

---

## 七、sendfile 与零拷贝

```c
// 将文件内容直接发送到 socket，全程在内核完成
sendfile(socket_fd, file_fd, &offset, size);
```

数据路径：
```
磁盘 → page cache → socket buffer   （内核内部，无用户态拷贝）
```

适用场景：静态文件服务（nginx 的核心优化之一）。

---

## 八、横向对比

| 方式 | 拷贝次数 | page cache | 对齐要求 | 随机读 | 顺序读 | 适用场景 |
|------|---------|-----------|---------|--------|--------|---------|
| stdio (buffered) | 2 | 经过 | 无 | 差 | 好 | 文本/日志 |
| read/write | 1 | 经过 | 无 | 中 | 好 | 通用 |
| mmap | 0 | 经过 | 无 | 好 | 好 | 大文件随机读，多进程共享 |
| O_DIRECT | 1 | 绕过 | 必须 | 差（无缓存）| 好（大块）| 数据库 buffer pool |
| mmap + DirectIO | 0 | 绕过 | 自行处理 | 好 | 好 | 大索引预热（Vespa）|
| io_uring | 1 | 经过 | 无 | 好 | 好 | 高并发异步 I/O |
| sendfile | 0 | 经过 | 无 | - | 极好 | 静态文件服务 |

---

## 九、大型只读索引文件的最佳实践

这是搜索引擎、向量数据库等场景的典型需求。

### 场景一：内存能装下整个文件，延迟敏感

```c
int fd = open("index.bin", O_RDONLY);
void *p = mmap(NULL, size, PROT_READ, MAP_SHARED | MAP_POPULATE, fd, 0);
madvise(p, size, MADV_HUGEPAGE);   // 启用 THP
madvise(p, size, MADV_RANDOM);     // 关闭 readahead（随机访问）
madvise(p, size, MADV_DONTDUMP);   // 不写入 core dump
mlock(p, size);                    // 防止被 swap
close(fd);  // mmap 后 fd 可以关闭
```

### 场景二：内存放不下，依赖热点缓存

```c
int fd = open("index.bin", O_RDONLY);
void *p = mmap(NULL, size, PROT_READ, MAP_SHARED, fd, 0);
madvise(p, size, MADV_RANDOM);  // 关闭 readahead，每次只读需要的页
// 让 OS page cache 做 LRU，热点数据自然留在内存
```

### 场景三：不想污染 page cache，需要大页（Vespa 方案）

```c
// 1. 分配匿名内存（对齐到 DirectIO 要求）
void *p = mmap(NULL, total, PROT_READ | PROT_WRITE,
               MAP_ANON | MAP_PRIVATE, -1, 0);
madvise(p, total, MADV_HUGEPAGE);

// 2. 用 DirectIO 将文件内容读入
int fd = open("index.bin", O_RDONLY | O_DIRECT);
pread(fd, p, total, 0);

// 3. 改为只读保护
mprotect(p, total, PROT_READ);
```

---

## 十、fsync 与数据持久性

写入 `write` 后数据只在 page cache（dirty page），断电会丢失。

```c
write(fd, data, size);   // 写入 page cache
fsync(fd);               // 强制刷盘，返回后数据持久化
fdatasync(fd);           // 只刷数据，不刷 metadata（inode mtime等），更快
```

**常见策略：**

| 策略 | 持久性 | 性能 | 典型场景 |
|------|--------|------|---------|
| 不 fsync | 无保证 | 最高 | 可重建的缓存文件 |
| 定期 fsync | 丢失最近 N 秒 | 高 | 日志（允许少量丢失）|
| 每次写后 fsync | 强保证 | 低 | 数据库事务提交 |
| `O_DSYNC` | 强保证 | 中 | 每次 write 自动 fdatasync |
| WAL + group commit | 强保证 | 高 | 数据库标准做法 |

---

## 十一、常用调优工具

```bash
# 查看 page cache 使用情况
cat /proc/meminfo | grep -E "Cached|Buffers|Dirty"

# 统计进程内存映射
cat /proc/<pid>/maps
cat /proc/<pid>/smaps  # 更详细，包含 RSS、PSS、Huge Pages

# 查看文件是否在 page cache 中
fincore <file>           # 需要安装 linux-ftools
vmtouch <file>           # 更常用

# 主动预热文件到 page cache
vmtouch -t <file>

# 清空 page cache（测试用）
echo 3 > /proc/sys/vm/drop_caches

# 查看 THP 使用情况
cat /proc/meminfo | grep Huge
cat /sys/kernel/mm/transparent_hugepage/enabled

# I/O 性能分析
iostat -x 1
iotop
perf trace -e 'read,write,mmap'
```

---

## 十二、总结选择流程

```
需要文件 I/O？
│
├── 需要格式化/行处理？ → stdio (fread/fwrite)
│
├── 普通读写，内容不大？ → read/write
│
├── 大文件随机读，进程内独享？
│   ├── 内存能装下 → mmap + MAP_POPULATE + mlock + MADV_HUGEPAGE
│   └── 内存装不下 → mmap + MADV_RANDOM（靠 page cache LRU）
│
├── 大文件随机读，多进程共享？ → mmap + MAP_SHARED
│
├── 不想污染 page cache？ → O_DIRECT（数据库 buffer pool）
│
├── 大文件一次性预热 + 大页 + 不污染 cache？
│   → MAP_ANON + MADV_HUGEPAGE + O_DIRECT ReadBuf（Vespa 方案）
│
├── 高并发 I/O，需要异步？ → io_uring
│
└── 文件内容发到网络？ → sendfile
```
