---
title: "Entity Framework Core performance and required properties"
date: 2019-12-03T12:47:16+01:00

tags:
- development
- c#
- entity framework
- issue
---

### Background

Entity Framework Core 3.0 was recently released with a lot of changes from 2.2, many of which were [breaking](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes). That's a pretty big list. Since I was updating projects to .NET Core 3.0, I decided to upgrade a small project to EF Core 3.0 and see what I'd have to go through for an upgrade against larger projects.

I ran into a breaking change straight away, and it's the one I've seen the most activity about: [LINQ queries are no longer evaluated on the client](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes#linq-queries-are-no-longer-evaluated-on-the-client). _Most_ of the time this happens it's caught as a run-time error when a query that can't be evaluated as a single SQL statement is executed, but sometimes it can be more subtle. Fortunately for me it was caught with a unit test using SQLite in-memory mode.[^1]

There have been "soft breaking" changes that result in performance regressions, and a bit of digging around revealed some projects on GitHub having issues with ["Eager loading of related entities now happens in a single query"](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-3.0/breaking-changes#eager-loading-single-query). Basically, eager loading would previously issue as a sequence of `SELECT` statements that were merged in-memory, but in EF Core 3.0 it's a bunch of joins which can be MUCH slower. It actually seems to be a side effect of the first issue, the new EF policy of doing no evaluation on the client.

After fixing my small issue, I ran up the application and ran some basic integration tests. Everything seemed fine, so I dropped it onto a test environment to push some more load through, and I quickly realised performance had taken a _dramatic_ turn for the worse -- _at least an order of magnitude slower_ than the EF Core 2.0 version. The database use in this application was trivial, but I still went to the trouble of creating a scenario that was as simple as possible. In this, application did nothing more than execute a bunch of `SELECT x FROM y WHERE z`, and performance was still terrible. Something was up.

### The problem

I checked the logs to work out exactly what SQL was being executed. In the previous "good" version of the code, on EF 2.2, it was the following:

``` sql
SELECT [l].[CorrelationId], [l].[BatchReferenceId], [l].[ContentReferenceId], [l].[CurrentState], [l].[RowVersion], [l].[SourceId], [l].[StoredAt], [l].[Version]
FROM [LifeCycle] AS [l]
WHERE (@__SourceId_0 = [l].[SourceId]) AND (@__ContentReferenceId_1 = [l].[ContentReferenceId])
```

The EF 3.0 code, however, was executing this:

``` sql
SELECT [l].[CorrelationId], [l].[BatchReferenceId], [l].[ContentReferenceId], [l].[CurrentState], [l].[RowVersion], [l].[SourceId], [l].[StoredOrResentAt], [l].[Version]
FROM [LifeCycle] AS [l]
WHERE (((@__SourceId_0 = [l].[SourceId]) AND (@__SourceId_0 IS NOT NULL AND [l].[SourceId] IS NOT NULL)) OR (@__SourceId_0 IS NULL AND [l].[SourceId] IS NULL)) AND (((@__ContentReferenceId_1 = [l].[ContentReferenceId]) AND (@__ContentReferenceId_1 IS NOT NULL AND [l].[ContentReferenceId] IS NOT NULL)) OR (@__ContentReferenceId_1 IS NULL AND [l].[ContentReferenceId] IS NULL))
```

What's going on? Let's start with `NULL`. As you may or may not be aware, in TSQL/SQL Server, `NULL` is not equal to `NULL`. Ever wonder why you have to use `IS NULL` or similar when writing your SQL queries? That's why. Try it for yourself:

``` sql
SELECT IIF(NULL = NULL, 1, 0)
```

Now this:

``` sql
SELECT IIF(NULL IS NULL, 1, 0)
```

Knowing this, and looking closely at the `WHERE` clause of the SQL query, you'll see it's not completely insane. Let's do some formatting:

``` sql
SELECT [l].[CorrelationId], [l].[BatchReferenceId], [l].[ContentReferenceId], [l].[CurrentState], [l].[RowVersion], [l].[SourceId], [l].[StoredOrResentAt], [l].[Version]
FROM [LifeCycle] AS [l]
WHERE 
(
    (
        (@__SourceId_0 = [l].[SourceId])
        AND
        (@__SourceId_0 IS NOT NULL AND [l].[SourceId] IS NOT NULL)
    )
    OR 
    (@__SourceId_0 IS NULL AND [l].[SourceId] IS NULL)
) 
AND 
(
    (
        (@__ContentReferenceId_1 = [l].[ContentReferenceId]) 
        AND
        (@__ContentReferenceId_1 IS NOT NULL AND [l].[ContentReferenceId] IS NOT NULL)
    ) 
    OR (@__ContentReferenceId_1 IS NULL AND [l].[ContentReferenceId] IS NULL)
)
```

### The solution

Entity Framework is just trying to protect us from ourselves! What happens if we pass in a `NULL` for the SourceId parameter, and expect it to match rows in the database with a `NULL` SourceId? In EF 2.2, we'd get no results.[^2] In EF 3.0, however, it would work as expected. But in my case, SourceId is always going to be there, so why this convoluted query? Turns out, I'd got sloppy and neglected to set a `NOT NULL` constraint in my model definition. So, after rectifying that, and a few other properties I'd forgotten...

``` cs
// ...

entityTypeBuilder.Property(x => x.SourceId).IsRequired();

// ...and so on...
```

I re-run my test, the SQL queries look sensible again, and most importantly the performance is back to where it used to be.

Why is this happening, and how did nothing go wrong before? In Entity Framework "full" (not Core!) this behaviour is controlled by the `DbContextConfiguration.UseDatabaseNullSemantics` property, and you can read about it [here](https://docs.microsoft.com/en-us/dotnet/api/system.data.entity.infrastructure.dbcontextconfiguration.usedatabasenullsemantics?view=entity-framework-6.2.0). This appears to have [vanished](https://docs.microsoft.com/en-us/dotnet/api/microsoft.entityframeworkcore.dbcontextoptionsbuilder?view=efcore-3.0) in EF Core and is replaced with what I can only describe as frighteningly unpredictable behaviour, though when I get a chance I will do a little more research to find out why.

I found it surprising that the lack of a constraint in this case would effect performance so significantly, but I think it's a good lesson. It's the case with ORMs, and also with databases in general, that the more information you can give the engine about your data, the less assumptions it has to make and the more performant your queries can be.

[^1]: I strongly recommend testing your data layer as close as possible to the metal as you can - things like this are why!
[^2]: I think, but this really surprises me. Further investigation is required!