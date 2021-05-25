# Persistence

By and large, we adhere to the principle of [persistence ignorance](https://deviq.com/principles/persistence-ignorance).

We intend for our persistence abstraction to completely decouple us from whichever underlying data storage mechanism actually records an entity. **This is not an abstract/purist position.** We actually have multiple persistence mechanisms (notably SQL and Git) and we switch dynamically between them. We also have different layers of caching, entity population, tracking and so on.

**TL;DR: To access an entity, take a constructor-level dependency on an `IDocumentStore<TDocument>`** where `TDocument` is the type of entity you need to access.

# Transactionality

Our API consumers have a not-unreasonable expectation that API calls they make either succeed or fail atomically. To this end, we have a [unit of work](https://martinfowler.com/eaaCatalog/unitOfWork.html) implementation which allows us to coordinate operations across multiple different types of transaction.

We generally have a single unit of work (UoW) per HTTP request. For background/long-running tasks, we have a UoW per logical operation.

NOTE: Our intention is to revisit our long-running `TaskController` pattern to decrease the need for long-lived, in-memory state, but that's outside the scope of this document.

Within a single unit of work, we might touch several different entity types from several different projects. Each of those projects will have
its own persistence settings:

- Some projects might be wholly in the database.
- Some projects might be in their own Git repository.
- Some projects might be in a Git monorepository shared with several other projects.

All of this means that there is no one "transaction" to commit.

We wrap a logical unit of work around this collection of (0-many) individual transactions. The unit of work is completed or abandoned automatically
via aspects (filters, middleware etc.) and should not need to be addressed directly during the normal course of operations.

# Custom queries

The `IDocumentStore<TDocument>` interface exposes a `.Query()` method which returns an `IQueryable<TDocument>`. The expected approach to writing a custom query is to use a LINQ expression against that `IQueryable<T>`. We have implementations of `IQueryable<T>` for both SQL and Git.

Where a custom query is reused in multiple locations, we use an extension method class on the `IDocumentStore<TDocument>` interface. For example, we have a [reusable mechanism for loading a `Project` via its slug](https://github.com/OctopusDeploy/OctopusDeploy/blob/fb83083950a4bcac1d76ebcb6fc0b9250ba262b3/source/Octopus.Core/Features/Projects/ProjectDocumentStoreExtensionMethods.cs#L16).

Where it is absolutely necessary to drop directly into SQL, there is a `.QuerySql()` method which provides a Nevermore `IQueryBuilder<T>`, which allows for raw SQL expressions to be constructed. This will obviously not work for Git, and may bypass several layers of decoration. **We strongly discourage use of this feature as much as is practical.**

# Space- and project-scoping of document stores

To access a child entity of a project, the document store implementation needs to know which project that entity belongs to in order to decide whether to try to find it in the database or a Git repository - and, if the latter, _which_ Git repository and branch.

Generally speaking, you're not likely to encounter this issue as we have aspects which extract `spaceId`, `projectId` and `gitRef` from HTTP request routes and inject them where appropriate.

You _are_ likely to encounter explicit knowledge of project scoping in the following scenarios:
1. Iterating over collections of projects;
1. During background task execution; and
1. In message bus event handlers (which implement `IEventuallyHandle<TEvent>`).

In these situations, you'll need to make it clear which space and project path you mean when you ask, for instance, for an `IDocumentStore<Runbook>` to fetch a runbook. To do that, take a dependency on an `IProjectScope` and structure your code like this:

```
using (projectScope.Push(spaceId, projectPath))
{
    var runbook = await runbookDocumentStore.GetAsync(runbookId, cancellationToken);
    runbook.DoTheThing();
}
```
You can compose a `ProjectPath` from either just a `ProjectId` or a `ProjectId` and `GitBranch`.

It is not necessary to resolve a document store directly from a container, begin a child `ILifetimeScope` or do any other trickery to change the project scope - just use the existing `IDocumentStore<TDocument>` from within the `using (...)` block and it will use the correct project scope.

# FAQ

## I need to create/read/modify/delete a Foo entity. How should I do that?

Take a constructor-level dependency on an `IDocumentStore<Foo>`.

## I need both a Foo and a Bar. How do I do that?

Take a constructor-level dependency on an `IDocumentStore<Foo>` and an `IDocumentStore<Bar>`.

## Where/how do I commit a transaction?

**TL;DR: That's not a good idea.**

If you take a dependency on the appropriate `IDocumentStore<TDocument>` and simply fetch and modify the relevant entity/entities then the changes will be written to the appropriate location at the completion of the unit of work.

## But I really, really need to commit a transaction!

If you really need control over a transactional boundary, please take a constructor dependency on an `IUnitOfWorkExecutor` and use that.

# History

We have had numerous iterations of persistence, and you will still find older usages throughout the codebase.

You're likely to see dependencies on `IOctopusRelationalStore`, `IOctopusRelationalTransaction`, `IQueryExecutor`, `IFullTableCache` and others. While they have served us to date, they have several shortcomings:

- They tie us to SQL-backed storage.
- They do not support tracking of entities involved in an operation.
    - This makes it difficult to use rich domain models.
    - This also limits our ability to have high-fidelity event brokerage.
- They do not support Git.
- They result in non-transactional behaviour within individual HTTP requests to our API.

If you encounter usages of these patterns, please give some thought to refactoring towards an `IDocumentStore<TDocument>` instead.
