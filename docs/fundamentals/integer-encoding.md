# 整数编解码：从 Varint 到 BitPacking 的完整指南

## 一、为什么需要整数编码

在存储和传输系统中，整数无处不在——文档 ID、时间戳、偏移量、计数器、索引值。朴素的定长编码（如 4 字节 `int32`）简单高效，但存在明显浪费：大量整数的实际值远小于类型所能表示的最大值。例如一个值为 `300` 的 `int64` 占用 8 字节，但有效信息仅需 2 字节。

```
场景                    数据特征                    核心需求
──────────────────────────────────────────────────────────────────
RPC 协议 (Protobuf)     单个整数逐一编码              低延迟、简单
搜索引擎倒排索引        百万级递增文档 ID 列表         极致压缩率
列式数据库 (Parquet)    同列大量相近整数              批量解码速度
视频编码 (H.264)        bit 级变长系数                bit 对齐、零开销
```

整数编码的设计空间可以沿两个维度展开：

```
               逐个编码                    批量编码
            ┌─────────────┐           ┌─────────────────┐
字节对齐    │ LEB128       │           │ Group Varint     │
            │ PrefixVarint │           │ Stream VByte     │
            │ SQLite Varint│           │                  │
            ├─────────────┤           ├─────────────────┤
位对齐      │ Exp-Golomb   │           │ BitPacking       │
            │ VLQ          │           │ Simple8b         │
            │              │           │ FOR / PFOR       │
            └─────────────┘           └─────────────────┘
```

本文将沿着这张地图，从最基础的字节序开始，逐步深入到生产级系统的编码选型。

---

## 二、定长编码基础

### 2.1 字节序（Byte Order）

一个多字节整数在内存中的存放顺序有两种约定：

```
整数 0x12345678 的存储方式：

Big-Endian（大端序）—— 高位字节在低地址
地址:  0x00   0x01   0x02   0x03
内容:  0x12   0x34   0x56   0x78
       ↑ 最高有效字节 (MSB)

Little-Endian（小端序）—— 低位字节在低地址
地址:  0x00   0x01   0x02   0x03
内容:  0x78   0x56   0x34   0x12
       ↑ 最低有效字节 (LSB)
```

| 字节序 | 使用场景 | 原因 |
|--------|----------|------|
| Big-Endian | 网络协议（TCP/IP）、Java `.class` 文件 | 人类阅读习惯一致，方便调试 |
| Little-Endian | x86/x64、ARM（默认）、大部分现代 CPU | 硬件加法器从低位开始，截断只需改长度 |

### 2.2 C++ 字节序转换

```cpp
#include <cstdint>
#include <bit>        // C++20 std::endian
#include <algorithm>  // std::reverse

// 编译期判断本机字节序（C++20）
constexpr bool is_little_endian() {
    return std::endian::native == std::endian::little;
}

// 通用字节交换
inline uint16_t bswap16(uint16_t x) { return __builtin_bswap16(x); }
inline uint32_t bswap32(uint32_t x) { return __builtin_bswap32(x); }
inline uint64_t bswap64(uint64_t x) { return __builtin_bswap64(x); }

// 主机序 → 网络序（Big-Endian）
inline uint32_t to_big_endian(uint32_t x) {
    if constexpr (std::endian::native == std::endian::little) {
        return bswap32(x);
    }
    return x;
}

// 网络序 → 主机序
inline uint32_t from_big_endian(uint32_t x) {
    return to_big_endian(x);  // 自反操作
}
```

!!! tip "Little-Endian 的截断优势"
    Little-Endian 下，一个 `uint64_t` 的低 4 字节地址与 `uint32_t` 完全重合，因此 `uint64_t*` 可以直接强制转换为 `uint32_t*` 读取低 32 位（在对齐正确的前提下）。这也是为什么很多变长编码在 Little-Endian 平台上实现更简洁。

---

## 三、逐个变长编码

### 3.1 LEB128（Little Endian Base 128）

LEB128 是最经典的变长整数编码，被 DWARF 调试信息格式和 WebAssembly 采用。核心思想：**每个字节用 7 bit 存数据，最高位（MSB）作为延续标志**。

```
MSB 含义：
  1xxxxxxx  →  后面还有更多字节
  0xxxxxxx  →  这是最后一个字节

编码 300 (= 0b100101100) 的过程：

  二进制:         1 0010 1100
  按 7-bit 分组:  0000010  0101100
                  ↓ 高组    ↓ 低组
  先发低组:       低组加 MSB=1:  1_0101100 = 0xAC
                  高组加 MSB=0:  0_0000010 = 0x02

  编码结果:  [0xAC, 0x02]  (2 字节)
```

#### 值范围与字节数对应

| 字节数 | 无符号范围 | 有效 bit 数 |
|--------|-----------|------------|
| 1 | 0 ~ 127 | 7 |
| 2 | 128 ~ 16,383 | 14 |
| 3 | 16,384 ~ 2,097,151 | 21 |
| 4 | 2,097,152 ~ 268,435,455 | 28 |
| 5 | 268,435,456 ~ 4,294,967,295 | 35 |
| 10 | 最大（uint64） | 70 |

!!! warning "LEB128 的 64 位整数最多需要 10 字节"
    64 / 7 = 9.14，向上取整为 10 字节。这意味着最坏情况下，LEB128 比定长 8 字节编码还多 2 字节。这是所有 MSB-continuation 方案的固有代价。

#### ULEB128 编解码（C++）

```cpp
// 编码 ULEB128，返回写入字节数
int encode_uleb128(uint64_t value, uint8_t* buf) {
    int len = 0;
    do {
        uint8_t byte = value & 0x7F;  // 取低 7 位
        value >>= 7;
        if (value != 0) {
            byte |= 0x80;  // 设置延续位
        }
        buf[len++] = byte;
    } while (value != 0);
    return len;
}

// 解码 ULEB128，返回读取字节数
int decode_uleb128(const uint8_t* buf, uint64_t* result) {
    *result = 0;
    int shift = 0;
    int len = 0;
    do {
        uint64_t byte = buf[len];
        *result |= (byte & 0x7F) << shift;
        shift += 7;
        len++;
    } while (buf[len - 1] & 0x80);
    return len;
}
```

#### SLEB128（有符号 LEB128）

有符号版本使用符号扩展：解码时如果最后一个字节的第 6 位（即 bit 6）为 1，则高位全部填 1。

```cpp
// 编码 SLEB128
int encode_sleb128(int64_t value, uint8_t* buf) {
    int len = 0;
    bool more = true;
    while (more) {
        uint8_t byte = value & 0x7F;
        value >>= 7;  // 算术右移，保留符号
        // 如果剩余值全 0 且当前字节符号位为 0，或
        // 剩余值全 -1 且当前字节符号位为 1，则结束
        if ((value == 0 && !(byte & 0x40)) ||
            (value == -1 && (byte & 0x40))) {
            more = false;
        } else {
            byte |= 0x80;
        }
        buf[len++] = byte;
    }
    return len;
}

// 解码 SLEB128
int decode_sleb128(const uint8_t* buf, int64_t* result) {
    *result = 0;
    int shift = 0;
    int len = 0;
    uint8_t byte;
    do {
        byte = buf[len++];
        *result |= (int64_t)(byte & 0x7F) << shift;
        shift += 7;
    } while (byte & 0x80);
    // 符号扩展
    if (shift < 64 && (byte & 0x40)) {
        *result |= -(int64_t(1) << shift);
    }
    return len;
}
```

### 3.2 Protocol Buffers Varint

Protobuf 的 Varint 本质就是 ULEB128，编码规则完全一致。区别在于 Protobuf 围绕它构建了完整的类型系统：

```
Protobuf wire type 0 (Varint) 编码的类型：
  int32, int64    → 直接 Varint 编码（负数会占 10 字节！）
  uint32, uint64  → 直接 Varint 编码
  sint32, sint64  → ZigZag 编码后再 Varint
  bool            → 0 或 1 的 Varint
  enum            → 对应整数值的 Varint
```

!!! warning "int32 负数陷阱"
    Protobuf 的 `int32` 类型编码负数时，会先符号扩展到 64 位再做 Varint 编码。因此 `-1` 会编码为 10 字节的 `0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0x01`。如果字段可能为负数，务必使用 `sint32`/`sint64`。

### 3.3 ZigZag 编码

ZigZag 编码解决了有符号整数在 Varint 中的效率问题。核心思想：**将有符号整数映射为无符号整数，使绝对值小的数映射到小的无符号值**。

```
映射关系：
  原始值     ZigZag 编码值
   0    →    0
  -1    →    1
   1    →    2
  -2    →    3
   2    →    4
  -3    →    5
   3    →    6
  ...        ...
   n    →    2n        (n >= 0)
  -n    →    2n - 1    (n > 0)
```

编码/解码是纯位运算，无分支：

```cpp
// ZigZag 编码：有符号 → 无符号
inline uint32_t zigzag_encode32(int32_t n) {
    return (static_cast<uint32_t>(n) << 1) ^ static_cast<uint32_t>(n >> 31);
}

inline uint64_t zigzag_encode64(int64_t n) {
    return (static_cast<uint64_t>(n) << 1) ^ static_cast<uint64_t>(n >> 63);
}

// ZigZag 解码：无符号 → 有符号
inline int32_t zigzag_decode32(uint32_t n) {
    return static_cast<int32_t>((n >> 1) ^ -(n & 1));
}

inline int64_t zigzag_decode64(uint64_t n) {
    return static_cast<int64_t>((n >> 1) ^ -(n & 1));
}
```

!!! note "位运算解析"
    以 `zigzag_encode32` 为例：`n >> 31` 是算术右移，正数得 `0x00000000`，负数得 `0xFFFFFFFF`。与左移一位的值异或后：正数 `n` 变成 `2n`，负数 `-n` 变成 `2n-1`。整个过程只需 2 条指令，零分支。

#### ZigZag + Varint 的效果

| 原始值 | 直接 Varint 字节数 | ZigZag + Varint 字节数 |
|--------|-------------------|----------------------|
| 0 | 1 | 1 |
| -1 | 10（int64） | 1 |
| 1 | 1 | 1 |
| -64 | 10 | 1 |
| 64 | 1 | 2 |
| -1000 | 10 | 2 |

### 3.4 SQLite Varint

SQLite 使用了一种改良的变长编码，最大 9 字节即可表示 64 位整数（而非 LEB128 的 10 字节）。其规则如下：

```
前缀字节 A0 的值决定编码长度：

A0 值范围         编码长度   数据存储方式
─────────────────────────────────────────────────
0 ~ 240           1 字节    值 = A0
241 ~ 248         2 字节    值 = 240 + 256*(A0-241) + A1
249               3 字节    值 = 2288 + 256*A1 + A2
250               4 字节    Big-Endian 3 字节无符号整数
251               5 字节    Big-Endian 4 字节无符号整数
252               6 字节    Big-Endian 6 字节带 48-bit
253               7 字节    Big-Endian 6 字节无符号整数
254               8 字节    Big-Endian 7 字节无符号整数
255               9 字节    Big-Endian 8 字节完整 int64
```

!!! tip "SQLite Varint 的设计哲学"
    与 LEB128 逐字节试探不同，SQLite Varint 只需看第一个字节就能确定编码长度，这对解码器非常友好——可以一次性读取所有需要的字节，无需循环。代价是编码规则更复杂，且对小整数的空间效率略低于 LEB128（0~127 占 1 字节 vs SQLite 的 0~240 占 1 字节，SQLite 在这个区间反而更优）。

### 3.5 VLQ（Variable-Length Quantity）

VLQ 与 LEB128 使用相同的 MSB 延续位机制，但**字节顺序相反**——高位组在前（Big-Endian 语义）：

```
编码 300 (= 0b100101100)：

LEB128:  低位组在前 → [0xAC, 0x02]
VLQ:     高位组在前 → [0x82, 0x2C]

VLQ 编码过程：
  按 7-bit 分组:  0000010  0101100
                  高组 ↓    低组 ↓
  先发高组:       高组加 MSB=1:  1_0000010 = 0x82
                  低组加 MSB=0:  0_0101100 = 0x2C
```

VLQ 主要用在两个场景：

- **MIDI 文件格式**：delta-time 字段使用 VLQ 编码
- **Source Map v3**：使用 Base64 VLQ 编码偏移量（用 Base64 字符集替代原始字节）

```cpp
// VLQ 编码
int encode_vlq(uint64_t value, uint8_t* buf) {
    // 先确定需要多少个 7-bit 组
    uint8_t temp[10];
    int n = 0;
    do {
        temp[n++] = value & 0x7F;
        value >>= 7;
    } while (value != 0);
    // 逆序输出，高位组在前
    int len = 0;
    for (int i = n - 1; i >= 0; i--) {
        buf[len++] = temp[i] | (i > 0 ? 0x80 : 0x00);
    }
    return len;
}
```

### 3.6 PrefixVarint

PrefixVarint（也称前缀变长编码）的思路是：**用第一个字节的前缀 bits 编码整个值的字节长度**，而非像 LEB128 那样每个字节都带延续位。

一种常见实现是用 leading zeros 的个数表示后续字节数：

```
第一个字节的前缀模式：

前缀         后续字节数   有效数据 bits   值范围
──────────────────────────────────────────────────
1xxxxxxx     0           7              0 ~ 127
01xxxxxx     1           6 + 8 = 14     128 ~ 16,511
001xxxxx     2           5 + 16 = 21    16,512 ~ 2,113,663
0001xxxx     3           4 + 24 = 28    ...
00001xxx     4           3 + 32 = 35
000001xx     5           2 + 40 = 42
0000001x     6           1 + 48 = 49
00000001     7           0 + 56 = 56
00000000     8           64             完整 64-bit
```

```cpp
// PrefixVarint 编码
int encode_prefix_varint(uint64_t value, uint8_t* buf) {
    if (value < 128) {
        buf[0] = static_cast<uint8_t>(value) | 0x80;
        return 1;
    }
    // 计算需要的字节数
    int bits_needed = 64 - __builtin_clzll(value);  // 有效位数
    int extra_bytes;
    if (bits_needed <= 14) extra_bytes = 1;
    else if (bits_needed <= 21) extra_bytes = 2;
    else if (bits_needed <= 28) extra_bytes = 3;
    else if (bits_needed <= 35) extra_bytes = 4;
    else if (bits_needed <= 42) extra_bytes = 5;
    else if (bits_needed <= 49) extra_bytes = 6;
    else if (bits_needed <= 56) extra_bytes = 7;
    else extra_bytes = 8;

    int total = extra_bytes + 1;
    // 前缀标记: bit (8-total) 为 1，更高位为 0
    uint8_t prefix_mask = (total < 9) ? (0x80 >> extra_bytes) : 0x00;
    // 将 value 的高位嵌入第一个字节
    int data_bits_in_first = (total < 9) ? (7 - extra_bytes) : 0;
    buf[0] = prefix_mask | static_cast<uint8_t>(
        (value >> (extra_bytes * 8)) & ((1 << data_bits_in_first) - 1));
    // 后续字节用 Big-Endian 存储
    for (int i = extra_bytes; i >= 1; i--) {
        buf[total - i] = static_cast<uint8_t>(value >> ((i - 1) * 8));
    }
    return total;
}

// PrefixVarint 解码
int decode_prefix_varint(const uint8_t* buf, uint64_t* result) {
    uint8_t first = buf[0];
    if (first & 0x80) {
        *result = first & 0x7F;
        return 1;
    }
    // 数 leading zeros 确定后续字节数
    int leading_zeros = __builtin_clz(static_cast<uint32_t>(first) << 24);
    int extra_bytes = leading_zeros + 1;
    int total = extra_bytes + 1;

    if (extra_bytes >= 8) {
        // 特殊处理：前缀为 0x00，后续 8 字节完整值
        *result = 0;
        for (int i = 1; i <= 8; i++) {
            *result = (*result << 8) | buf[i];
        }
        return 9;
    }

    int data_bits_in_first = 7 - extra_bytes;
    *result = first & ((1 << data_bits_in_first) - 1);
    for (int i = 1; i <= extra_bytes; i++) {
        *result = (*result << 8) | buf[i];
    }
    return total;
}
```

#### PrefixVarint vs LEB128

| 特性 | LEB128 | PrefixVarint |
|------|--------|-------------|
| 长度确定 | 需逐字节扫描延续位 | 看第一个字节即可 |
| 最大字节数（64-bit） | 10 | 9 |
| 有效数据比例 | 7/8 = 87.5% | 约 89%（平均） |
| 解码速度 | 较慢（循环 + 分支） | 较快（查表 + memcpy） |
| 实现复杂度 | 简单 | 中等 |
| 典型应用 | DWARF, WebAssembly, Protobuf | Cap'n Proto, FlatBuffers 内部 |

### 3.7 Exp-Golomb 编码

Exp-Golomb（指数哥伦布编码）是一种位对齐的通用码（universal code），广泛用于 H.264/H.265 视频编码标准。

#### 0 阶 Exp-Golomb（Exp-Golomb-0）

编码规则：将值 `x` 编码为 `[M 个 0] [1] [M 位信息]`，其中 `M = floor(log2(x+1))`。

```
编码过程（0 阶）：

值 x  →  code_num = x + 1  →  二进制  →  前补零

x=0:  code_num=1   二进制=1          编码: 1           (1 bit)
x=1:  code_num=2   二进制=10         编码: 010         (3 bits)
x=2:  code_num=3   二进制=11         编码: 011         (3 bits)
x=3:  code_num=4   二进制=100        编码: 00100       (5 bits)
x=4:  code_num=5   二进制=101        编码: 00101       (5 bits)
x=5:  code_num=6   二进制=110        编码: 00110       (5 bits)
x=6:  code_num=7   二进制=111        编码: 00111       (5 bits)
x=7:  code_num=8   二进制=1000       编码: 0001000     (7 bits)

规律: 编码长度 = 2*floor(log2(x+1)) + 1 bits
```

```
位布局：

      ←── M 个零 ──→ 1 ←── M 位信息 ──→
      0 0 0 ... 0    1   i i i ... i
      └────────────┘    └──────────────┘
       前缀长度指示         数据部分

解码: 数 leading zeros 得 M → 读后续 M 位 → value = 2^M + info - 1
```

#### k 阶 Exp-Golomb

k 阶编码将值的低 k 位直接附加在后面：

```
编码 x（k 阶）：
  q = x >> k           (商)
  r = x & ((1<<k)-1)   (余数)
  输出: 0阶编码(q) + r的k位二进制

例: 编码 x=13, k=2
  q = 13 >> 2 = 3
  r = 13 & 3 = 1
  0阶编码(3) = 00100
  r 的 2-bit = 01
  最终: 0010001  (7 bits)
```

H.264 使用多种 Exp-Golomb 变体：

| 语法元素 | 编码方式 | 用途 |
|---------|---------|------|
| `ue(v)` | 无符号 0 阶 | 宏块类型、参考帧索引 |
| `se(v)` | 有符号 0 阶（ZigZag 映射） | 运动矢量差值、QP 偏移 |
| `te(v)` | 截断 Exp-Golomb | 小范围值 |
| `ce(v)` | k 阶 | CABAC 相关参数 |

```cpp
// 位流写入器（辅助类）
class BitWriter {
    uint8_t* buf_;
    int bit_pos_ = 0;  // 当前写入位位置

public:
    explicit BitWriter(uint8_t* buf) : buf_(buf) {
        memset(buf, 0, 64);  // 预清零
    }

    void write_bits(uint32_t value, int num_bits) {
        for (int i = num_bits - 1; i >= 0; i--) {
            int byte_idx = bit_pos_ / 8;
            int bit_idx = 7 - (bit_pos_ % 8);
            if (value & (1u << i)) {
                buf_[byte_idx] |= (1u << bit_idx);
            }
            bit_pos_++;
        }
    }

    int bits_written() const { return bit_pos_; }
};

// 位流读取器
class BitReader {
    const uint8_t* buf_;
    int bit_pos_ = 0;

public:
    explicit BitReader(const uint8_t* buf) : buf_(buf) {}

    uint32_t read_bits(int num_bits) {
        uint32_t result = 0;
        for (int i = 0; i < num_bits; i++) {
            int byte_idx = bit_pos_ / 8;
            int bit_idx = 7 - (bit_pos_ % 8);
            result = (result << 1) | ((buf_[byte_idx] >> bit_idx) & 1);
            bit_pos_++;
        }
        return result;
    }

    int bits_read() const { return bit_pos_; }
};

// Exp-Golomb-0 编码
void encode_exp_golomb0(BitWriter& writer, uint32_t value) {
    uint32_t code_num = value + 1;
    int m = 0;
    uint32_t tmp = code_num;
    while (tmp > 1) { tmp >>= 1; m++; }  // m = floor(log2(code_num))

    // 写 m 个零
    for (int i = 0; i < m; i++) writer.write_bits(0, 1);
    // 写 code_num 的 (m+1) 位二进制
    writer.write_bits(code_num, m + 1);
}

// Exp-Golomb-0 解码
uint32_t decode_exp_golomb0(BitReader& reader) {
    int m = 0;
    while (reader.read_bits(1) == 0) m++;  // 数 leading zeros
    // 已经读了 1（分隔符），再读 m 位
    uint32_t info = (m > 0) ? reader.read_bits(m) : 0;
    return (1u << m) + info - 1;
}

// 有符号 Exp-Golomb（se(v)），ZigZag 映射
void encode_signed_exp_golomb(BitWriter& writer, int32_t value) {
    uint32_t mapped;
    if (value > 0) mapped = 2 * value - 1;
    else           mapped = -2 * value;
    encode_exp_golomb0(writer, mapped);
}

int32_t decode_signed_exp_golomb(BitReader& reader) {
    uint32_t code = decode_exp_golomb0(reader);
    if (code & 1) return static_cast<int32_t>((code + 1) / 2);
    else          return -static_cast<int32_t>(code / 2);
}
```

#### PrefixVarint vs Exp-Golomb 对比

| 特性 | PrefixVarint | Exp-Golomb |
|------|-------------|-----------|
| 对齐方式 | 字节对齐 | 位对齐 |
| 前缀含义 | leading zeros → 字节数 | leading zeros → bit 数 |
| 解码复杂度 | O(1) 查表 | O(M) 扫描零 |
| 空间效率 | 字节粒度，有填充浪费 | bit 精确，无浪费 |
| 随机访问 | 支持（已知长度） | 不支持（必须顺序解码） |
| 适用场景 | RPC、数据库、通用序列化 | 视频编码、熵编码 |
| 对硬件的要求 | 通用 CPU | 位操作密集，DSP/硬件编码器 |

---

## 四、批量变长编码

当需要编码大量整数（如倒排索引的文档 ID 列表）时，逐个 Varint 编码的开销不可忽视：每个字节都有分支判断，流水线不友好。批量编码方案将多个整数打包处理，分摊控制信息开销。

### 4.1 Group Varint（Google）

Google 在 2008 年的论文中提出了 Group Varint Encoding，核心思想：**4 个整数为一组，用 1 个控制字节描述每个整数的长度**。

```
控制字节布局：

  ┌──────┬──────┬──────┬──────┐
  │ 2bit │ 2bit │ 2bit │ 2bit │
  │ len0 │ len1 │ len2 │ len3 │
  └──────┴──────┴──────┴──────┘
     ↓      ↓      ↓      ↓
  len 值:  00=1字节  01=2字节  10=3字节  11=4字节

示例: 编码 [300, 1, 70000, 5]
  300   = 0x012C   → 2 字节 → len0 = 01
  1     = 0x01     → 1 字节 → len1 = 00
  70000 = 0x11170  → 3 字节 → len2 = 10
  5     = 0x05     → 1 字节 → len3 = 00

  控制字节: 01_00_10_00 = 0x48

  编码结果:
  [0x48] [0x2C,0x01] [0x01] [0x70,0x11,0x01] [0x05]
          └── 300 ──┘        └── 70000 ─────┘
                                                    共 8 字节
  对比 LEB128: 2+1+3+1 = 7 字节数据 + 无控制字节 = 7 字节
  Group Varint: 1 控制字节 + 2+1+3+1 = 8 字节
```

!!! note "空间略大，但解码快得多"
    Group Varint 的空间效率可能略差于 LEB128（因为控制字节额外开销），但解码速度可提升 2-4 倍。原因是：控制字节一次性确定了 4 个整数的长度，可以用查表法直接跳转到对应的解码路径，甚至用 SIMD 指令一次解码整组。

```cpp
// Group Varint 编码：4 个 uint32 为一组
int encode_group_varint(const uint32_t values[4], uint8_t* buf) {
    // 计算每个值需要的字节数（1~4）
    auto byte_len = [](uint32_t v) -> int {
        if (v < (1u << 8))  return 1;
        if (v < (1u << 16)) return 2;
        if (v < (1u << 24)) return 3;
        return 4;
    };

    int lens[4];
    uint8_t ctrl = 0;
    for (int i = 0; i < 4; i++) {
        lens[i] = byte_len(values[i]);
        ctrl |= (lens[i] - 1) << (6 - i * 2);
    }

    buf[0] = ctrl;
    int pos = 1;
    for (int i = 0; i < 4; i++) {
        // Little-Endian 写入
        memcpy(buf + pos, &values[i], lens[i]);
        pos += lens[i];
    }
    return pos;
}

// Group Varint 解码
int decode_group_varint(const uint8_t* buf, uint32_t values[4]) {
    uint8_t ctrl = buf[0];
    int pos = 1;
    for (int i = 0; i < 4; i++) {
        int len = ((ctrl >> (6 - i * 2)) & 0x03) + 1;
        values[i] = 0;
        memcpy(&values[i], buf + pos, len);  // Little-Endian 读取
        pos += len;
    }
    return pos;
}
```

#### 查表优化

实际生产实现中，控制字节只有 256 种可能，可以为每种组合预计算解码表：

```cpp
// 预计算解码表：每种控制字节对应的总数据长度和各值偏移
struct GroupVarintTable {
    uint8_t total_len;      // 数据部分总字节数
    uint8_t offsets[4];     // 每个值的起始偏移
    uint8_t lengths[4];     // 每个值的字节数
};

GroupVarintTable g_table[256];

void init_group_varint_table() {
    for (int ctrl = 0; ctrl < 256; ctrl++) {
        int offset = 0;
        for (int i = 0; i < 4; i++) {
            int len = ((ctrl >> (6 - i * 2)) & 0x03) + 1;
            g_table[ctrl].offsets[i] = offset;
            g_table[ctrl].lengths[i] = len;
            offset += len;
        }
        g_table[ctrl].total_len = offset;
    }
}

// 查表解码——无分支
int decode_group_varint_table(const uint8_t* buf, uint32_t values[4]) {
    uint8_t ctrl = buf[0];
    const auto& t = g_table[ctrl];
    const uint8_t* data = buf + 1;
    for (int i = 0; i < 4; i++) {
        values[i] = 0;
        memcpy(&values[i], data + t.offsets[i], t.lengths[i]);
    }
    return 1 + t.total_len;
}
```

### 4.2 Stream VByte

Stream VByte（2017, Lemire 等）将 Group Varint 的思想进一步发展：**控制字节和数据字节完全分离成两个独立的流**，从而实现更好的 SIMD 并行解码。

```
传统 Group Varint 布局（交错）：
  [ctrl][data0..][data1..][data2..][data3..][ctrl][data0..]...

Stream VByte 布局（分离）：
  控制流:  [ctrl0][ctrl1][ctrl2]...
  数据流:  [data0..data3..][data4..data7..]...

分离的优势：
  1. 控制流可以连续扫描，预取友好
  2. 数据流可以用 SIMD shuffle 指令批量解码
  3. 两个流可以独立压缩
```

```cpp
// Stream VByte 编码
struct StreamVByteEncoded {
    std::vector<uint8_t> ctrl;  // 控制流
    std::vector<uint8_t> data;  // 数据流
};

StreamVByteEncoded encode_stream_vbyte(const uint32_t* values, int count) {
    StreamVByteEncoded result;
    result.ctrl.reserve(count / 4 + 1);
    result.data.reserve(count * 2);  // 估计平均 2 字节

    for (int i = 0; i < count; i += 4) {
        uint8_t ctrl = 0;
        int group_size = std::min(4, count - i);
        for (int j = 0; j < group_size; j++) {
            uint32_t v = values[i + j];
            int len;
            if (v < (1u << 8))       len = 1;
            else if (v < (1u << 16)) len = 2;
            else if (v < (1u << 24)) len = 3;
            else                     len = 4;
            ctrl |= (len - 1) << (j * 2);
            // Little-Endian 写入数据流
            for (int k = 0; k < len; k++) {
                result.data.push_back(static_cast<uint8_t>(v >> (k * 8)));
            }
        }
        result.ctrl.push_back(ctrl);
    }
    return result;
}
```

!!! tip "SIMD 解码的关键"
    Stream VByte 的 SIMD 解码利用 `_mm_shuffle_epi8`（SSSE3 的 `pshufb` 指令）。通过控制字节查表得到一个 128-bit 的 shuffle mask，一条指令就能把变长的 4 个整数排列到正确的 4×32-bit 位置。在现代 CPU 上，Stream VByte 的解码吞吐可达 4GB/s 以上。

### 4.3 Masked VByte

Masked VByte 保留传统 VByte（LEB128）的编码格式，但用 SIMD 加速解码。思路是：

```
步骤 1: 用 SIMD 加载 16 字节
步骤 2: 提取每个字节的 MSB 形成 16-bit mask
步骤 3: 通过 mask 查表得到 shuffle 控制向量
步骤 4: 用 pshufb 重排字节，一次解码多个整数

优势: 编码格式与传统 VByte 兼容，仅解码侧加速
劣势: 查表和 shuffle 逻辑较复杂，实现难度高
```

---

## 五、位压缩（BitPacking）

当整数集合具有已知的值范围或分布特征时，可以在 bit 粒度上进行更紧凑的编码。

### 5.1 基本 BitPacking

如果一组整数的最大值需要 `b` 位表示，则每个整数都用恰好 `b` 位存储，紧密排列：

```
示例: 编码 [5, 3, 7, 1, 6, 2, 4, 0]，最大值 7，需要 b=3 位

  值:     5     3     7     1     6     2     4     0
  二进制: 101   011   111   001   110   010   100   000

  紧密排列（8 个值 × 3 位 = 24 位 = 3 字节）:
  字节0: [101][011][11]    = 0xAF (10101111)
  字节1: [1][001][110][0]  = 0x9C (10011100)
  字节2: [10][100][000]    = 0x90 (10010000)

  压缩比: 3 字节 / 32 字节(uint32×8) = 9.4%
```

```cpp
// BitPacking: 将 n 个整数用 b 位紧密编码
void bitpack(const uint32_t* values, int n, int b, uint8_t* out) {
    int bit_pos = 0;
    memset(out, 0, (n * b + 7) / 8);
    for (int i = 0; i < n; i++) {
        uint32_t v = values[i] & ((1u << b) - 1);  // 取低 b 位
        // 将 b 位写入 bit_pos 开始的位置
        int byte_idx = bit_pos / 8;
        int bit_offset = bit_pos % 8;
        // 可能跨越 1~2 个字节边界
        uint64_t wide = static_cast<uint64_t>(v) << bit_offset;
        // 写入（Little-Endian bit order）
        out[byte_idx]     |= static_cast<uint8_t>(wide);
        out[byte_idx + 1] |= static_cast<uint8_t>(wide >> 8);
        out[byte_idx + 2] |= static_cast<uint8_t>(wide >> 16);
        out[byte_idx + 3] |= static_cast<uint8_t>(wide >> 24);
        bit_pos += b;
    }
}

// BitUnpacking: 解码
void bitunpack(const uint8_t* in, int n, int b, uint32_t* values) {
    int bit_pos = 0;
    uint32_t mask = (1u << b) - 1;
    for (int i = 0; i < n; i++) {
        int byte_idx = bit_pos / 8;
        int bit_offset = bit_pos % 8;
        uint64_t wide;
        memcpy(&wide, in + byte_idx, sizeof(uint64_t));
        values[i] = static_cast<uint32_t>((wide >> bit_offset) & mask);
        bit_pos += b;
    }
}
```

!!! tip "SIMD BitPacking"
    实际系统（如 Apache Arrow、Parquet）中，BitPacking 通常以 32 或 128 个整数为一个 block，对每种位宽 `b`（1~32）生成专用的 pack/unpack 函数（可以用模板或代码生成），完全展开循环并利用 SIMD 指令。Daniel Lemire 的 [simdcomp](https://github.com/lemire/simdcomp) 库是参考实现。

### 5.2 FOR（Frame of Reference）

当整数集中在某个范围附近时（如时间戳），可以先减去一个基准值（reference），再对差值做 BitPacking：

```
原始数据:   [1001, 1003, 1005, 1002, 1007, 1004, 1006, 1000]
基准值:     min = 1000
差值:       [1, 3, 5, 2, 7, 4, 6, 0]
最大差值:   7 → 需要 3 位

编码结构:
  ┌───────────────────┬─────────────────────────────┐
  │ reference = 1000  │ 3-bit packed [1,3,5,2,7,4,6,0] │
  │   (4 字节)         │   (3 字节)                     │
  └───────────────────┴─────────────────────────────┘

  总计: 4 + 1(位宽) + 3 = 8 字节
  vs 原始: 8 × 4 = 32 字节
  压缩比: 25%
```

```cpp
struct ForBlock {
    uint32_t reference;   // 基准值
    uint8_t bit_width;    // 每个差值的位宽
    uint8_t count;        // 元素个数
    uint8_t data[];       // 位压缩的差值
};

// FOR 编码
int encode_for(const uint32_t* values, int n, uint8_t* out) {
    // 找最小值和最大差值
    uint32_t min_val = *std::min_element(values, values + n);
    uint32_t max_diff = 0;
    for (int i = 0; i < n; i++) {
        max_diff = std::max(max_diff, values[i] - min_val);
    }

    int b = (max_diff == 0) ? 0 : (32 - __builtin_clz(max_diff));

    // 写头部
    auto* block = reinterpret_cast<ForBlock*>(out);
    block->reference = min_val;
    block->bit_width = b;
    block->count = n;

    // 计算差值并位压缩
    std::vector<uint32_t> diffs(n);
    for (int i = 0; i < n; i++) {
        diffs[i] = values[i] - min_val;
    }
    bitpack(diffs.data(), n, b, block->data);

    return sizeof(ForBlock) + (n * b + 7) / 8;
}
```

### 5.3 PFOR（Patched Frame of Reference）

FOR 的弱点：如果数据中有少量异常大值（outlier），会抬高整个 block 的位宽。PFOR 的解决方案：**用较小的位宽编码大多数值，异常值单独存储**。

```
示例:
  数据: [3, 1, 2, 500, 4, 1, 3, 2, 1000, 5]
  去掉异常值 500 和 1000 后，其余值最大为 5，需要 3 位

  PFOR 编码结构:
  ┌──────────┬───────────────────┬──────────────────┐
  │ 头部      │ 主数据 (3-bit)     │ 异常值列表        │
  │ b=3      │ [3,1,2,0,4,1,3,2, │ [(3,500),         │
  │ nexc=2   │  0,5]              │  (8,1000)]        │
  └──────────┴───────────────────┴──────────────────┘
                  ↑ 异常位置填 0          ↑ (位置, 原始值)

  主数据: 10 × 3 = 30 bits ≈ 4 字节
  异常值: 2 × (1 + 4) = 10 字节
  总计: ~16 字节 vs 原始 40 字节 vs 纯FOR需要 10-bit 宽 = ~15 字节
```

```cpp
struct PforBlock {
    uint32_t reference;
    uint8_t bit_width;        // 主位宽
    uint8_t count;            // 总元素数
    uint8_t exception_count;  // 异常值个数
    // 后接: packed_data[] + exception_positions[] + exception_values[]
};

// PFOR 编码（简化版）
int encode_pfor(const uint32_t* values, int n, uint8_t* out,
                int target_bits = 0) {
    uint32_t min_val = *std::min_element(values, values + n);

    // 计算差值
    std::vector<uint32_t> diffs(n);
    for (int i = 0; i < n; i++) diffs[i] = values[i] - min_val;

    // 确定最优位宽：选择覆盖 90%~95% 值的位宽
    if (target_bits == 0) {
        std::vector<uint32_t> sorted_diffs(diffs);
        std::sort(sorted_diffs.begin(), sorted_diffs.end());
        uint32_t threshold = sorted_diffs[n * 9 / 10];  // 90th percentile
        target_bits = (threshold == 0) ? 1 : (32 - __builtin_clz(threshold));
    }

    uint32_t max_normal = (1u << target_bits) - 1;

    // 分离正常值和异常值
    std::vector<uint8_t> exc_positions;
    std::vector<uint32_t> exc_values;
    std::vector<uint32_t> packed_diffs(n);
    for (int i = 0; i < n; i++) {
        if (diffs[i] > max_normal) {
            packed_diffs[i] = 0;  // 占位
            exc_positions.push_back(i);
            exc_values.push_back(diffs[i]);
        } else {
            packed_diffs[i] = diffs[i];
        }
    }

    // 写入
    auto* block = reinterpret_cast<PforBlock*>(out);
    block->reference = min_val;
    block->bit_width = target_bits;
    block->count = n;
    block->exception_count = exc_positions.size();

    int pos = sizeof(PforBlock);
    // 主数据
    int packed_bytes = (n * target_bits + 7) / 8;
    bitpack(packed_diffs.data(), n, target_bits, out + pos);
    pos += packed_bytes;
    // 异常位置
    memcpy(out + pos, exc_positions.data(), exc_positions.size());
    pos += exc_positions.size();
    // 异常值
    memcpy(out + pos, exc_values.data(), exc_values.size() * 4);
    pos += exc_values.size() * 4;

    return pos;
}
```

### 5.4 Delta Encoding + BitPacking

对于有序递增的整数序列（如排序后的文档 ID），先做差分（delta），再对差值做位压缩，效果极佳：

```
原始数据（有序文档 ID）:
  [100, 102, 105, 106, 110, 115, 116, 120]

Delta 编码:
  第一个值: 100
  差值: [2, 3, 1, 4, 5, 1, 4]
  最大差值 5 → 需要 3 位

  编码结构:
  ┌────────────┬──────────────────────┐
  │ base = 100 │ 3-bit packed deltas  │
  │ (4 字节)    │ (3 字节)              │
  └────────────┴──────────────────────┘

  总计: ~8 字节 vs 原始 32 字节
```

```cpp
// Delta + BitPack 编码
struct DeltaBlock {
    uint32_t base;       // 第一个值
    uint8_t bit_width;   // delta 的位宽
    uint8_t count;       // 元素个数
    uint8_t data[];      // 位压缩的 delta 值
};

int encode_delta_bitpack(const uint32_t* values, int n, uint8_t* out) {
    auto* block = reinterpret_cast<DeltaBlock*>(out);
    block->base = values[0];
    block->count = n;

    // 计算 delta
    std::vector<uint32_t> deltas(n - 1);
    uint32_t max_delta = 0;
    for (int i = 0; i < n - 1; i++) {
        deltas[i] = values[i + 1] - values[i];
        max_delta = std::max(max_delta, deltas[i]);
    }

    int b = (max_delta == 0) ? 0 : (32 - __builtin_clz(max_delta));
    block->bit_width = b;

    bitpack(deltas.data(), n - 1, b, block->data);
    return sizeof(DeltaBlock) + ((n - 1) * b + 7) / 8;
}

// Delta + BitPack 解码（支持 seek 到第 k 个元素）
void decode_delta_bitpack(const uint8_t* in, uint32_t* values) {
    auto* block = reinterpret_cast<const DeltaBlock*>(in);
    values[0] = block->base;

    std::vector<uint32_t> deltas(block->count - 1);
    bitunpack(block->data, block->count - 1, block->bit_width, deltas.data());

    // 前缀和还原
    for (int i = 0; i < block->count - 1; i++) {
        values[i + 1] = values[i] + deltas[i];
    }
}
```

!!! note "Delta-of-Delta"
    对于近似等差数列（如均匀采样的时间戳），可以做两次差分（delta-of-delta），进一步减小值范围。InfluxDB 的 TSM 引擎就使用了这种策略。

---

## 六、高级批量编码

### 6.1 Simple8b

Simple8b（Anh & Moffat, 2010）是一种自适应的整数压缩方案，将多个小整数打包进一个 64-bit 字中。核心思想：**用 4-bit 选择器描述打包模式，剩余 60 bit 存储数据**。

```
64-bit 字的布局:
  ┌────────┬──────────────────────────────────────────────────────────┐
  │ 4-bit  │                    60-bit 数据区                         │
  │selector│                                                          │
  └────────┴──────────────────────────────────────────────────────────┘

selector 值决定打包模式:
```

| Selector | 每个值的 bit 数 | 打包个数 | 可表示最大值 |
|----------|---------------|---------|-------------|
| 0 | 0 | 240 | 0（全零的 run） |
| 1 | 1 | 60 | 1 |
| 2 | 2 | 30 | 3 |
| 3 | 3 | 20 | 7 |
| 4 | 4 | 15 | 15 |
| 5 | 5 | 12 | 31 |
| 6 | 6 | 10 | 63 |
| 7 | 7 | 8 | 127 |
| 8 | 8 | 7 | 255 |
| 9 | 10 | 6 | 1,023 |
| 10 | 12 | 5 | 4,095 |
| 11 | 15 | 4 | 32,767 |
| 12 | 20 | 3 | 1,048,575 |
| 13 | 30 | 2 | 1,073,741,823 |
| 14 | 60 | 1 | 2^60 - 1 |

!!! note "Simple8b 的命名"
    "Simple" 系列编码因其实现简单而得名。"8b" 表示使用 8 字节（64-bit）的字。类似的还有 Simple-9（32-bit 字，4-bit 选择器 + 28-bit 数据）和 Simple-16（32-bit 字，更精细的选择器）。

```cpp
// Simple8b 选择器表
struct Simple8bMode {
    int bits_per_value;  // 每个值的位数
    int count;           // 可打包的值个数
};

static const Simple8bMode kModes[15] = {
    {0, 240}, {1, 60}, {2, 30}, {3, 20}, {4, 15},
    {5, 12},  {6, 10}, {7, 8},  {8, 7},  {10, 6},
    {12, 5},  {15, 4}, {20, 3}, {30, 2}, {60, 1},
};

// Simple8b 编码一个 64-bit 字
// 返回消耗的输入值个数
int encode_simple8b_word(const uint64_t* values, int n, uint64_t* word) {
    // 贪心选择：找能打包最多值的 selector
    for (int sel = 0; sel < 15; sel++) {
        int count = std::min(kModes[sel].count, n);
        int bits = kModes[sel].bits_per_value;
        uint64_t max_val = (bits == 0) ? 0 : ((1ULL << bits) - 1);

        // 检查前 count 个值是否都能用 bits 位表示
        bool fits = true;
        for (int i = 0; i < count; i++) {
            if (values[i] > max_val) { fits = false; break; }
        }
        if (!fits) continue;

        // 打包
        *word = static_cast<uint64_t>(sel) << 60;
        for (int i = 0; i < count; i++) {
            *word |= values[i] << (i * bits);
        }
        return count;
    }
    return 0;  // 不应该到这里
}

// Simple8b 解码一个 64-bit 字
int decode_simple8b_word(uint64_t word, uint64_t* values) {
    int sel = static_cast<int>(word >> 60);
    int count = kModes[sel].count;
    int bits = kModes[sel].bits_per_value;

    if (bits == 0) {
        // 全零
        for (int i = 0; i < count; i++) values[i] = 0;
    } else {
        uint64_t mask = (1ULL << bits) - 1;
        for (int i = 0; i < count; i++) {
            values[i] = (word >> (i * bits)) & mask;
        }
    }
    return count;
}
```

#### Simple8b 的选择器策略

编码时需要选择最优的 selector。上面的实现使用贪心策略（选能打包最多值的），但也有其他选择：

```
贪心策略（从 selector 0 开始尝试）:
  优点: 每个 word 打包尽可能多的值 → 更好的压缩率
  缺点: O(n) 扫描，编码稍慢

从大 selector 开始（先试 selector 14）:
  优点: 更快找到匹配
  缺点: 可能浪费空间

实际系统通常使用贪心策略，因为编码是一次性的，
而解码需要反复执行，Simple8b 的解码始终是 O(1) per word。
```

### 6.2 Roaring Bitmap 中的编码策略

Roaring Bitmap 是现代系统中最常用的压缩位图实现（Lucene、Spark、Redis 等都在使用）。它对整数集合的编码策略值得单独分析：

```
Roaring 将 32-bit 整数空间按高 16 位分桶（chunk），
每个桶内存储低 16 位的集合，根据基数选择容器类型：

               整数 0xHHHHLLLL
                   /        \
              高 16 位       低 16 位
              (桶索引)      (容器内值)
                 |              |
                 v              v
           ┌──────────┬─────────────────┐
           │ chunk_id │   container      │
           │  0xHHHH  │   (三选一)        │
           └──────────┴─────────────────┘
```

三种容器类型及其编码：

| 容器类型 | 适用基数 | 存储方式 | 空间 |
|---------|---------|---------|------|
| Array | < 4096 | 排序的 uint16 数组 | 2n 字节 |
| Bitmap | >= 4096 | 65536-bit 位图 | 8192 字节固定 |
| Run | 连续区间多 | (start, length) 对 | 4r 字节（r 个 run） |

```
基数与容器选择的交叉点：

空间
 ↑
 │          ╱ Array (2n)
 │         ╱
 │        ╱
8192 ────╱──────────────── Bitmap (固定 8192)
 │      ╱
 │     ╱
 │    ╱
 │   ╱
 └───┼───────────────────→ 基数 n
     4096

Array: 基数 < 4096 时，2×4096 = 8192，恰好等于 Bitmap
切换点: n = 4096
Run: 当数据包含长连续区间时最优（独立判断）
```

!!! tip "Roaring 的动态切换"
    Roaring 在运行时根据每个容器的实际基数动态选择类型。插入/删除操作后，如果基数跨过阈值，容器会自动转换类型。这种自适应策略是 Roaring 性能优异的关键原因。

---

## 七、字节对齐 vs 位对齐

前面介绍的编码方案在对齐方式上分为两大阵营，这个选择对性能影响深远：

### 7.1 对比分析

```
字节对齐 (Byte-Aligned)：
  ┌────────┬────────┬────────┐
  │  byte  │  byte  │  byte  │   每个值占整数个字节
  └────────┴────────┴────────┘
  ↑ 任意值都可以用 memcpy/指针直接读取

位对齐 (Bit-Aligned)：
  ┌───┬─────┬───┬────┬──┐
  │3b │ 5b  │3b │ 4b │2b│   值跨越字节边界
  └───┴─────┴───┴────┴──┘
  ↑ 需要 shift + mask 操作提取
```

| 维度 | 字节对齐 | 位对齐 |
|------|---------|--------|
| 压缩率 | 较差（浪费 padding bits） | 最优（bit 精确） |
| 解码速度 | 快（memcpy / 指针读取） | 较慢（shift + mask） |
| SIMD 友好度 | 高（shuffle 操作天然字节粒度） | 中（需要额外移位） |
| 随机访问 | 容易（偏移 = index × byte_len） | 需要计算 bit 偏移 |
| 实现复杂度 | 低 | 中~高 |
| 典型代表 | Group Varint, Stream VByte | BitPacking, Simple8b |

### 7.2 工程上的选择原则

```
选择字节对齐（速度优先）:
  ✓ 解码在关键路径上（如在线查询）
  ✓ 数据量不是瓶颈（内存充足）
  ✓ 需要支持随机访问
  ✓ 实现需要简单可维护

选择位对齐（空间优先）:
  ✓ 数据量巨大（TB 级存储）
  ✓ I/O 带宽是瓶颈
  ✓ 批量顺序扫描为主
  ✓ 可以投入 SIMD 优化
```

---

## 八、综合性能对比

以下是各编码方案在典型场景下的性能特征总结：

### 8.1 逐个编码对比

| 编码方案 | 压缩率 | 编码速度 | 解码速度 | 最大开销 | 适用场景 |
|---------|--------|---------|---------|---------|---------|
| 定长 32-bit | 1:1 | 极快 | 极快 | 无 | 值域均匀分布 |
| LEB128 | 好 | 快 | 中 | +25%（10B/8B） | 通用序列化 |
| ZigZag+Varint | 好 | 快 | 中 | 同上 | 有符号小整数 |
| SQLite Varint | 好 | 快 | 快 | +12.5%（9B/8B） | 数据库存储 |
| PrefixVarint | 好 | 中 | 快 | +12.5% | 需要快速长度判定 |
| Exp-Golomb | 最优 | 慢 | 慢 | bit 开销 | 视频编码 |
| VLQ | 好 | 快 | 中 | +25% | MIDI, Source Map |

### 8.2 批量编码对比

| 编码方案 | 压缩率 | 编码速度 | 解码速度 | SIMD | 适用场景 |
|---------|--------|---------|---------|------|---------|
| Group Varint | 中 | 快 | 很快 | 部分 | 通用整数列表 |
| Stream VByte | 中 | 快 | 极快 | 是 | 高吞吐解码 |
| BitPacking | 好 | 快 | 很快 | 是 | 已知位宽 |
| FOR | 很好 | 快 | 很快 | 是 | 范围集中的整数 |
| PFOR | 很好 | 中 | 快 | 是 | 有少量异常值 |
| Delta+BitPack | 极好 | 快 | 快 | 是 | 有序递增序列 |
| Simple8b | 很好 | 中 | 快 | 部分 | 大量小整数 |

### 8.3 典型吞吐量参考

以下数据基于现代 x86-64 CPU（~3GHz），单线程，仅供量级参考：

```
解码吞吐量（百万整数/秒）:

                    标量       SIMD
                ─────────────────────
LEB128            ~400         N/A
Group Varint      ~800        ~1500
Stream VByte      ~600        ~4000
BitPacking        ~1200       ~5000
Simple8b          ~1000        N/A
FOR               ~1200       ~5000
Delta+BitPack     ~1000       ~4000

说明: SIMD 实现通常使用 SSE4/AVX2 指令集
```

---

## 九、实际系统选型

### 9.1 各系统的编码选择

| 系统 | 使用的编码 | 选型理由 |
|------|-----------|---------|
| **Protocol Buffers** | LEB128 + ZigZag | 通用性、实现简单、跨语言 |
| **Apache Thrift** | LEB128 (Compact Protocol) | 与 Protobuf 类似的考量 |
| **FlatBuffers** | 定长 + 可选 Varint | 零拷贝优先，避免变长编码开销 |
| **Cap'n Proto** | 定长（不压缩） | 极致零拷贝，传输即内存布局 |
| **Lucene 倒排索引** | PFOR + VInt | 文档 ID 列表用 PFOR，元数据用 VInt |
| **LevelDB / RocksDB** | LEB128 + Prefix 压缩 | Block 内 key 共享前缀 + Varint 长度 |
| **Apache Parquet** | Delta + RLE + BitPacking | 列式存储，同列数据相似性高 |
| **Apache Arrow** | BitPacking + Dictionary | 内存分析格式，SIMD 优化 |
| **ClickHouse** | Delta + DoubleDelta + Gorilla | 时序数据优化 |
| **InfluxDB (TSM)** | Delta-of-Delta + Simple8b | 时间戳用 DeltaDelta，值用 Simple8b |
| **H.264 / H.265** | Exp-Golomb + CABAC | bit 级效率，硬件解码器支持 |
| **WebAssembly** | LEB128 | 标准规范要求，流式解析 |
| **Roaring Bitmap** | Array / Bitmap / Run | 自适应，根据基数动态选择 |
| **Source Map v3** | Base64 VLQ | 文本传输友好，JS 生态兼容 |

### 9.2 选型决策树

```
需要编码整数？
│
├─ 单个整数（RPC/序列化场景）
│  ├─ 需要跨语言兼容 → LEB128 (Protobuf Varint)
│  ├─ 有符号且常为小值 → ZigZag + LEB128
│  ├─ 需要快速长度判定 → PrefixVarint / SQLite Varint
│  └─ bit 级效率至上 → Exp-Golomb
│
├─ 大量整数（批量场景）
│  ├─ 无序、值域宽
│  │  ├─ 解码速度优先 → Stream VByte (SIMD)
│  │  └─ 压缩率优先 → Simple8b
│  │
│  ├─ 有序递增（文档ID、时间戳）
│  │  ├─ 等差近似 → Delta-of-Delta + BitPacking
│  │  └─ 非等差 → Delta + FOR/PFOR
│  │
│  ├─ 值域集中（某范围附近）
│  │  ├─ 无异常值 → FOR + BitPacking
│  │  └─ 有少量异常值 → PFOR
│  │
│  └─ 集合运算（交并差）
│     └─ Roaring Bitmap
│
└─ 视频/信号编码
   └─ Exp-Golomb + 算术编码
```

### 9.3 工程实践建议

!!! tip "先 profile，再选型"
    不要过早优化编码方案。在多数应用中，I/O 延迟远大于编解码 CPU 开销。建议：(1) 先用 LEB128 快速实现；(2) 用 profiler 确认编解码是否真的是瓶颈；(3) 如果是，按数据特征选择合适的批量编码。

!!! warning "分块大小的选择"
    批量编码都需要选择 block size（每块多少个整数）。常见选择：
    
    - **128 个整数/块**：SIMD 友好（128-bit SSE 寄存器 × 32-bit = 4 个整数，32 次迭代）
    - **256 个整数/块**：AVX2 友好
    - **1024 个整数/块**：更好的压缩率，但内存局部性下降
    
    Lucene 使用 128，Parquet 使用 miniblock 概念（默认 128），InfluxDB 使用 1000。

!!! note "混合编码"
    生产系统很少只用一种编码。典型做法是：
    
    - **元数据层**（长度、偏移量、少量标记）：LEB128 / PrefixVarint
    - **数据层**（大量整数列表）：Delta + BitPacking / PFOR / Simple8b
    - **索引层**（跳表、块索引）：定长编码（方便随机访问）
    
    不同层使用不同编码，各取所长。
