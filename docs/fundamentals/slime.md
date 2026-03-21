# Vespa SLIME 编解码：深入二进制格式与源码实现

## 一、SLIME 是什么

SLIME（**S**chema-**L**ess **I**nterface/**M**odel/**E**xchange）是 Vespa 搜索引擎内部使用的二进制序列化协议。它的设计目标是在**无 Schema** 的前提下，提供一种紧凑、高效的 JSON-like 数据交换格式。

与 Protobuf/Avro 等需要预定义 Schema 的协议不同，SLIME 像 JSON 一样灵活，但采用二进制编码以获得更高的空间效率和编解码速度。

### 1.1 设计目标

- **Schema-less yet strictly typed**：不需要预定义结构，但每个值都有明确的类型
- **Simple yet efficient binary format**：编码规则简单，但产出紧凑
- **Simple application interaction**：通过 Cursor 接口操作，屏蔽内部复杂性

### 1.2 应用场景

在 Vespa 系统中，SLIME 被广泛用于：

- **Interface**：RPC 调用、网络消息
- **Model**：状态、指标、配置、Schema 描述
- **Exchange**：文档、记录的序列化传输

---

## 二、数据模型

### 2.1 整体结构

一个 SLIME 结构由两部分组成：

```
Slime = Symbol Table + Value
```

**符号表（Symbol Table）** 集中存储所有字段名，Value 中的对象通过整数 ID 引用符号表中的名称。这种分离设计在处理嵌套或重复结构时能显著节省空间——同一个字段名无论出现多少次，都只存储一份。

### 2.2 类型系统

SLIME 定义了 8 种类型，按复杂度递增编号：

| ID | 类型 | 说明 |
|----|------|------|
| 0 | NIX | 空类型，无值 |
| 1 | BOOL | 布尔值 |
| 2 | LONG | 有符号整数（64 位） |
| 3 | DOUBLE | 双精度浮点数 |
| 4 | STRING | 字符串 |
| 5 | DATA | 原始字节数据 |
| 6 | ARRAY | 有序数组，元素类型可不同 |
| 7 | OBJECT | 键值对集合，键为符号 ID |

对应的 C++ 类型定义（`type.h`）：

```cpp
using NIX    = TypeType<0>;
using BOOL   = TypeType<1>;
using LONG   = TypeType<2>;
using DOUBLE = TypeType<3>;
using STRING = TypeType<4>;
using DATA   = TypeType<5>;
using ARRAY  = TypeType<6>;
using OBJECT = TypeType<7>;
```

Java 中则使用枚举：

```java
public enum Type {
    NIX(0), BOOL(1), LONG(2), DOUBLE(3),
    STRING(4), DATA(5), ARRAY(6), OBJECT(7);
}
```

### 2.3 与 JSON 的对应关系

```
JSON                    SLIME
────                    ─────
null                    NIX
true / false            BOOL
42, -7                  LONG
3.14                    DOUBLE
"hello"                 STRING
（无对应）               DATA（原始字节）
[1, "a", true]          ARRAY
{"key": "value"}        OBJECT + Symbol Table
```

SLIME 比 JSON 多了一个 `DATA` 类型（原始二进制），也多了 `NIX`（JSON 的 `null` 语义更宽泛）。数组元素不要求类型一致，这一点与 JSON 相同。

---

## 三、二进制编码基础构件

SLIME 的二进制格式由三种基本构件组合而成。

### 3.1 type_meta 字节

单个字节，同时编码**类型**和**元数据**：

```
MSB                 LSB
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 7 │ 6 │ 5 │ 4 │ 3 │ 2 │ 1 │ 0 │
├───┴───┴───┴───┴───┼───┴───┴───┤
│    meta (5 bit)    │ type (3bit)│
└───────────────────┴───────────┘
```

- **低 3 位**：类型编号（0-7）
- **高 5 位**：元数据（0-31），含义取决于具体类型

编码函数（C++）：

```cpp
inline char encode_type_and_meta(uint32_t type, uint32_t meta) {
    return (((type & 0x7) | (meta << 3)) & 0xff);
}

inline uint32_t decode_type(uint32_t type_and_meta) {
    return (type_and_meta & 0x7);
}

inline uint32_t decode_meta(uint32_t type_and_meta) {
    return ((type_and_meta >> 3) & 0x1f);
}
```

Java 版本完全一致：

```java
static byte encode_type_and_meta(int type, int meta) {
    return (byte) ((meta << 3) | (type & 0x7));
}
```

### 3.2 cmpr_ulong（压缩无符号整数）

自定界的变长无符号整数编码。每个字节的**最高位是继续标志**，低 7 位是数据位。字节序为小端（低位在前）。

```
值 300 的编码过程：
300 = 0b100101100

拆分为 7-bit 组（低位在前）：
  第 1 组: 0101100  (44)
  第 2 组: 0000010  (2)

加继续标志位（非最后一个字节 MSB=1）：
  字节 1: 1_0101100 = 0xAC  (继续)
  字节 2: 0_0000010 = 0x02  (终止)

编码结果: [0xAC, 0x02]  →  2 字节
```

C++ 实现（`binary_format.h`）：

```cpp
inline uint32_t encode_cmpr_ulong(char *out, uint64_t value) {
    char *pos = out;
    char next = (value & 0x7f);
    value >>= 7;
    while (value != 0) {
        *pos++ = (next | 0x80);   // 设置继续标志
        next = (value & 0x7f);
        value >>= 7;
    }
    *pos++ = next;                // 最后一个字节无继续标志
    return (pos - out);
}

inline uint64_t read_cmpr_ulong(InputReader &in) {
    uint64_t next = in.read();
    uint64_t value = (next & 0x7f);
    int shift = 7;
    while ((next & 0x80) != 0) {
        next = in.read();
        value |= ((next & 0x7f) << shift);
        shift += 7;
    }
    return value;
}
```

!!! note "与 Protobuf Varint 的对比"
    SLIME 的 `cmpr_ulong` 与 Protobuf 的 Varint 编码几乎完全相同——都是 7-bit 分组、MSB 继续标志、小端序。这不是巧合，而是该编码方式在紧凑性和解码效率之间的最佳平衡。

### 3.3 size(meta) — 尺寸内联优化

STRING、DATA、ARRAY、OBJECT 类型都需要存储一个"大小"值。SLIME 使用 type_meta 字节的 meta 字段来**内联**小尺寸值，避免额外开销：

```
meta == 0  →  大小存储在紧随其后的 cmpr_ulong 中
meta != 0  →  大小 = meta - 1（范围 [0, 30]）
```

这意味着长度不超过 30 的字符串、数组、对象，其大小信息**不占用任何额外字节**——直接编码在 type_meta 字节的高 5 位中。

C++ 解码：

```cpp
inline uint64_t read_size(InputReader &in, uint32_t meta) {
    return (meta == 0) ? read_cmpr_ulong(in) : (meta - 1);
}
```

C++ 编码：

```cpp
void write_type_and_size_impl(OutputWriter &out, uint32_t type, uint64_t size) {
    char *start = out.reserve(11);
    char *pos = start;
    if (size <= 30) {
        *pos++ = encode_type_and_meta(type, size + 1);  // 内联
    } else {
        *pos++ = encode_type_and_meta(type, 0);          // meta=0 表示外置
        pos += encode_cmpr_ulong(pos, size);
    }
    out.commit(pos - start);
}
```

---

## 四、各类型编码详解

### 4.1 NIX（类型 0）

NIX 是最简单的类型——没有可能的值，编码只有 1 个字节：

```
┌──────────┐
│ 00000_000 │  type=0, meta=0
└──────────┘
   1 byte
```

```cpp
void encodeNix() {
    out.write(NIX::ID);  // 0x00
}
```

### 4.2 BOOL（类型 1）

布尔值直接存储在 meta 字段中，同样只需 1 个字节：

```
false:  ┌──────────┐
        │ 00000_001 │  type=1, meta=0
        └──────────┘

true:   ┌──────────┐
        │ 00001_001 │  type=1, meta=1
        └──────────┘
```

```cpp
void encodeBool(bool value) {
    out.write(encode_type_and_meta(BOOL::ID, value ? 1 : 0));
}

// 解码
Cursor &decodeBool(const Inserter &inserter, uint32_t meta) {
    return inserter.insertBool(meta != 0);
}
```

### 4.3 LONG（类型 2）— Zigzag + 截断

整数编码是 SLIME 最精巧的设计之一，分三步完成：

**Step 1：Zigzag 编码**

将有符号整数转换为无符号整数，使绝对值小的负数也能用少量字节表示：

```
原值         Zigzag 结果
   0    →    0
  -1    →    1
   1    →    2
  -2    →    3
   2    →    4
 ...
```

```cpp
inline uint64_t encode_zigzag(int64_t x) {
    return ((x << 1) ^ (x >> 63));  // 算术右移
}

inline int64_t decode_zigzag(uint64_t x) {
    return ((x >> 1) ^ (-(x & 0x1)));
}
```

**Step 2：去除高位零字节**

Zigzag 编码后的值按小端序排列，只保留非零字节：

```
值 300 → zigzag → 600 = 0x0000000000000258
只保留非零字节: [0x58, 0x02]  →  2 字节
```

**Step 3：meta 记录字节数**

存储的字节数写入 type_meta 的 meta 字段。

```cpp
template <bool top>
void write_type_and_bytes(OutputWriter &out, uint32_t type, uint64_t bits) {
    char *start = out.reserve(9);  // 最多 1 + 8 字节
    char *pos = start + 1;        // 跳过 type_meta 字节
    while (bits != 0) {
        if (top) {
            *pos++ = (bits >> 56);  // 大端：高字节在前
            bits <<= 8;
        } else {
            *pos++ = (bits & 0xff); // 小端：低字节在前
            bits >>= 8;
        }
    }
    // 回填 type_meta 字节
    *start = encode_type_and_meta(type, pos - start - 1);
    out.commit(pos - start);
}
```

LONG 使用 `top=false`（小端序）调用：

```cpp
void encodeLong(int64_t value) {
    write_type_and_bytes<false>(out, LONG::ID, encode_zigzag(value));
}
```

**编码示例**：

```
值 42 的编码：
  zigzag(42) = 84 = 0x54
  非零字节: [0x54]  →  1 字节
  type_meta: type=2, meta=1  →  (1 << 3) | 2 = 0x0A

  编码结果: [0x0A, 0x54]  →  2 字节

值 0 的编码：
  zigzag(0) = 0
  无非零字节
  type_meta: type=2, meta=0  →  0x02

  编码结果: [0x02]  →  仅 1 字节！

值 -1 的编码：
  zigzag(-1) = 1 = 0x01
  非零字节: [0x01]  →  1 字节
  type_meta: type=2, meta=1  →  0x0A

  编码结果: [0x0A, 0x01]  →  2 字节
```

!!! tip "零值优化"
    整数 0 和浮点数 0.0 都只需要 **1 个字节**（type_meta 字节本身，meta=0 表示 0 个数据字节）。这在实际数据中非常常见，对稀疏数据尤其有效。

### 4.4 DOUBLE（类型 3）— 位翻转 + 截断

浮点数的编码与整数类似，但有一个巧妙的**字节反转**步骤：

**Step 1：获取 IEEE 754 位模式**

```cpp
inline uint64_t encode_double(double x) {
    union { uint64_t UINT64; double DOUBLE; } val;
    val.DOUBLE = x;
    return val.UINT64;
}
```

**Step 2：字节反转**

IEEE 754 双精度浮点数的内存布局：

```
┌──────┬──────────┬──────────────────────────────────────────┐
│ sign │ exponent │              mantissa                     │
│ 1bit │  11bit   │              52bit                        │
└──────┴──────────┴──────────────────────────────────────────┘
  MSB (byte 7)                                    LSB (byte 0)
```

对于常见的小浮点数（如 1.0、3.14），指数部分相对较小，尾数的高位多为零。但这些零位在高字节端。**字节反转后**，零字节移到了低端，就可以像整数一样截断高位零字节。

**Step 3：大端存储非零字节**

与 LONG 的小端不同，DOUBLE 使用 `top=true`（大端序），从高位开始存储：

```cpp
void encodeDouble(double value) {
    write_type_and_bytes<true>(out, DOUBLE::ID, encode_double(value));
}
```

**编码示例**：

```
值 1.0 的编码：
  IEEE 754: 0x3FF0000000000000
  字节反转: 0x000000000000F03F
  去除高位零: 从低位取非零字节 → [0xF0, 0x3F]
  但因为是 top=true（大端），实际过程是：
    原始 bits = 0x3FF0000000000000
    每次取 bits >> 56 → 0x3F, 然后 bits <<= 8
    → [0x3F, 0xF0]
    后续 bits 全为 0，停止
  meta = 2（2 个字节）
  type_meta: type=3, meta=2 → (2 << 3) | 3 = 0x13

  编码结果: [0x13, 0x3F, 0xF0]  →  3 字节

值 0.0 的编码：
  IEEE 754: 0x0000000000000000
  无非零字节
  type_meta: type=3, meta=0 → 0x03

  编码结果: [0x03]  →  仅 1 字节！
```

!!! info "为什么 LONG 用小端而 DOUBLE 用大端？"
    这是 SLIME 的一个精妙设计。对于整数，zigzag 编码后有效位集中在**低字节**，所以从低位（小端）开始存储，遇到高位零就停止。对于浮点数，IEEE 754 的有效位（符号位 + 指数）集中在**高字节**，所以从高位（大端）开始存储，低位的零尾数被截断。两种策略都是为了最大化截断效果。

### 4.5 STRING（类型 4）和 DATA（类型 5）

这两种类型的编码方式完全相同：`size(meta)` + 原始字节。

```
字符串 "hi" 的编码：
  长度 = 2，可以内联（2 ≤ 30）
  type_meta: type=4, meta=2+1=3  →  (3 << 3) | 4 = 0x1C
  内容: [0x68, 0x69]

  编码结果: [0x1C, 0x68, 0x69]  →  3 字节
```

```cpp
void encodeString(const Memory &memory) {
    write_type_and_size_impl(out, STRING::ID, memory.size);
    out.write(memory.data, memory.size);
}
```

对比 JSON 中 `"hi"` 需要 4 字节（含两个引号），SLIME 只需 3 字节。

### 4.6 ARRAY（类型 6）

数组编码为 `size(meta)` + N 个连续编码的 value：

```
数组 [42, true] 的编码：
  元素数 = 2，内联
  type_meta: type=6, meta=2+1=3  →  (3 << 3) | 6 = 0x1E

  元素 1: LONG 42  →  [0x0A, 0x54]
  元素 2: BOOL true →  [0x09]

  编码结果: [0x1E, 0x0A, 0x54, 0x09]  →  4 字节
```

C++ 编码（通过遍历器模式）：

```cpp
void encodeArray(const Inspector &inspector) {
    write_type_and_size_impl(out, ARRAY::ID, inspector.children());
    inspector.traverse(array_traverser);  // 递归编码每个元素
}

// 遍历器回调
void BinaryEncoder::entry(size_t, const Inspector &inspector) {
    encodeValue(inspector);
}
```

解码：

```cpp
Cursor &decodeArray(const Inserter &inserter, uint32_t meta) {
    Cursor &cursor = inserter.insertArray();
    ArrayInserter childInserter(cursor);
    uint64_t size = read_size(in, meta);
    for (size_t i = 0; i < size; ++i) {
        decodeValue(childInserter);
    }
    return cursor;
}
```

### 4.7 OBJECT（类型 7）

对象编码为 `size(meta)` + N 个 `(symbol_id, value)` 对：

```
对象 {"name": "张三", "age": 30}

假设符号表: [0: "name", 1: "age"]

编码过程：
  字段数 = 2，内联
  type_meta: type=7, meta=2+1=3  →  (3 << 3) | 7 = 0x1F

  字段 1: symbol_id=0 (cmpr_ulong: [0x00])
           value: STRING "张三"
  字段 2: symbol_id=1 (cmpr_ulong: [0x01])
           value: LONG 30

  编码结果: [0x1F, 0x00, <"张三"的编码>, 0x01, <30的编码>]
```

```cpp
void encodeObject(const Inspector &inspector) {
    write_type_and_size_impl(out, OBJECT::ID, inspector.children());
    inspector.traverse(object_traverser);  // 递归编码每个字段
}

// 遍历器回调：先写 symbol_id，再写 value
void BinaryEncoder::field(const Symbol &symbol, const Inspector &inspector) {
    write_cmpr_ulong_impl(out, symbol.getValue());
    encodeValue(inspector);
}
```

---

## 五、符号表编码

符号表在整个 SLIME 结构的最前面编码，先于根值：

```
┌─────────────────────┬──────────────────────┐
│    Symbol Table      │      Root Value       │
└─────────────────────┴──────────────────────┘
```

编码格式：

```
<cmpr_ulong: 符号数量 N>
重复 N 次:
  <cmpr_ulong: 符号名字节长度>
  <bytes: 符号名>
```

符号按出现顺序编号，从 0 开始。

```cpp
void encodeSymbolTable(const Slime &slime) {
    size_t numSymbols = slime.symbols();
    write_cmpr_ulong_impl(out, numSymbols);
    for (size_t i = 0; i < numSymbols; ++i) {
        Memory image = slime.inspect(Symbol(i));
        write_cmpr_ulong_impl(out, image.size);
        out.write(image.data, image.size);
    }
}
```

!!! tip "符号表的空间优势"
    考虑一个包含 1000 个用户对象的数组，每个对象都有 `name`、`email`、`age` 字段。在 JSON 中，这些字段名重复 1000 次。在 SLIME 中，字段名只在符号表中存储一次，对象中只需 1 字节的符号 ID（`cmpr_ulong` 编码的 0、1、2）。仅此一项就能节省数 KB。

---

## 六、完整编码示例

以下面的 JSON 等价数据为例，逐字节展示 SLIME 二进制编码：

```json
{
  "id": 12345,
  "name": "Alice",
  "active": true,
  "tags": ["dev", "ops"]
}
```

### Step 1：构建符号表

```
符号 0: "id"       (2 字节)
符号 1: "name"     (4 字节)
符号 2: "active"   (6 字节)
符号 3: "tags"     (4 字节)
```

### Step 2：编码符号表

```
04                    符号数量: 4
02 69 64              符号 0: 长度 2, "id"
04 6E 61 6D 65        符号 1: 长度 4, "name"
06 61 63 74 69 76 65  符号 2: 长度 6, "active"
04 74 61 67 73        符号 3: 长度 4, "tags"
```

### Step 3：编码根对象

```
根对象 (4 个字段):
  type=7(OBJECT), meta=4+1=5
  type_meta = (5 << 3) | 7 = 0x2F
```

### Step 4：编码每个字段

```
字段 "id": 12345
  symbol_id: 00
  LONG: zigzag(12345) = 24690 = 0x6072
    字节 (LE): [0x72, 0x60]  →  2 字节
    type_meta: type=2, meta=2  →  (2 << 3) | 2 = 0x12
  → 00 12 72 60

字段 "name": "Alice"
  symbol_id: 01
  STRING: 长度 5, 内联 meta=5+1=6
    type_meta: (6 << 3) | 4 = 0x34
  → 01 34 41 6C 69 63 65

字段 "active": true
  symbol_id: 02
  BOOL true: type_meta = (1 << 3) | 1 = 0x09
  → 02 09

字段 "tags": ["dev", "ops"]
  symbol_id: 03
  ARRAY: 2 个元素, 内联 meta=2+1=3
    type_meta: (3 << 3) | 6 = 0x1E
    元素 "dev": STRING 长度 3, meta=3+1=4
      type_meta: (4 << 3) | 4 = 0x24
      → 24 64 65 76
    元素 "ops": STRING 长度 3, meta=3+1=4
      → 24 6F 70 73
  → 03 1E 24 64 65 76 24 6F 70 73
```

### Step 5：完整二进制

```
符号表:
04 02 69 64 04 6E 61 6D 65 06 61 63 74 69 76 65 04 74 61 67 73

根对象:
2F 00 12 72 60 01 34 41 6C 69 63 65 02 09 03 1E 24 64 65 76 24 6F 70 73

总计: 44 字节
```

对比同样数据的 JSON 紧凑模式（`{"id":12345,"name":"Alice","active":true,"tags":["dev","ops"]}`）需要 **62 字节**。SLIME 节省约 **29%**。

如果这样的对象有 1000 个组成数组，符号表仍然只有 21 字节（不随对象数量增长），而 JSON 中的字段名会重复 1000 次。此时 SLIME 的空间优势会更加明显。

---

## 七、编解码架构

### 7.1 编码器（BinaryEncoder）

C++ 编码器通过**遍历器模式（Traverser Pattern）**递归编码：

```
BinaryEncoder
  ├── implements ArrayTraverser      (数组元素回调)
  └── implements ObjectSymbolTraverser (对象字段回调)

编码流程:
  encode(Slime)
    ├── encodeSymbolTable()
    └── encodeValue(root)
          ├── encodeNix()
          ├── encodeBool()
          ├── encodeLong()       → zigzag + write_type_and_bytes<false>
          ├── encodeDouble()     → encode_double + write_type_and_bytes<true>
          ├── encodeString()     → write_type_and_size + raw bytes
          ├── encodeData()       → write_type_and_size + raw bytes
          ├── encodeArray()      → write_type_and_size + traverse(ArrayTraverser)
          └── encodeObject()     → write_type_and_size + traverse(ObjectTraverser)
```

### 7.2 解码器（BinaryDecoder）

解码器通过 **Inserter 模式**将解码出的值插入到 Slime 结构中：

```
Inserter 层次:
  SlimeInserter        设置 Slime 的根值
  ArrayInserter        向数组追加元素
  ObjectSymbolInserter 向对象设置字段

解码流程:
  decode(bytes, slime)
    ├── decodeSymbolTable()    读取符号表
    └── decodeValue(SlimeInserter)
          ├── 读取 type_meta 字节
          ├── 分离 type 和 meta
          └── 根据 type 调用对应 decode 函数
                ├── decodeNix()
                ├── decodeBool(meta)
                ├── decodeLong(meta)      → read_bytes<false> + decode_zigzag
                ├── decodeDouble(meta)    → read_bytes<true>  + decode_double
                ├── decodeString(meta)    → read_size + read bytes
                ├── decodeData(meta)
                ├── decodeArray(meta)     → read_size + 循环 decodeValue(ArrayInserter)
                └── decodeObject(meta)    → read_size + 循环 (read symbol_id + decodeValue)
```

### 7.3 符号重映射

当使用 `decode_into` 将一个 SLIME 结构合并到另一个已有的 SLIME 结构中时，两个结构的符号表可能不一致。C++ 实现通过模板参数 `remap_symbols` 在编译期选择是否启用符号重映射：

```cpp
// 直接解码：符号 ID 直接使用
template<> struct BinaryDecoder<false> : DirectSymbols { ... };

// 合并解码：符号 ID 需要重映射
template<> struct BinaryDecoder<true>  : MappedSymbols { ... };
```

`MappedSymbols` 维护一个从源符号 ID 到目标符号 ID 的映射表：

```cpp
struct MappedSymbols {
    std::vector<Symbol> symbol_mapping;
    Symbol map_symbol(Symbol symbol) const {
        return symbol_mapping[symbol.getValue()];
    }
};
```

### 7.4 BinaryView — 零拷贝只读视图（Java）

Java 实现中有一个独特的 `BinaryView` 类，它提供了一种**不反序列化**就能读取 SLIME 数据的方式。类似 FlatBuffers 的零拷贝理念：

```java
public static Inspector inspect(byte[] data) {
    // 1. 解码符号表
    var names = new SymbolTable();
    BinaryDecoder.decodeSymbolTable(input, names);

    // 2. 构建索引（不解码值）
    var index = new DecodeIndex(...);
    buildIndex(input, index, 0, 0);

    // 3. 返回 BinaryView，直接在原始字节上操作
    return new BinaryView(input.getBacking(), names, index.getBacking(), 0);
}
```

`BinaryView` 通过预构建的索引直接在原始 `byte[]` 上定位和读取字段值，避免了完整反序列化的内存分配开销。每次访问字段时才按需解码该字段的值。

---

## 八、错误处理

SLIME 的解码器采用**宽容解码 + 后置报错**的策略：

```cpp
// 解码失败不立即抛异常，而是标记失败
if (input.failed() && !remap_symbols) {
    slime.wrap("partial_result");
    slime.get().setLong("offending_offset", input.get_offset());
    slime.get().setString("error_message", input.get_error_message());
}
return input.failed() ? 0 : input.get_offset();
```

当解码遇到问题时，已成功解码的部分被包装到 `partial_result` 字段中，同时附加错误偏移和错误消息。调用者通过返回值 0 判断解码是否失败。

规范中还提到一个简化技巧：

> When decoding, it is safe to pad with 0 bytes if there is a buffer underflow.

缓冲区不足时用 0 填充是安全的，因为 0 对应 NIX 类型、meta=0、cmpr_ulong 值 0——都是合法的空/零值。这大大简化了解码器的边界检查逻辑。

---

## 九、JSON 互转

SLIME 与 JSON 之间可以无损互转（除了 DATA 类型，JSON 中无对应）。Vespa 的 `JsonFormat` 类提供了 JSON 编解码能力，且支持 compact/pretty 两种模式：

```cpp
// SLIME → JSON
JsonFormat::encode(slime, output, true);   // compact
JsonFormat::encode(slime, output, false);  // pretty

// JSON → SLIME
JsonFormat::decode(json_string, slime);
```

DATA 类型在 JSON 中编码为十六进制字符串，前缀 `0x`：

```cpp
void encodeDATA(const Memory &memory) {
    // 输出 "0x" + hex dump
    *p++ = '"'; *p++ = '0'; *p++ = 'x';
    for (; pos < end; ++pos) {
        *p++ = hex[(*pos >> 4) & 0xf];
        *p++ = hex[*pos & 0xf];
    }
    *p = '"';
}
```

---

## 十、设计取舍分析

### 10.1 前置大小 vs 结束标记

SLIME 选择了**前置元素个数**而非结束标记或前置字节大小：

| 方案 | 编码 | 解码 | 跳过 |
|------|------|------|------|
| 结束标记 | 简单 | 需要扫描标记 | 需要扫描 |
| 前置字节大小 | 需要两次遍历或回填 | 简单 | 可以直接跳过 |
| 前置元素个数（SLIME） | 简单（计数器） | 简单 | 不能跳过 |

SLIME 选择前置元素个数是因为：
- 编码时无需预计算子结构的字节大小（单次遍历）
- 解码时无需查找结束标记
- 代价是无法在二进制流中跳过子结构（如需跳过，可将子结构编码为 DATA 类型）

### 10.2 与其他协议的对比

| 特性 | SLIME | MessagePack | CBOR | Protobuf |
|------|-------|-------------|------|----------|
| Schema | 不需要 | 不需要 | 不需要 | 需要 |
| 符号表 | 有 | 无 | 无 | N/A |
| 整数编码 | Zigzag + 截断 | 类型前缀 | 主类型编码 | Varint |
| 浮点优化 | 字节反转截断 | 无（固定 5/9 字节） | 可选半精度 | 固定 4/8 字节 |
| 零值优化 | 1 字节 | 1 字节 | 1 字节 | 0 字节（不传） |
| 自描述 | 结构级（符号表） | 值级 | 值级 | 否 |

SLIME 的独特之处在于**符号表**和**浮点字节反转**。符号表使其在重复字段名场景下比 MessagePack/CBOR 更紧凑；字节反转使常见浮点数（0.0、1.0、小整数浮点）的编码极为紧凑。

### 10.3 局限性

- **不支持随机跳过**：由于使用前置元素个数而非字节大小，无法在不解析的情况下跳过子结构
- **符号 ID 仅结构内有效**：不同 SLIME 结构间的符号 ID 不通用，合并时需要重映射
- **生态封闭**：作为 Vespa 内部协议，缺乏独立的语言实现和工具链
- **无 Schema 演进机制**：不像 Protobuf 有字段编号保证兼容性
