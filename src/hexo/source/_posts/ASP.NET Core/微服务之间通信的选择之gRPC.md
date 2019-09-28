---
title: 微服务之间通信的选择之gRPC
categories: ASP.NET Core
comments: true
date: 2019/09/28
updated: 2019/09/28
tags:
    - ASP.NET Core
    - gRPC
---

## 介绍 
[gRPC](https://grpc.io/)是一种与语言无关的高性能远程过程调用 (RPC) 框架。

**gRPC 的主要优点是：**
- 现代高性能轻量级 RPC 框架。
- 协定优先 API 开发，默认使用协议缓冲区，允许与语言无关的实现。
- 可用于多种语言的工具，以生成强类型服务器和客户端。
- 支持客户端、服务器和双向流式处理调用。
- 使用 Protobuf 二进制序列化减少对网络的使用。

这些优点使 **gRPC 适用于：**
- 效率至关重要的轻量级微服务。
- 需要多种语言用于开发的 Polyglot 系统。
- 需要处理流式处理请求或响应的点对点实时服务。

以上摘自[Microsoft Document](https://docs.microsoft.com/en-us/aspnet/core/grpc/index?view=aspnetcore-3.0)


## .NET中gRPC的简单使用

ASP.NET Core中 gRPC Server 配置
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddGrpc();
}
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapGrpcService<GreeterService>();
    });
}
```

ASP.NET Core中 gRPC Server 服务类
```csharp
public class GreeterService : Greeter.GreeterBase
{
    private readonly ILogger<GreeterService> _logger;
    public GreeterService(ILogger<GreeterService> logger)
    {
        _logger = logger;
    }

    public override Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
    {
        return Task.FromResult(new HelloReply
        {
            Message = "Hello " + request.Name
        });
    }
}
```

greet.proto
```
syntax = "proto3";

option csharp_namespace = "GrpcServer";

package Greet;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply);
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings.
message HelloReply {
  string message = 1;
}

```
有关 protobuf 文件语法的详细信息，参考[官方文档（protobuf）](https://developers.google.com/protocol-buffers/docs/proto3)

客户端：
添加对应的包，复制服务端的proto文件并配置：
```xml
  <ItemGroup>
    <PackageReference Include="Google.Protobuf" Version="3.9.2" />
    <PackageReference Include="Grpc.Net.Client" Version="2.23.2" />
    <PackageReference Include="Grpc.Tools" Version="2.24.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <Protobuf Include="Protos\greet.proto" GrpcServices="Client" />
  </ItemGroup>
```
直接生成客户端项目即可根据proto生成对应的类文件。
然后通过如下代码调用：
```csharp
static async Task Main(string[] args)
{
    var channel = GrpcChannel.ForAddress("https://localhost:5001");
    var client = new Greeter.GreeterClient(channel);
    var reply = await client.SayHelloAsync(new HelloRequest { Name = "张三" });
    Console.WriteLine(reply.Message);
}
```

运行程序：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190928210331441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9idzgzMQ==,size_16,color_FFFFFF,t_70)

## proto中标量值类型与C#对应关系
|.proto Type| C# Type |Notes |
|--|--|--|
|  double| double| |
|  float| float| |
|  int32| int|Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. |
|  int64| long|Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. |
|  uint32| uint| Uses variable-length encoding.|
|  uint64| ulong| 	Uses variable-length encoding.|
|  sint32| int|	Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. |
|  sint64| long|Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. |
|  fixed32| uint|Always four bytes. More efficient than uint32 if values are often greater than $2^{28}$. |
|  fixed64| ulong| Always eight bytes. More efficient than uint64 if values are often greater than $2^{56}$ .|
|  sfixed32| int| Always four bytes.|
|  sfixed64| long| Always eight bytes.|
|  bool| bool| |
|  string| string| A string must always contain UTF-8 encoded or 7-bit ASCII text, and cannot be longer than $2^{32}$.|
|  bytes| ByteString| May contain any arbitrary sequence of bytes no longer than $2^{32}$.|
