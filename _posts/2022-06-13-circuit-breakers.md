---
layout: post
title: Circuit Breakers to handle failures in serverless applications
subtitle:
gh-repo: daattali/beautiful-jekyll
gh-badge: [star, fork, follow]
tags: [patterns]
comments: true
---

# Introduction

When you are developing micro-services, you might find that there might be some dependencies which could fail or be temporary unavailable. Examples of these dependencies are calls to other services, data stores, etc. These dependencies might be unavailable for multiple reasons such as maintenance, throttling, etc.

The idea behind the circuit breaker pattern is wrapping the dependency, so if there is a consistent failure, we temporary stop trying to call the dependency and instead we can have define an alternative fallback function.

Normally a circuit breaker can be in three different states:

- Closed: we call the dependency as usual. If there are failures, we track them and we open the circuit if a defined threshold is reached.
- Open: instead of calling the dependency, we call the fallback function. If we don't define a fallback function, it will fail early, without calling the dependency. This can prevent overloading an unhealthy dependent service or data store. While the circuit breaker is OPEN, if we reach a timeout, we will change the status to HALF.
- Half (open): if the request is successful, the status will change to CLOSED and it will start processing all incoming requests. However, if the requests are still failing, the CircuitBreaker will change to OPEN.

The following diagram illustrates the different statuses and transitions:

# Proposed Architecture for serverless applications

The Circuit Breaker pattern is a well know pattern and some popular libraries such us Polly (https://github.com/App-vNext/Polly) implement this pattern. In the implementation offered by Polly, the status of the circuit breaker is persisted in memory. This works well for single instance services, but if we are running multiple instances of the service concurrently, you need to share the status of the circuit breaker, so all the instances are consistently handling the circuit breaker. This applies for AWS lambda functions or ECS if we are running multiple nodes in parallel. For lambdas, you also have to bear in mind that if the lambda gets cold, it will erase the memory. For that reason it is important to persist the status of the circuit breaker.

Regarding implementations more focus on serverless, I found this implementation written in Node.js. Since our micro-services are implanted using .NET 6, I couldn't use a node.js implementation. In addition, that implementation is meant to work at lambda level and for my scenario I wanted something more generic, which can be used by any service and any dependency.

Regarding the persistence, in order to have lower latency reading/updating the state, we decided to store the status of the circuit breaker in Amazon MemoryDB, which is based on Redis with persistent storage. The object we store to persist the status is really small and we want to make the storage as performant as possible. That is the reason why Amazon MemoryDB can be a better candidate than DynamoDB for storing circuit breaker objects.

Based on those principles, the proposed architecture is displayed in the following diagram, where a lambda might have two circuit breaker to manage two dependencies and persist the status in Amazon MemoryDB. In the diagram we see a lambda which has two dependencies, Amazon ElasticSearch and a external API, so we can define one circuit breaker for each one. In addition, we can also define a fallback function for each dependency, the failure handling process could be different depending on the dependency.

# .NET 6 implementation

You can find the full source code of this implementation on my GitHub repository: https://github.com/albertocorrales/circuitbreaker-dotnet.

First of all, you need to configure the dependency injection

```csharp
internal static void RegisterCircuitBreakers(this IServiceCollection services, IConfiguration configuration)
{
    services.Configure<ElastiCacheConfig>(configuration.GetSection("CircuitBreaker"));
    services.AddSingleton<ICircuitBreakerFactory, CircuitBreakerFactory>();
    services.AddSingleton<ICircuitBreakerRepository, CircuitBreakerRepositoryElastiCache>();
}
```

For the persistence, we have a generic repository defined by the interface ICircuitBreakerRepository. Then we could implement other persistence storages if we want to. For this implementation, as it was discussed above, we are using MemoryDB, so the implementation of the repository is provided by the class CircuitBreakerRepositoryElastiCache.

In addition, we are injecting the class CircuitBreakerFactory, so we just need to have this dependency in the classes where we are using circuit breakers.

Then you can use it in your Web APIs, Lambdas, etc. For example in a Web API controller, we can it as follows:

```csharp
public class MyController : ControllerBase
{
    private readonly ICircuitBreakerFactory _circuitBreakerFactory;

    public MyController (ICircuitBreakerFactory circuitBreakerFactory)
    {
        _circuitBreakerFactory = circuitBreakerFactory;
    }

    public async Task<IActionResult> MyMethod([FromBody] RequestModel request)
    {
        var circuitBreakerOptions = new CircuitBreakerOptions<ResultModel>("circuit-breaker-id-123", () =>  asyncFunction(request));
        var circuitBreaker = _circuitBreakerFactory.CreateCircuitBreaker<ResultModel>(circuitBreakerOptions);

        return await circuitBreaker.Fire();
    }
}
```

In the model CircuitBreakerOptions, we can configure the following properties:

- CircuitBreakerId (required): id for the circuit breaker. This value has to be unique, but we could also use the same value if for example we want to use the same circuit breaker in two places. (For example, if EventStore is down, it might affect projections and commands).
- Request (required): function to wrap under the circuit breaker.
- FailureThreshold: number of failure executions that should happen until the circuit is OPEN.
- SuccessThreshold: number of successful executions to change the status of the circuit breaker to CLOSED.
- Timeout: number of milliseconds to try a new request since the circuit breaker changed to OPEN.
- Fallback: alternative function to execute if the main function fails or the circuit breaker is OPEN.

On the other hand, the CircuitBreakerModel that we persists, it is defined by the following properties:

- Id: unique id to identify the circuit breaker
- Status: current status of the circuit breaker (open, closed or half)
- FailureCount: number of consecutive failure requests
- SuccessCount: number of consecutive successful requests
- NextAttempt. datetime when the circuit breaker is closed and we should try next time.

# Conclusions

In this article we have seen the importance of managing failures properly in your applications with circuit breakers. In addition, we discussed about the reason why multi-node applications should persist the status of the circuit breakers, so all the instances can work consistently.

Finally, we have seen an implementation for .NET 6, which helps us to manage and persist our circuit breakers. The implementation is flexible and it allows us to define multiple circuit breakers and handle them independently. Furthermore, we can use MemoryDB as persistent storage or implement our own repository.
