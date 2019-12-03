---
title: "Receiving a message on all service instances in MassTransit"
date: 2019-11-19T13:33:09+02:00

categories:
- development
---

[MassTransit](https://masstransit-project.com) is an excellent distributed application framework for .NET that I intend to write more about later. This post documents a solution to a problem you'll probably come across when developing distributed applications with MassTransit.

MassTransit is built upon a message broker, usually RabbitMQ or Azure Service Bus. MassTransit "consumers" are attached to message broker queues, to which messages are delivered according to the rules of the message broker. One thing about this that's handy is messages put into a queue are delivered to a consumer only once,[^1] which means you can start multiple instances of a service and, since they'll be using the same queue name, you'll get load balancing "for free".[^2]

However, in some occasions you might want a message to be received by _all_ of your service instances, and a likely example of this is you broadcast a message that indicates a service might want to clear its cache. With a normal consumer configured to listen to that message, only one of your service instances will receive it and you'll be left with a happily running instance with stale cache.

Luckily, MassTransit provides a simple way to configure temporary endpoints, exclusive to each instance. All you need to do is configure your receive endpoint without specifying a queue name:

``` cs
containerBuilder.AddMassTransit(r =>
{
    r.AddBus(context => Bus.Factory.CreateUsingInMemory(cfg =>
    {
        // a 'regular' endpoint
        cfg.ReceiveEndpoint("processing_queue", e => e.Consumer<ProcessConsumer>(context));

        // a temporary endpoint
        cfg.ReceiveEndpoint(e => e.Consumer<InvalidateCacheConsumer>(context));
    }));
});
```

Once starting your application you'll see a queue named something like `MYHOST_dotnet_endpoint_8ymoyyn7ywybkygzbdms35zan8`. This looks messy, but it's all just to avoid queue naming collisions across services -- as you can see, it's prefixed with the host name of the server, some metadata, and a random(ish) string. It'll be set to auto-delete, meaning it'll disappear when your service does. After all, no point in receiving cache invalidation messages when your service instance isn't running.

[^1]: Actually, [at least once](https://www.cloudcomputingpatterns.org/at_least_once_delivery/).
[^2]: Well, for the low, low cost of running a message broker.