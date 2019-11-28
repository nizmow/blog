---
title: "Why you should (or shouldn't) consider MassTransit for distributed applications"
date: 2019-11-27T14:46:09+02:00
draft: true
categories:
- development
---

I wrote a brief post before with a small tip about [MassTransit](https://masstransit-project.com), and I intend to write more, so I thought I'd write down a quick note about what it is and what it's for. It's a question I'm frequently asked when I talk with other developers and honestly I'm a little surprised more people don't use it.

MassTransit is a framework for writing distributed applications in .NET. You can probably think about it in a similar way you think about ASP.NET, except instead of passing data as HTTP payloads it uses a message broker (like RabbitMQ or Azure Service bus) and passes messages. If you're considering using HTTP to communicate between your applications (microservices, communication between monoliths, even, perhaps, communication _within_ monoliths) you should consider using a message broker and MassTransit instead.

## ADVANTAGES

* The ability to easily publish events using a pub-sub type of architecture. The post I

* Message durability -- messages are queued at the endpoint