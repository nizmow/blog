---
title: "Entity Framework Core, performance and required properties"
date: 2019-11-28T14:47:16+01:00
draft: true
---

Entity Framework Core 3.0 was recently released with a lot of changes from 2.2, many of which were [breaking](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes) - yup, that's a huge list. Since I was updating projects to .NET Core 3.0, I decided to upgrade a small project to EF Core 3.0 and see what I'd have to go through for an upgrade against larger projects.

I ran into a breaking change straight away, and it's the one I've seen the most activity about: [LINQ queries are no longer evaluated on the client](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes#linq-queries-are-no-longer-evaluated-on-the-client). _Most_ of the time this happens it's caught as a run-time error when a query that can't be evaluated as a single SQL statement is executed, but sometimes it can be more subtle. Fortunately for me it was caught with a unit test using SQLite in-memory mode[^1].

There have been "breaking" changes that result in performance regressions, and a bit of digging around revealed some projects on GitHub having issues with ["Eager loading of related entities now happens in a single query"](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes#eager-loading-single-query) - basically, eager loading would previously issue as a sequence of `SELECT` statements, and in EF Core 3.0 it's a bunch of joins which can be MUCH slower. It actually seems to be a side effect of the issue above, the new EF policy of doing no evaluation on the client.

After fixing my small issue, I ran up the application and ran some basic integration tests. Everything seemed fine, so I dropped it onto a test environment to push some more load through, and I quickly realised performance had taken a _dramatic_ turn for the worse. The database use in this application was very simple, but in troubleshooting I ended up creating a scenario that was as simple as possible. The application did nothing more than execute a bunch of `SELECT x FROM y WHERE z`, and performance was still terrible. Something was up.

[^1]: I strongly recommend testing your data layer as close as possible to the metal as you can - things like this are why!
