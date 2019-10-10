---
title: gRPC的四种调用方式
categories: ASP.NET Core
comments: true
date: 2019/10/10
updated: 2019/10/10
tags:
    - ASP.NET Core
    - gRPC
---

在 {% link 上一篇 https://zhaobingwang.gitee.io/2019/09/28/ASP.NET%20Core/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E4%B9%8B%E9%97%B4%E9%80%9A%E4%BF%A1%E7%9A%84%E9%80%89%E6%8B%A9%E4%B9%8BgRPC/ %}  介绍了`gRPC`的使用场景及基本使用，本文将介绍`gRPC`的四种调用方式。

## 一元调用
> 普通RPC调用，客户端带一个请求对象进行调用，服务端返回一个响应对象。

- proto
```ts
syntax = "proto3";

option csharp_namespace = "GrpcDemoServices";

package Greet;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply);
  rpc SayHellos (HelloRequest) returns (stream HelloReply);
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

- Client C# Code
```csharp
private static async Task UnaryStreamCallExample(Greeter.GreeterClient client)
{
    var reply = await client.SayHelloAsync(new HelloRequest { Name = "张三" });
    Console.WriteLine(reply.Message);
}
```
- Server C# Code
```csharp
public class GreeterService : Greeter.GreeterBase
{
    //private readonly ILogger<GreeterService> _logger;
    //public GreeterService(ILogger<GreeterService> logger)
    private readonly ILogger _logger;
    public GreeterService(ILoggerFactory loggerFactory)
    {
        _logger = loggerFactory.CreateLogger<GreeterService>();
    }

    public override Task<HelloReply> SayHello(HelloRequest request, ServerCallContext context)
    {
        _logger.LogInformation($"Sending hello to {request.Name}");
        return Task.FromResult(new HelloReply
        {
            Message = "Hello " + request.Name
        });
    }
}
```

## 服务端流式调用
> 客户端带一个请求对象进行调用，服务端返回多个响应对象

*proto同上*

- Client C# Code 
```csharp
private static async Task ServerStreamingCallExample(Greeter.GreeterClient client)
{
    var cts = new CancellationTokenSource();
    cts.CancelAfter(TimeSpan.FromSeconds(3.0));

    using (var call = client.SayHellos(new HelloRequest { Name = "张三" }, cancellationToken: cts.Token))
    {
        try
        {
            await foreach (var response in call.ResponseStream.ReadAllAsync())
            {
                Console.WriteLine($"reply:{response.Message}");
            }
        }
        catch (RpcException ex) when (ex.StatusCode == StatusCode.Cancelled)
        {
            Console.WriteLine("Stream cancelled.");
        }
    }
}
```
- Server C# Code 
```csharp
public class GreeterService : Greeter.GreeterBase
{
    private readonly ILogger _logger;
    public GreeterService(ILoggerFactory loggerFactory)
    {
        _logger = loggerFactory.CreateLogger<GreeterService>();
    }

    public override async Task SayHellos(HelloRequest request, IServerStreamWriter<HelloReply> responseStream, ServerCallContext context)
    {
        var i = 0;
        while (!context.CancellationToken.IsCancellationRequested)
        {
            var message = $"How are you {request.Name}?{++i}";
            await responseStream.WriteAsync(new HelloReply { Message = message });
            await Task.Delay(1000);
        }
    }
}
```
## 客户端流式调用
> 客户端带有多个对象进行请求，服务端返回一个响应对象

- proto
```ts
syntax = "proto3";

option csharp_namespace = "GrpcDemoServices";

package Count;

service Counter{
    // Increment count through multiple counts
    rpc AccumulateCount (stream CounterRequest) returns (CounterReply);
}

// The reqeust message containing the count to increment by
message CounterRequest {
    int32 count = 1;
}

// The response message containing the current count
message CounterReply {
    int32 count = 1;
}
```

- Client C# Code
```csharp
private static async Task ClientStreamingCallExample(Counter.CounterClient client)
{
    using (var call = client.AccumulateCount())
    {
        for (int i = 0; i < 3; i++)
        {
            var count = RNG.Next(5);
            Console.WriteLine($"Accumulating with {count}");
            await call.RequestStream.WriteAsync(new CounterRequest { Count = count });
            await Task.Delay(2000);
        }
        await call.RequestStream.CompleteAsync();

        var response = await call;
        Console.WriteLine($"Count:{response.Count}");
    }
}
```

- Server C# Code
```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IncrementingCounter>(); // Configure IncrementingCounter as a singleton
}
public class IncrementingCounter
{
    public int Count { get; private set; }

    public void Increment(int amount)
    {
        Count += amount;
    }
}
public class CounterService : Counter.CounterBase
{
    private readonly ILogger _logger;
    private readonly IncrementingCounter _counter;

    public CounterService(IncrementingCounter counter, ILoggerFactory loggerFactory)
    {
        _counter = counter;
        _logger = loggerFactory.CreateLogger<CounterService>();
    }

    public override async Task<CounterReply> AccumulateCount(IAsyncStreamReader<CounterRequest> requestStream, ServerCallContext context)
    {
        await foreach (var request in requestStream.ReadAllAsync())
        {
            _logger.LogInformation($"Incrementing count by {request.Count}");
            _counter.Increment(request.Count);
        }
        return new CounterReply { Count = _counter.Count };
    }
}
```

## 双向流式调用
> 客户端带多个请求对象进行调用，服务端返回多个响应对象

- proto
```ts
syntax = "proto3";
option csharp_namespace = "GrpcDemoServices";
package Race;

service Racer {
  rpc ReadySetGo (stream RaceMessage) returns (stream RaceMessage);
}

message RaceMessage {
  int32 count = 1;
}
```

- Client C# Code
```csharp
private static async Task BidirectionalStreamingExample(Racer.RacerClient client)
{
    var headers = new Metadata { new Metadata.Entry("race-duration", RaceDuration.ToString()) };

    Console.WriteLine("Ready, set, go!");
    using (var call = client.ReadySetGo(new CallOptions(headers)))
    {

        // Read incoming messages in a background task
        RaceMessage? lastMessageReceived = null;
        var readTask = Task.Run(async () =>
        {
            await foreach (var message in call.ResponseStream.ReadAllAsync())
            {
                lastMessageReceived = message;
            }
        });

        // Write outgoing messages until timer is complete
        var sw = Stopwatch.StartNew();
        var sent = 0;
        while (sw.Elapsed < RaceDuration)
        {
            await call.RequestStream.WriteAsync(new RaceMessage { Count = ++sent });
        }

        // Finish call and report results
        await call.RequestStream.CompleteAsync();
        await readTask;

        Console.WriteLine($"Messages sent: {sent}");
        Console.WriteLine($"Messages received: {lastMessageReceived?.Count ?? 0}");
    }
}
```

- Server C# Code
```csharp
public class RacerService : Racer.RacerBase
{
    public override async Task ReadySetGo(IAsyncStreamReader<RaceMessage> requestStream, IServerStreamWriter<RaceMessage> responseStream, ServerCallContext context)
    {
        var raceDuration = TimeSpan.Parse(context.RequestHeaders.Single(h => h.Key == "race-duration").Value);

        // Read incoming messages in a background task
        RaceMessage? lastMessageReceived = null;
        var readTask = Task.Run(async () =>
        {
            await foreach (var message in requestStream.ReadAllAsync())
            {
                lastMessageReceived = message;
            }
        });

        // Write outgoing messages until timer is complete
        var sw = Stopwatch.StartNew();
        var sent = 0;
        while (sw.Elapsed < raceDuration)
        {
            await responseStream.WriteAsync(new RaceMessage { Count = ++sent });
        }

        await readTask;
    }
}
```

## 附
所有代码均可在[Github](https://github.com/zhaobingwang/sample/tree/master/src-demo)仓库下找到，本文中的代码在`GrpcClient`和`GrpcServer`目录下。