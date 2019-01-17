---
title: Thrift 相关知识
category: Thrift
date: 2019-01-16 18:00
mathjax: true
---

# Thrift 是什么

Thrift 是一套包含序列化功能和支持服务通信的 RPC 框架，主要包含三大部分: 代码生成，序列化框架，RPC 框架。
其大致相当于 protoc + protobufer + gprc 的组合，并且支持大量语言，保证常用功能在跨语言间功能一致，是一套全栈式的 RPC 解决方案。整体架构如下

![thrift_structure](/image/thrift_structure.png)

# IDL 与代码生成

Thrift IDL 语法可参考[官方文档](http://thrift.apache.org/docs/idl)
其实现细节可见 [github](https://github.com/apache/thrift/tree/master/compiler/cpp/src/thrift/generate)

## 需要注意的地方

### Type 中 `string` 类型说明及正确使用
  
- `string` 按照协议要求需为 UTF-8 编码的字符串
- `binary` 可以理解为 `list<byte>`, 因此字节数组应使用 `binary` 类型
  ps. 部分语言没有实现 `binary` 类型，使用 `list<byte>` 代替
- Java 和 Python 的生成代码有检查 `string` 一定为 UTF-8, 但是由于 go 语言 `string` 底层实现为字节数组因此无法严格保证 UTF-8 编码

### 字段是否必须

Thrift 在字段是否必须有三种语义: `required`, `optional`, 不加显式声明时为 `default`.

协议要求 `default` 为写必填 `required`, 读可选 `optional`, 但是各种语言实现有差异，建议字段都声明 `required` 或者 `optional`

# Thrift 序列化

Thrift 是一个支持跨语言的序列化框架，对标 JSON, Protobuf, Avro, 因此一些使用场景为了保持和 RPC 序列化协议的统一或者 RPC 中间数据存储就会使用 Thrift 序列化。
Thrift 序列化协议实现主要是 Binary, Compact, JSON

## Thrift 支持数据类型

### 简单数据类型

`bool` | `byte` | `i8` | `i16` | `i32` | `i64` | `double`

### 复合数据类型

`string` | `binary` | `map` | `set` | `list` | `struct`

### 特殊数据类型

`void` | `stop`

### PS

`i8` 和 `binary` 有些语言没有实现，建议 `i8` 使用 `byte` 或者 `i16` 替换， `binary` 使用 `list<byte>` 替换

## TLV 编码

二进制编码常用 TLV 编码实现，TLV 是指由数据的类型 Tag, 数据的长度 Length, 数据的值 Value 组成的结构体，几乎可以描述任意数据类型， TLV 的 Value 也可以是一个 TLV 结构，正因为这种嵌套的特性，可以让我们用来包装协议的实现。
Thrift 中 Binary 和 Compact 编码都是采用的 TLV 编码变种实现，两者不同点在于对整数类型处理方面。

## Binary 序列化

简单数据类型为定长编码，包含一个字节类型标识 + 两个字节编号 + 类型对应定长值

这里内部几乎都是使用 Binary 序列化，示例

```thrift
struct Test {
    1: bool Open,
    2: string Name,
    3: i32 ID,
}
```

比如上面的 Test 结构体，Binary 序列化结果是

```plain
test = Test(Open=True, Name="hi", ID=18)
test // 实例序列化 16 进制显示为
    02 0001 01
    0b 0002 00000002 6869
    08 0003 00000012
```

## Compact 序列化

Compact 序列化当时不同于 Binary 点主要在于整数类型使用 zigzag 和 varint 压缩编码实现

### varint 编码

步行长无符号整数编码，每个字节只使用 低 7 位，最高一位作为一个标志位 (msb)

- 下一个 byte 也是该数字的一部分
- 下一个 byte 不是该字节的一部分

该编码好处的对于小数字采用更少字节，大叔街采用更多字节，但大部分使用都是小数字，则整体看压缩效率明显。
比如 300(i32), Binary 序列化下需要 4 个字节，采用 varint 只需要两个字节

### zigzag 编码

varint 解决了无符号编码的问题，假设有符号数也使用 varint 编码，因为负数最高位是 1, 比如 i32 就都会使用 5 个字节了，反而使用更多字节，为了解决有符号负数的问题，先采用 zigzag 编码将有符号数映射到无符号数上，zigzag 具体算法如下

$$
m =
\begin{cases}
n \times 2 & n > 0 \\\
2 \times | n | - 1 & n < 0
\end{cases}
$$

### compact 实现

大致逻辑与 Binary 序列化实现一样，就是将 i16, i32, i64 三种类型使用 zigzag + varint 编码实现， string, map, list, set 复合类型长度只采用 varint 编码

# RPC 框架

Thrift RPC 整个网络服务一般有五个步骤

![thrift_net_service_steps](/image/thrift_net_service_steps.png)

## 通讯协议

Thrift 中包含 `BinaryProtocol` 和 `CompactProtocol` 通讯协议，分别是前面 `Binary` 和 `Compact` 序列化协议加上 `Message` 传输的协议部分。

以典型常见的 HTTP 协议为例，主要包含三部分

- 路由信息(URL)
- 控制信息(Header)
- 数据负载(Body)

主要分析下 `BinaryProtocol` 的实现

`BinaryProtocol` 协议分为严格模式和非严格模式，严格模式下会带上版本 Version 信息，非严格模式下没有版本信息，默认为严格模式。

其中通讯的消息类型主要有四种

- `CALL`
  值为 1, 请求
- `REPLY`
  值为 2, 响应
- `EXCEPTION`
  值为 3, 异常
- `ONEWAY`
  值为 4, 无返回值请求

### 严格模式

四个字节的版本(含调用类型), 四个字节的消息名称长度，四个字节的流水号，消息负载的值，一个字节的结束标记。

```plain
version := uint32(VERSION_1) | uint32(typeID)
WriteI32(int32(version))
WriteString(name)
WriteI32(seqID)
WriteBody(body)
WriteByte(STOP)
```

### 非严格模式

四个字节的消息名称长度，一个字节调用类型，四个字节的流水号，消息负载数据的值，一个字节的结束标记。

```plain
WriteString(name)
WriteByte(typeID)
WriteI32(seqID)
WriteBody(body)
WriteByte(STOP)
```

## Transport 实现

Transport 主要分为两类

- 上层传输通道，负责消息的读写和存储
- 底层传输通道，负责消息在 client/server 之间传输

### 上层 Transport 实现

Transport 主要接口有 open, close, read, write, flush, 官方大部分语言都有多种实现，最常使用的是 `TBufferedTransport` 和 `TFramedTransport`.

- `TBufferedTransport`
  ``TBufferedTransport` 实现主要是采用了 `BufferIO` 来存储实现，主要使用场景是在 BIO(阻塞式IO)下使用
- `TFramedTransport`
  在 `Protocol` 加了 `Header`(四个字节的消息体大小), 主要使用场景为 NIO(非阻塞IO), 其中 C++ 大部分 Thrift Server 采用 NIO 实现。GO 的 Socket 底层是 NIO, 但是在用户层实现了阻塞，所以可以使用 `TBufferedTransport`.

### 如何选择 Transport

- client
  调用下游使用 `Transport`, 联系下游，这是服务的元信息
- server
  从性能和内存使用角度建议使用 TFramedTransport

### 下层 Transport 实现

最常用的有基于 TCP 和 Unix Socket 两种实现方式，大部分 RPC 服务就是使用 TCP 实现，也有 使用 Unix Socket 实现的场景

## Server 实现

由于 Server 实现不考虑跨语言问题，只需要关心实现语言自身特点选用就可以。
一般实现有以下几种

- TSimpleServer (单进程单线程模式，调试使用）)
- ThreadPoolServer (单进程多线程模式)
- TProcessPoolServer (多进程单线程模式，Pie 目前采用)
- 其他基于 NIO 的各种 Server
- AIO 的实现