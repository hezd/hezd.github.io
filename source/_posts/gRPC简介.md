---
title: gRPC简介
date: 2022-03-15 16:21:13
tags:
- gRPC
- Protocol Buffers
categories:
- Android应用层
- 网络
---

##### 1.什么是gRPC

​	gRPC是一个现代开源高性能的远程调用（RPC，remote procedure call）框架，能在任何环境中运行。

使用Protocol Buffers作为接口定义语言（IDL，Interface Definition Language）和底层消息交换格式

##### 2.什么是Protocol Buffers

​	Protocol Buffers提供了一种结构化序列化数据的机制，它像Json除了更快更小并且生成本地语言绑定。

Protocol Buffers是定义语言（DL）的组合（在proto文件中创建），通过proto编译器生成序列化的数据接口，用于本地序列化或网络传输。

##### 3.Protocol Buffers定义语法

protocol buffers使用需要先定义一个proto文件进行数据结构定义，protocol buffers支持数据类型包括常用的原始数据类型，还有以下几个常用类型：

- message 消息类型

- enum 枚举类型

- oneof 限定类型，当消息有多个可选字段最多只能同时设置一个时使用

- map 映射表类型，key-value映射关系数据结构

下面通过一个简单例子来说明：

```protobuf
//helloworld.proto（文件名）
//1.指定使用proto3语法
syntax = "proto3";
//option 是指文件选项，对于proto编译器编译会产生影响
//2.是否生成多个类文件（对于顶级service,message类型），如果false，全部生成在一个类文件中
option java_multiple_files = true;
//3.生成类文件所属包名
option java_package = "io.grpc.examples.helloworld";
//4.生成java包装类的类名，如果不指定默认以proto文件名驼峰形式（proto文件会生成一个包装类）
option java_outer_classname = "HelloWorldProto";
//5.生成Object-C类的前缀
option objc_class_prefix = "HLW";
//6.声明包名，避免消息类型名称冲突
package helloworld;

// 7.用service声明RPC接口
service Greeter {
  // 8.用rpc声明rpc方法，HelloRequest方法参数，HelloReply返回类型
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 9.声明消息类型HelloRequest
message HelloRequest {
  string name = 1;
}

// 10.声明消息类型HelloReply
message HelloReply {
// 11.声明字段 message类型为字符串，指定字段编码为1
  string message = 1;
}

```

##### 4.android中使用gRPC

> 开发环境：
>
> windows10
>
> Android Studio Bumblebee | 2021.1.1 Patch 2
>
> Gradle 7.1.2
>
> JDK 11

以gRPC客户端为例

###### 4.1引入protobuf插件

工程顶层build.gradle中添加

```groovy
id 'com.google.protobuf' version '0.8.18' apply false
```

###### 4.2proto脚本配置

工程子module的build.gradle中进行配置

```groovy
plugins {
   	//...
    // 使用protobuf
    id 'com.google.protobuf'
}
android {
	//...
}
protobuf {
    protoc { artifact = 'com.google.protobuf:protoc:3.19.2' }
    plugins {
        grpc { artifact = 'io.grpc:protoc-gen-grpc-java:1.45.0' // CURRENT_GRPC_VERSION
        }
    }
    generateProtoTasks {
        all().each { task ->
            task.builtins {
                java { option 'lite' }
            }
            task.plugins {
                grpc { // Options added to --grpc_out
                    option 'lite' }
            }
        }
    }
}

dependencies {
	//grpc dependencies
    // You need to build grpc-java to obtain these libraries below.
    implementation 'io.grpc:grpc-okhttp:1.44.1' // CURRENT_GRPC_VERSION
    implementation 'org.apache.tomcat:annotations-api:6.0.53'
    implementation 'io.grpc:grpc-protobuf-lite:1.45.0'
    implementation 'io.grpc:grpc-stub:1.45.0'
}
```

项目build之后会自动生成gRPC相关的类文件

![](https://gitee.com/hezd/GrpcSample/raw/main/screenshots/img.png)

具体实现可参考[gRPC for android](https://grpc.io/docs/platforms/android/ "gRPC for android")和文中的示例程序[gRPCSample](https://github.com/hezd/GrpcSample "gRPCSample")

##### 参考：

https://grpc.io/

https://developers.google.com/protocol-buffers/docs/overview
