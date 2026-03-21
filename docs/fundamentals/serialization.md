# 序列化与反序列化协议：全面对比与选型指南

## 一、什么是序列化

序列化（Serialization）是将内存中的数据结构转换为可存储或传输的字节序列的过程；反序列化（Deserialization）则是其逆过程。

```
内存对象  ──序列化──→  字节流  ──传输/存储──→  字节流  ──反序列化──→  内存对象
(语言相关)             (语言无关)                                    (可以是另一种语言)
```

序列化协议的选择直接影响系统的**性能**、**兼容性**、**可维护性**和**跨语言能力**，是分布式系统设计中的基础决策之一。

### 1.1 评估维度

选择序列化协议时，需要关注以下核心维度：

| 维度 | 说明 |
|------|------|
| 编码效率 | 序列化后数据体积大小 |
| 编解码速度 | 序列化/反序列化的 CPU 耗时 |
| 可读性 | 人类能否直接阅读和调试 |
| Schema 要求 | 是否需要预定义数据结构 |
| 兼容性 | 字段增删后能否前后向兼容 |
| 语言支持 | 支持的编程语言数量和质量 |
| 生态成熟度 | 工具链、文档、社区活跃度 |

---

## 二、文本协议

### 2.1 JSON（JavaScript Object Notation）

JSON 是当前使用最广泛的文本序列化格式，源自 JavaScript 对象字面量语法，由 Douglas Crockford 在 2001 年推广。

#### 数据类型

JSON 仅支持 6 种数据类型：

```json
{
  "string": "hello",
  "number": 42,
  "float": 3.14,
  "boolean": true,
  "null_value": null,
  "array": [1, 2, 3],
  "object": {"nested": "value"}
}
```

#### 编码示例

以一个用户对象为例，后续所有协议都使用同一结构作为对比：

```json
{
  "id": 12345,
  "name": "张三",
  "email": "zhangsan@example.com",
  "tags": ["developer", "golang"],
  "active": true
}
```

该 JSON 编码后约 **120 字节**（含空格格式化）或 **98 字节**（紧凑模式）。

#### 优点

- **人类可读**：无需任何工具即可阅读和编辑，调试极为方便
- **语言无关**：几乎所有主流语言都有高质量的 JSON 库
- **浏览器原生支持**：`JSON.parse()` / `JSON.stringify()` 是 Web 开发的基石
- **Schema-free**：无需预定义结构，灵活应对快速迭代
- **标准明确**：RFC 8259 规范简洁，实现一致性好

#### 缺点

- **体积冗余**：字段名在每条记录中重复传输，批量数据场景浪费严重
- **不支持二进制**：图片、音频等二进制数据必须 Base64 编码，体积膨胀约 33%
- **数值精度问题**：JavaScript 的 `Number` 类型是 IEEE 754 双精度浮点数，超过 2^53 的整数会丢失精度
- **无注释支持**：标准 JSON 不允许注释，作为配置文件不够友好
- **解析性能**：文本解析天然慢于二进制格式

!!! warning "大整数陷阱"
    在与前端交互时，后端返回的 `int64` 类型 ID 如果超过 `9007199254740991`（2^53 - 1），JavaScript 会静默丢失精度。常见解决方案是将大整数序列化为字符串：
    ```json
    {"id": "9223372036854775807"}
    ```

#### 适用场景

- RESTful API 请求与响应
- 前后端数据交互
- 配置文件（需要人工编辑的场景）
- 日志结构化输出

---

### 2.2 XML（eXtensible Markup Language）

XML 诞生于 1998 年，曾是 Web 服务和企业集成的主流格式。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<user>
  <id>12345</id>
  <name>张三</name>
  <email>zhangsan@example.com</email>
  <tags>
    <tag>developer</tag>
    <tag>golang</tag>
  </tags>
  <active>true</active>
</user>
```

同样的数据，XML 编码约 **230 字节**，是 JSON 的 2 倍以上。

#### 优点

- **强 Schema 验证**：XSD/DTD 提供严格的类型约束和结构校验
- **命名空间**：支持多个 Schema 在同一文档中共存，避免命名冲突
- **成熟的查询语言**：XPath、XQuery 提供强大的数据查询能力
- **转换能力**：XSLT 可以将 XML 转换为任意格式
- **属性与元素分离**：`<user id="123" active="true">` 的属性语法在某些场景更紧凑

#### 缺点

- **极度冗余**：开闭标签导致大量重复，数据信噪比低
- **解析复杂**：DOM 解析内存占用大，SAX 解析编程复杂
- **开发体验差**：手写 XML 痛苦，XML Schema 本身的学习曲线陡峭
- **安全隐患**：XXE（XML External Entity）注入是常见攻击面

!!! danger "XXE 攻击"
    如果 XML 解析器没有禁用外部实体，攻击者可以读取服务器文件：
    ```xml
    <?xml version="1.0"?>
    <!DOCTYPE foo [
      <!ENTITY xxe SYSTEM "file:///etc/passwd">
    ]>
    <user><name>&xxe;</name></user>
    ```
    务必在解析 XML 时禁用 `EXTERNAL_GENERAL_ENTITIES` 和 `EXTERNAL_PARAMETER_ENTITIES`。

#### 适用场景

- SOAP Web 服务（遗留系统）
- 企业应用集成（EDI、HL7 医疗数据）
- 文档标记（XHTML、SVG、Office Open XML）
- Android 布局文件

---

### 2.3 YAML（YAML Ain't Markup Language）

YAML 以人类可读性为首要设计目标，广泛用于配置文件。

```yaml
id: 12345
name: 张三
email: zhangsan@example.com
tags:
  - developer
  - golang
active: true
```

#### 优点

- **可读性最佳**：缩进表示层级，没有多余的括号和引号
- **支持注释**：`#` 开头的注释让配置文件更易维护
- **锚点与引用**：`&anchor` 和 `*anchor` 可以复用重复数据
- **多文档支持**：单文件中用 `---` 分隔多个文档

#### 缺点

- **缩进敏感**：Tab 和空格混用会导致难以发现的错误
- **规范复杂**：YAML 1.2 规范超过 80 页，解析器实现差异大
- **隐式类型转换**：`yes`/`no`、`on`/`off` 被自动转为布尔值，`010` 被当作八进制

!!! warning "YAML 的隐式类型转换陷阱"
    ```yaml
    # 这些值不是你以为的字符串
    country: NO        # 布尔值 false，不是挪威国家代码
    version: 1.10      # 浮点数 1.1，不是字符串 "1.10"
    port: 010          # 八进制 8，不是十进制 10
    ```
    建议对所有非数值、非布尔的值加引号：`country: "NO"`。

- **安全隐患**：某些 YAML 库支持反序列化为任意对象，可导致远程代码执行

#### 适用场景

- Kubernetes 资源定义
- Docker Compose 配置
- CI/CD 流水线配置（GitHub Actions、GitLab CI）
- Ansible Playbook

!!! note "YAML 不适合数据传输"
    YAML 的设计目标是人类可读的配置，而非高效的数据交换。在服务间通信中使用 YAML 会带来不必要的解析开销和安全风险。

---

## 三、二进制协议（Schema-based）

这一类协议要求预先定义数据结构（Schema），编码时只传输值，不传输字段名，因此体积小、速度快。

### 3.1 Protocol Buffers（Protobuf）— Google

Protobuf 是 Google 于 2008 年开源的二进制序列化协议，目前是 gRPC 的默认序列化格式，在业界使用极为广泛。

#### Schema 定义

```protobuf
syntax = "proto3";

message User {
  int32  id     = 1;
  string name   = 2;
  string email  = 3;
  repeated string tags = 4;
  bool   active = 5;
}
```

其中 `= 1`、`= 2` 是**字段编号**（field number），而非默认值。编码时使用字段编号代替字段名，这是 Protobuf 体积小的关键。

#### 编码原理

Protobuf 使用 **Tag-Length-Value (TLV)** 编码：

```
┌──────────┬──────────┬────────────┐
│   Tag    │  Length  │   Value    │
│ (varint) │ (varint) │  (bytes)  │
└──────────┴──────────┴────────────┘

Tag = (field_number << 3) | wire_type
```

Wire Type 定义了值的编码方式：

| Wire Type | 含义 | 适用类型 |
|-----------|------|----------|
| 0 | Varint | int32, int64, bool, enum |
| 1 | 64-bit | fixed64, double |
| 2 | Length-delimited | string, bytes, 嵌套 message |
| 5 | 32-bit | fixed32, float |

**Varint 编码**是 Protobuf 高效编码整数的核心：每个字节的最高位是继续标志位（MSB），剩余 7 位存储数据。小整数只需 1-2 字节。

```
数值 300 的 Varint 编码：
300 = 100101100 (二进制)

拆分为 7-bit 组（低位在前）：
  0101100  0000010
加 MSB 继续位：
 10101100 00000010  → 2 字节

对比 int32 固定编码需要 4 字节
```

同样的用户对象，Protobuf 编码后约 **45 字节**，仅为 JSON 的一半不到。

#### 兼容性规则

Protobuf 的前后向兼容性是其核心优势之一：

```protobuf
// v1：原始版本
message User {
  int32  id   = 1;
  string name = 2;
}

// v2：新增字段（前向兼容）
message User {
  int32  id     = 1;
  string name   = 2;
  string email  = 3;  // 新字段，旧代码会忽略
}

// v3：删除字段（后向兼容）
message User {
  int32  id     = 1;
  string name   = 2;
  reserved 3;          // 保留已删除的字段编号
}
```

!!! tip "兼容性黄金规则"
    - **绝不更改已有字段的编号**
    - **绝不更改已有字段的类型**（部分类型兼容除外）
    - **删除字段时使用 `reserved`** 防止编号被复用
    - **新增字段使用新编号**

#### 优点

- 体积小，编解码速度快
- 强类型约束，编译期发现错误
- 优秀的前后向兼容性
- 多语言代码自动生成（C++、Java、Go、Python、Rust 等）
- 与 gRPC 深度集成

#### 缺点

- 二进制格式不可读，调试需要 `protoc --decode` 或 grpcurl
- 需要预定义 `.proto` 文件并编译，开发流程增加一步
- 不适合动态或自描述场景
- Map 类型的序列化顺序不确定，不能直接做字节级比较
- proto3 取消了 `required`/`optional` 语义（后来部分恢复），默认值不被序列化

#### 适用场景

- gRPC 微服务间通信
- 移动端与后端通信（省流量）
- 高性能内部 RPC
- 持久化存储格式

---

### 3.2 Apache Thrift（原 Facebook）

Thrift 由 Facebook 于 2007 年开发，提供了序列化 + RPC 的一体化方案。

#### Schema 定义

```thrift
struct User {
  1: i32    id,
  2: string name,
  3: string email,
  4: list<string> tags,
  5: bool   active,
}

service UserService {
  User getUser(1: i32 id),
  void createUser(1: User user),
}
```

#### 传输协议选项

Thrift 的一大特色是支持多种传输协议：

| 协议 | 特点 | 适用场景 |
|------|------|----------|
| TBinaryProtocol | 简单二进制，字段类型+编号+值 | 通用场景 |
| TCompactProtocol | 使用 varint 和 zigzag 编码，体积更小 | 带宽受限 |
| TJSONProtocol | JSON 格式，可读 | 调试 |

#### 优点

- 序列化 + RPC 一体化，降低技术选型复杂度
- 多种传输协议可选，灵活适配不同场景
- 支持丰富的数据类型（map、set、enum）
- 多语言代码生成

#### 缺点

- 社区活跃度持续下降，不如 Protobuf/gRPC
- 文档质量和工具链偏弱
- 缺少官方的 Schema 演进指导
- 框架整体较重

#### 适用场景

- Facebook 系技术栈
- 已有 Thrift 基础的遗留系统
- 需要一体化 RPC 方案的场景

---

### 3.3 Apache Avro

Avro 是 Hadoop 之父 Doug Cutting 创建的序列化框架，专为大数据场景设计。

#### Schema 定义

Avro 的 Schema 使用 JSON 表示：

```json
{
  "type": "record",
  "name": "User",
  "fields": [
    {"name": "id",     "type": "int"},
    {"name": "name",   "type": "string"},
    {"name": "email",  "type": "string"},
    {"name": "tags",   "type": {"type": "array", "items": "string"}},
    {"name": "active", "type": "boolean"}
  ]
}
```

#### 编码原理

与 Protobuf 不同，Avro 编码时**完全不包含字段标识信息**——既没有字段名，也没有字段编号。解码完全依赖 Schema 中定义的字段顺序：

```
Protobuf:  [tag1][value1] [tag2][value2] [tag3][value3]  ← 每个值前有 tag
Avro:      [value1] [value2] [value3]                     ← 纯值，无 tag
```

这使得 Avro 的编码体积比 Protobuf 更小，但也意味着**没有 Schema 就无法解码**。

#### Schema 演进

Avro 的 Schema 演进模型是其最强大的特性。读端和写端可以使用不同版本的 Schema，Avro 运行时通过 **Schema Resolution** 自动适配：

```
Writer Schema (v1)          Reader Schema (v2)
┌───────────────┐           ┌───────────────┐
│ id:    int     │           │ id:    int     │
│ name:  string  │    ──→    │ name:  string  │
│ email: string  │  resolve  │ email: string  │
└───────────────┘           │ role:  string   │  ← 新增字段，使用默认值
                            └───────────────┘
```

#### 与 Kafka 的结合

Avro 在 Kafka 生态中通常与 **Schema Registry** 配合使用：

```
Producer                    Schema Registry               Consumer
   │                              │                           │
   ├─ 注册 Schema v1 ──────────→  │                           │
   │  ← schema_id=1 ────────────  │                           │
   │                              │                           │
   ├─ 发送消息 ──────────────────────────────────────────────→ │
   │  [magic][schema_id=1][avro_bytes]                        │
   │                              │                           │
   │                              │  ← 获取 Schema (id=1) ── │
   │                              │  ── 返回 Schema ────────→ │
   │                              │                           │
   │                              │         解码 avro_bytes   │
```

#### 优点

- Schema 演进能力最强（读写 Schema 可独立演进）
- 编码体积最小（无字段标识开销）
- 不需要代码生成（可动态解析）
- 与 Hadoop/Spark/Kafka 生态深度集成
- Avro 文件格式自带 Schema（自描述）

#### 缺点

- 随机访问能力弱（流式格式，需顺序读取）
- 编解码速度不如 Protobuf
- 脱离大数据生态使用较少
- Schema 用 JSON 编写，较为冗长

#### 适用场景

- Kafka 消息序列化（配合 Schema Registry）
- Hadoop/Spark 数据管道
- 数据湖存储格式
- 需要强 Schema 演进的场景

---

## 四、二进制协议（Schema-free）

这类协议不需要预定义 Schema，可以像 JSON 一样灵活使用，但采用二进制编码以获得更好的性能。

### 4.1 MessagePack

MessagePack 自称 "It's like JSON, but fast and small"，是 JSON 的二进制替代品。

#### 编码对比

```
JSON (27 bytes):
{"compact":true,"schema":0}

MessagePack (18 bytes):
82 a7 63 6f 6d 70 61 63 74 c3 a6 73 63 68 65 6d 61 00
```

MessagePack 的类型系统是 JSON 的超集：

| 类型 | JSON | MessagePack |
|------|------|-------------|
| 整数 | Number | int 8/16/32/64, uint 8/16/32/64 |
| 浮点 | Number | float 32/64 |
| 字符串 | String | str |
| 二进制 | 不支持 | bin |
| 布尔 | Boolean | bool |
| 空值 | null | nil |
| 数组 | Array | array |
| 映射 | Object | map |
| 扩展 | 不支持 | ext（自定义类型） |

#### 优点

- 比 JSON 体积小 30%-50%，速度快 2-5 倍
- Schema-free，使用方式和 JSON 一样灵活
- 原生支持二进制数据
- 类型系统比 JSON 丰富（区分整数/浮点、有二进制类型）
- 多语言支持广泛

#### 缺点

- 不可读，调试需要工具
- 没有 Schema 约束，无法做编译期校验
- 压缩率不如 Protobuf 等 Schema-based 方案
- 不同语言实现的兼容性偶有差异

#### 适用场景

- Redis 等缓存系统中的值序列化
- 实时通信协议（游戏、IM）
- 需要替换 JSON 但不想引入 Schema 管理的场景
- 嵌入式系统间通信

---

### 4.2 CBOR（Concise Binary Object Representation）

CBOR 是 IETF 标准（RFC 8949），设计目标是在极小的代码量下实现高效的二进制编码，特别适合受限设备。

#### 编码结构

CBOR 使用统一的 **主类型 + 附加信息** 编码：

```
每个数据项的第一个字节：
┌─────────────┬──────────────────┐
│ 主类型 (3bit) │ 附加信息 (5bit)   │
└─────────────┴──────────────────┘

主类型：
0 = 无符号整数    1 = 负整数
2 = 字节串        3 = 文本串
4 = 数组          5 = 映射
6 = 语义标签      7 = 浮点/简单值
```

#### 优点

- IETF 标准（RFC 8949），规范严谨且稳定
- 代码实现极小（适合嵌入式 MCU）
- 支持语义标签扩展（时间戳、大数、正则等）
- 确定性编码模式（可用于签名和去重）

#### 缺点

- 生态不如 MessagePack 和 JSON
- 开发者熟悉度低，工具链偏少

#### 适用场景

- IoT / 嵌入式设备通信（CoAP 协议默认格式）
- WebAuthn / FIDO2 认证协议
- COSE 数字签名
- 需要标准化二进制编码的场景

---

## 五、零拷贝协议

传统的序列化协议在反序列化时需要将字节流解析为内存对象，这涉及内存分配和数据拷贝。零拷贝协议消除了这个开销——序列化后的字节流可以直接作为内存中的数据结构访问。

### 5.1 FlatBuffers — Google

FlatBuffers 由 Google 为游戏开发创建，是 Android 平台上 TensorFlow Lite 模型格式的基础。

#### Schema 定义

```fbs
table User {
  id:    int;
  name:  string;
  email: string;
  tags:  [string];
  active: bool = true;
}

root_type User;
```

#### 零拷贝原理

```
传统协议（如 Protobuf）：
  字节流 ──parse──→ 分配内存 ──→ 填充字段 ──→ 返回对象
                    (malloc)     (memcpy)

FlatBuffers：
  字节流 ──────→ 直接在 buffer 上读取字段（通过偏移量表）
                 无内存分配，无数据拷贝
```

FlatBuffers 的数据布局：

```
┌────────────┬───────────┬──────────────────────────┐
│ root offset │  vtable   │        data tables       │
│  (4 bytes)  │ (offsets) │  (字段值，按偏移量访问)    │
└────────────┴───────────┴──────────────────────────┘

访问 user.name：
1. 从 vtable 查找 name 字段的偏移量
2. 从 buffer 的该偏移位置直接读取字符串
全程无需 malloc 或 memcpy
```

#### 优点

- **零反序列化开销**：访问时间 O(1)，不随数据大小增长
- 内存效率极高，无需额外分配
- 支持随机访问任意字段
- 适合只读场景（如读取配置、模型参数）
- 支持可选的对象 API（牺牲零拷贝换取易用性）

#### 缺点

- 构建数据时 API 较繁琐（需要先创建子对象，再组装）
- 序列化后体积不一定比 Protobuf 小（有对齐填充）
- 修改已序列化的数据困难（本质是只读设计）
- 生态和社区不如 Protobuf

#### 适用场景

- 游戏引擎（Unity、Unreal）
- TensorFlow Lite 模型格式
- 对反序列化延迟极度敏感的系统
- 嵌入式/移动端内存受限场景

---

### 5.2 Cap'n Proto

Cap'n Proto 由 Protobuf v2 的作者 Kenton Varda 创建，可以看作是 Protobuf 理念的极致演进。

#### Schema 定义

```capnp
struct User {
  id    @0 : Int32;
  name  @1 : Text;
  email @2 : Text;
  tags  @3 : List(Text);
  active @4 : Bool;
}
```

#### 核心理念

Cap'n Proto 的核心思想是 **"编码即是内存布局"**——数据在内存中的表示和在网络上的表示完全一致，因此序列化的开销严格为零。

```
传统流程：  内存对象 ──serialize──→ 字节流 ──send──→ 字节流 ──deserialize──→ 内存对象
Cap'n Proto：内存对象 ═══════════════ 字节流 ══send══→ 字节流 ═══════════════ 内存对象
             (相同的东西，零转换)
```

#### 与 FlatBuffers 的区别

| 特性 | FlatBuffers | Cap'n Proto |
|------|-------------|-------------|
| 随机访问 | 通过 vtable | 通过指针 |
| 修改数据 | 困难 | 支持（原地修改） |
| 内置 RPC | 无 | 有 |
| 对齐方式 | 需要手动对齐 | 自动 8 字节对齐 |
| 主要用户 | Google, Android | Cloudflare |

#### 优点

- 严格零拷贝，序列化开销为零
- 内置 RPC 框架（支持 promise pipelining）
- 支持原地修改数据
- 设计精良，API 比 FlatBuffers 更直观

#### 缺点

- 语言支持有限（C++、Rust 为一等公民，其他语言支持参差不齐）
- 8 字节对齐导致体积可能大于 Protobuf
- 社区规模小，生态尚未成熟
- 学习资料少

#### 适用场景

- 进程间通信（IPC）
- Cloudflare Workers 内部通信
- 对延迟有极致要求的系统
- 需要 RPC + 零拷贝序列化一体化的场景

---

## 六、全景对比

### 6.1 性能与体积

以同一个包含 1000 条用户记录的数据集为基准（数据为典型估算值，实际取决于数据特征和实现库）：

| 协议 | 编码体积 | 编码速度 | 解码速度 | 零拷贝 |
|------|----------|----------|----------|--------|
| JSON | 100% (基准) | 1x | 1x | 否 |
| XML | ~200% | 0.5x | 0.3x | 否 |
| Protobuf | ~35% | 5-10x | 5-10x | 否 |
| Thrift (compact) | ~38% | 4-8x | 4-8x | 否 |
| Avro | ~30% | 3-6x | 3-6x | 否 |
| MessagePack | ~55% | 2-4x | 2-4x | 否 |
| FlatBuffers | ~45% | 3-5x | 即时 | 是 |
| Cap'n Proto | ~50% | 即时 | 即时 | 是 |

### 6.2 特性矩阵

| 特性 | JSON | Protobuf | Avro | MsgPack | CBOR | FlatBuf | Cap'n Proto |
|------|------|----------|------|---------|------|---------|-------------|
| 人类可读 | 是 | 否 | 否 | 否 | 否 | 否 | 否 |
| 需要 Schema | 否 | 是 | 是 | 否 | 否 | 是 | 是 |
| Schema 演进 | N/A | 好 | 最佳 | N/A | N/A | 好 | 好 |
| 自描述 | 是 | 否 | 文件级 | 是 | 是 | 否 | 否 |
| 二进制数据 | 否 | 是 | 是 | 是 | 是 | 是 | 是 |
| 内置 RPC | 否 | gRPC | 否 | 否 | 否 | 否 | 是 |
| 语言支持 | 极广 | 广 | 中 | 广 | 中 | 中 | 窄 |
| 标准化 | RFC 8259 | Google | Apache | 社区 | RFC 8949 | Google | 社区 |

---

## 七、选型决策树

```
需要序列化协议？
│
├─ 与浏览器/前端交互？
│  └─→ JSON
│
├─ 配置文件？
│  ├─ 需要注释？ → YAML
│  └─ 不需要？   → JSON
│
├─ 服务间通信（RPC）？
│  ├─ 愿意引入 Schema？
│  │  ├─ 需要 gRPC？        → Protobuf
│  │  ├─ 遗留 Thrift 系统？  → Thrift
│  │  └─ 需要零拷贝？       → Cap'n Proto / FlatBuffers
│  └─ 不想引入 Schema？
│     └─→ MessagePack
│
├─ 大数据管道 / 消息队列？
│  └─→ Avro（配合 Schema Registry）
│
├─ IoT / 嵌入式？
│  ├─ 需要 IETF 标准？ → CBOR
│  └─ 内存极度受限？   → FlatBuffers
│
└─ 游戏引擎 / 极致延迟？
   └─→ FlatBuffers
```

---

## 八、实践建议

### 8.1 不要过早优化序列化

对于大多数 Web 应用，JSON 的性能完全够用。只有当序列化成为瓶颈时（通过 profiling 确认），才值得切换到二进制协议。切换协议带来的维护成本、工具链成本和团队学习成本不容忽视。

### 8.2 混合使用是常态

一个典型的现代系统可能同时使用多种协议：

```
浏览器 ──JSON──→ API Gateway ──Protobuf──→ 微服务
                                             │
                                        Avro + Kafka
                                             │
                                        Spark / Flink
```

### 8.3 关注 Schema 演进策略

如果选择了 Schema-based 协议，务必在项目初期就建立 Schema 演进规范：

- **Protobuf**：永远不要复用已删除的字段编号，使用 `reserved` 关键字
- **Avro**：使用 Schema Registry 管理版本，设置兼容性级别（BACKWARD / FORWARD / FULL）
- **通用规则**：新增字段给默认值，删除字段保持向后兼容，字段类型变更要格外谨慎

### 8.4 安全考量

- XML：禁用外部实体解析，防止 XXE 攻击
- YAML：使用 `safe_load` 而非 `load`，禁止反序列化任意对象
- 所有协议：对反序列化的数据进行校验，不信任外部输入的合法性
- 设置反序列化的大小上限，防止内存耗尽攻击
