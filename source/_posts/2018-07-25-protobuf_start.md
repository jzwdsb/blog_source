---
title: protobuf 浅试
date: 2018-07-25
mathjax: true
tags: [protobuf]
---

protobuf 是 Google 推出的语言独立，平台独立的针对序列化数据的组件，可以类比 xml, 但是比 xml 精简。
通过使用 protobuf 语法自定义想要的数据结构，接着用 `protoc` 命令就可以产生针对这种数据格式文件读写的各种语言(C/C++, Python, Java等)的 API.

# Mac 上安装 protobuf

```shell
brew install protobuf
```

# `proto` 语法

自定义序列化数据结构使用特殊的语法，创建 `proto` 文件，其包含如下内容

- `syntax= proto2 | proto3`
- `package`
- `message`
  - `field`
    - `[required | optional | repeated]`
    - `type`
    - `name`
    - `[message]`
  - `tag`

```plain
syntax = "proto2" | "proto3“
package "name";
message "type"
{
    [required | optional | repeated] type "filed" = tags;
}
```

# `message`

一个 `message` 类型表示了一个可以序列化的结构。由若干个 `field` 组成

# `field`

一个 `field` 由以下部分构成

- `type`
  `proto` 支持的类型可见[官网](https://developers.google.com/protocol-buffers/docs/proto3#scalar), 可以是用户自定义的 message
- `field number`
  每个 `field` 最后包含一个独特数字，这个数字在二进制格式中标记这个 `field`,当整个 `message` 投入使用后不应该再修改，这些 `Number` 可取的范围是 $1 - 2^29 - 1$, 但是 $19000 - 19999$ 这个区间属于保留区，不应使用，否则会提示 `FieldDescriptor::FieldDescriptor` 错误。
- `field rule`
  - `singular`
    在一个组织良好的 `message` 中，这 `field` 可以出现一次或零次
  - `repeated`
    在一个组织良好的 `message` 中，这个 `field` 可以出现任意次数，值出现的顺序保留。

## 保留域

可以注明一个 `field` 是 `reserved`，可以标注其 `field number` 或者 `name`, 示例如下

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

`reserved` 标注意为未来可能会抛弃这个 `field`, 提示用户尽可能少使用这个 `field`, 每当 `proto buffer compiler` 编译这个 `proto` 文件生成 `api` 时都会给出相应的警告。

# `Enumeration`

`proto` 中定义枚举类型的语法与 `C++` 基本相同

```proto
enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
```

注意，一定要有 0 值

# `Any`

`Any` 类型可以保存任意类型的信息，直接将任何序列化的 `message` 看做看做字节串存储

# `oneof`

类似 C++ 中的 `union` ,存储时只存储 `oneof` 域中的一个。

```C++
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

当取其中一个域时，其他域自动会被删除，当使用 C++ 时要注意内存崩溃问题.

# Maps

定义 `Map` 的语法如下

```proto
map<key_type, value_type> map_field = N;
```