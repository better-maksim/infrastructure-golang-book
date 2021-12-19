# gRPC 快速起步

### 什么是 gRPC 和 Protobuf

#### gRPC

GRPC 是一个 Google 开源的高性能，开源和通用的 RPC 框架，面向移动和 HTTP/2设计。

#### Protobuf

在开发 gRPC 应用程序时，要先定义服务接口，其中应包括如下信息：消费者消费服务的方式，消费者能够远程调用的方法以及调用这些方法所使用的参数和消息格式等。在服务定义中所使用的语言叫做**接口定义语言(interface definition language, IDL)**。

Protocol Buffer 其实是 Google 出品的一种轻量级、高效的结构化存储格式，目前 pb3 是主流版本。

## gRPC 的优点

- 性能 
  - 压缩性好
  - 序列化和反序列化快，比xml 和 json 快 2-100 倍
  - 传输速度快
- 便捷性
  - 使用简单     自动生成序列化和反序列化代码
  - 维护成本低 只需要维护 proto 文件
  - 向后兼容  不必破坏旧格式
  - 加密性好  二进制的
- 跨语言
  - 跨平台
  - 支持各种主流语言

缺点：

- 通用性差，json 可以任何语言都支持，但是 protobuf 需要专门的解析库

- 自解释性差，只能通过 proto 文件才能了解数据结构

## Mac 环境安装

1.从github上下载protobuf3

[protobuf3下载地址](https://link.zhihu.com/?target=https%3A//github.com/protocolbuffers/protobuf/releases)

![img](https://pic1.zhimg.com/80/v2-9126c2d16d038d069f1d58b50b2b7768_1440w.jpg)


有很多语言版本的，mac下选择第一个。

2.下载下来后解压压缩包，并进入目录

```text
cd protobuf-3.7.0/
```

3.设置编译目录

```text
./configure --prefix=/usr/local/protobuf
```

*4.切换到root用户*

```text
sudo -i
```

5.安装

```text
make
make install
```

6.配置环境变量

找到用户目录/Users/pauljiang 的 .bash_profile文件并编辑

```text
vim .bash_profile
```

![img](https://pic4.zhimg.com/80/v2-f8240ea4048ebbf3cdf8bdd75aa1ed97_1440w.jpg)



按一下回车键

按i进入编辑模式

![img](https://pic3.zhimg.com/80/v2-10f76cf65d3d56f2d014a3dd65833b9a_1440w.jpg)

添加

```text
export PROTOBUF=/usr/local/protobuf 
export PATH=$PROTOBUF/bin:$PATH
```

source一下使文件生效

```text
source .bash_profile
```

7.测试安装结果

```text
protoc --version
```

![img](https://pic3.zhimg.com/80/v2-57275f65c7bb6913c7fe340a30455142_1440w.jpg)

8. 获取 protoc 生成 golang GRPC 插件

```shell
go get github.com/golang/protobuf/protoc-gen-go
```

9. 编写 proto 文件

```protobuf
syntax = "proto3";  //声明版本
option go_package = ".;proto";


service Greeter {
  rpc Hello(HelloRequest) returns (HelloResponse){}
}

message HelloRequest{
  string name = 1;
}

message HelloResponse {
  string greeting = 1;
}
```

```
protoc -I . hello.proto --go_out=plugins=grpc:.
```



为了可以乱序