---
title: Thrift 相关知识
category: Thrift
date: 2019-01-16 18:00
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