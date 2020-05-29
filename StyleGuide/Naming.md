# Naming

This section documents the conventions used to name method, classes and other elements. 

Consistent naming conventions aid the reader by reducing the cognitive load required to understand the code. They aid the consumer of an API by being able to make assumptions about the contract.

## Retrieval Methods

This subsection outlines the conventions used when retrieving one or more objects from a collection or store. In particular for the `*DocumentStore` classes. 

In general:
- Omit redundant description the method name if the behaviour is obvious from the parameters. e.g. `FindBy(ProjectGroup)` instead of `FindByProjectGroup(ProjectGroup)`
- Pass document IDs as their TinyType (see the [Id passing](Documents.md#Id%20Passing) convention)
- Follow the [Collection](Collections.md) conventions for parameters and return types

*The documents `Project` and `WorkerPool` are used for illustration.*

| Return | Method | Description |
|-|-|-|
| Project | **Get**(ProjectId) | Gets a single item by it's primary key, the equivalent to Nevermore's **LoadRequired**. Throws an exception if not found. |
| ICollection&lt;Project&gt; | **Get**(ICollection&lt;ProjectId&gt;) | Gets multiple items by their primary key. Throws an exception if not all values are found. |
| WorkerPool |  **Get**Default() | Gets a single item by a secondary unique key or attribute that is expected to exist (e.g. there should always be a default worker pool) |
| Project? | **Find**(ProjectName) | Returns the item that matches the parameter which is a unique key, otherwise null, the equivalent to Nevermore's **Load**  |
| ICollection&lt;Project&gt; | **Find**(ICollection&lt;ProjectId&gt;) | Gets multiple items by their primary key, ignoring invalid ids. |
| Project? | **FindBy**Name(string) | Returns the item that matches the parameter which is a unique key (otherwise null) and the parameter type is not specific enough  |
| IReadOnlyList&lt;Project&gt; | **Find**By(ProjectGroup) | Returns all items that match the parameter. Never returns null  |
| bool | **AreAny**Active | Do any items match |
| bool | **AreAll**Active | Do all items match |
| Dictionary&lt;ProjectName, Project&gt; | **Map**ByName() | Dictionary keyed by a unique key with the value as the type |
| Dictionary&lt;ProjectId, ProjectName&gt; | **Map**NameById() | Dictionary keyed by a unique key with property (or subset of properties) of the type |
| WorkerPoolId? | **Resolve**Id(WorkerPoolIdOrName) | Look up a unique key based on another unique key. Returns null of there is no match is not found.
