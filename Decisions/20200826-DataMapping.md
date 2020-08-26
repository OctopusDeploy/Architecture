# Overview

- **Subject**: Implementing A New Data Mapping Approach  
- **Decision date**: [when was the decision made]

# Executive Summary

To replace the existing implicit data mapping approach with an explicit approach. The new proposed approach would not only provide better code navigation/discoverability/usage analysis, debugging experience, and other benefits but offer us a good opportunity to address some of the tech debt(s) caused by the existing approach such as violation of single responsibility and incorrect use of `ILifetimeScope` etc, to improve our overall code quality. 

# Detail

## Why do We wanna move away from implicit DataMapping (ResourceMapper)?

- To eliminate any unseen/unpredictable bug(s) that could be caused by renaming properties of either Domain Model(s) or wire format (Resources)
- To improve code navigation/discoverability/usage analysis as `Show Usage` cannot help us much e.g. Some required properties of wire format (Resources) have no usage at all, which may encourage developers to remove them. Moreover, some tool(s) such as static analysis of property usage may provide misleading outcome
- To improve debugging experience e.g. Instead of stepping into some common/generic code implementation and looping through all the properties until reaching the specific property we are after, we could simply toggle a breakpoint at an executable line of code which corresponds to the property mapping
- To remove the complexity of our implicit DataMapping such as how we make use of those special attributes (`Writeable`, `WriteableOnCreate` and `NotReadable`) to control Create, Read and Update operations, and how we construct those auto property mappings etc, so it's easier for developers to understand

Apart from the above reasons, there are some existing problems/tech debts with our current implementation we would also like to avoid doing again. More details can be found [here](https://docs.google.com/document/d/1rb-dUOMsf7R6B5KjPbnAggJSn3p_rcqk3KokXWuAiQE/edit?usp=sharing).

## Assumptions

- We currently only employ this new `DataMapping` "on the way out", going from domain to wire format in 'the new world' of ASP.NET Core.
(While the above assumption applies here, there isn't anything stopping us from using the proposed mapping shape elsewhere - it just isn't the focus of this PR)

## Principals

- We want to move toward writing explicit mapping code, and away from implicit/magic mapping (like [ResourceMapper](https://github.com/OctopusDeploy/OctopusDeploy/blob/de145f341f540247e8356e9db5452585e42e60df/source/Octopus.Server/Web/Mapping/ResourceMapper.cs#L17))
- We want to make the new data mapping approach simple by only taking one single responsibility: mapping, so it's easier for any developers to understand
- We want to increase the discoverability of each concrete mapping
- We want to be able to migrate to explicit mappings easily
- We want to make the new data mappings able to inject any dependencies they need
- We want the new data mappings to comply with convention(s) like naming and inheritance etc
- We want tests to be able to optionally supply stub mapping implementations or mock the implementation for tests of mapping dependents, such as ASP.NET Core controllers
- We want each data mapping to represent a 1-1 relationship between an input and an output
- We want each data mapping to be defined in one single class only
- We want to improve the debugging experience

## Exclusions

- Where should this new `DataMapping` implementation live? For example, where might it lives so it can be referenced by `Octopus.Server` & `Sashimi`
- Data projection from multi-sources

## Brief Summary

The new proposed approach is using explicit concrete mapping implementation to map between one source and one target. Each mapping derives from a common abstract class `DataMapping<TSource, TTarget>` for consistency. A Spike PR can be found [here](https://github.com/OctopusDeploy/OctopusDeploy/pull/6686), and please refer to `TestController` as a concrete example in the PR, it accepts a direct dependency of the concrete data mapping implementation `TestToTestResponseDataMapping` which aligns with most of the above principals apart from the convention and migration ones.

To enforce our convention principal, we introduce the `DataMappingConventionFixture` to ensure every `DataMapping` implementation complies with the desired naming pattern and inheritance hierarchy.

For our migration principal, it could be achieved differently depending on when the migration needs to happen. 
More details will be discussed in the `Considerations` section.

## Considerations

To move away from the implicit `DataMapping` we have with `ResourceMapper`, we have considered the following explicit approaches:

1. Use a static method for each mapping
2. Introduce a new explicit data mapping approach, where each concrete mapping defines/implements the mapping between one source and one target. Each mapping derives from a common abstract class `DataMapping<TSource, TTarget>` for consistency (as demonstrated in the above PR)
3. Another explicit data mapping approach mentioned at this [discussion thread](https://octopusdeploy.slack.com/archives/CTZT49JFJ/p1600048892014600) is instead of sharing the same shape (interface or generic methods) between each concrete mapping, we would like each concrete mapping to be able to pass any contextual information it needs for mapping e.g. `Map(VcsRunBook runBook, string gitref, string spaceId, string projectId, string runbookId)` etc. However, we are still uncertain about how the current VCS model and resource would be modelled as we currently have two different implementation(s) for VCS modelling e.g. `Project` and `VcsRunbook`

We believe the second option is better than the others, because:

1. Better consistency as each `DataMapping` has a similar shape so it's easier for developers to understand
2. Better dependency graph as each `DataMapping` takes care of its dependencies
3. Better testability as each `DataMapping` can be mocked/stubbed for testing

### Migration Concern(s)

Migration concern(s) could be addressed differently depending on when it needs to happen:
1. Migrate independently in all Nancy responder(s) (before `Nancy Modules to ASP .Net Core Controllers` migration)
2. Migrate along with or after `Nancy Modules to ASP .Net Core Controllers` migration

#### Migrate independently in all Nancy responder(s)

Unfortunately, we are currently unable to use concrete explicit data mapping in `Nancy` world, because there is a generic method `Collection<TResource>` in `CustomResponder`, which is trying to map each item from type `object` to a generic type `TResource`. 

Thus, to migrate mappings independently, we need to implement a temporary solution. One of the possible solutions is introducing a new virtual method `MapModelToResource<TResource>(object model)` in `CustomResponder` along with an extra interface `IDataMapper` (containing a generic method signature `TTarget Map<TTarget>(object source)`) and its implementation. With these, responders may now override the mapping and use a new explicit `DataMapping` if desired. (Please refer to the above [PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/6686) and [a POC PR for migrating TaskMapping](https://github.com/OctopusDeploy/OctopusDeploy/pull/6913) for more details)

Alternatively, we could also make `IDataMapper` smarter so it knows when to fall back to use `ResourceMapper` when it cannot find the corresponding explicit data mapping between the given source and target type. 

We can safely remove this temporary solution once we have successfully migrated all our `Nancy` responders.

#### Migrate along with or after Nancy Modules to ASP .NET Core Controllers migration

Migrating along with or after `Nancy Modules to ASP .NET Core Controllers` migration doesn't require a temporary solution, because we can still use the current `ResourceMapper` and safely migrate a specific mapping to the new explicit approach after converting all responder(s) in a nancy module to controllers completely. However, if we only wish to make use of ASP .NET controllers and still invoke the existing responders underneath each controller's action as the first step of the Nancy migration, we may still need to consider employing the above temporary solution.

### Dependency Injection

This is one of the existing problems we have with the `ResourceMapper`, as it takes direct dependency of `ILifeTimeScope` and passes the scope down to `ResourceMapping` for the following purpose(s):

- Switching space context for `SpaceScopedRoutes`
- Resolving any `IEnricher`(s) for after-mapping operation(s)

For best practice, we should try our best to avoid taking `ILifeTimeScope` directly, because:

1. It allows developers to manipulate/mess with the current `ILifeTimeScope` unnecessarily
2. It creates a tightly-coupled relationship between `Autofac` and our codebase everywhere unnecessarily  

To address this, we prefer the second or third option over the static method approach as they allow developers to inject any component(s)/service(s) required for mapping the data from source type to the desired target shape.

As for the scope and lifetime of each `DataMapping`, we have considered the following two option(s):

1. Register each `DataMapping` as `InstancePerLifetimeScope`
2. Register each `DataMapping` as `SingleInstance` then use [Autofac Dynamic instantiation](https://autofaccn.readthedocs.io/en/latest/resolve/relationships.html#dynamic-instantiation-func-b) to resolve the dependencies

We are currently using the first option to start with for simplicity. 

### Discoverability

It's often very painful for developers to discover a file/class/interface in a very large codebase, especially when we use/inject an interface instead of the concrete type.
To ease the pain and provide better maintenance/development experience to not just us but new developers, we would like to:

- Always inject the concrete `DataMapping` type in our ASP.NET Core controller (which will be further enforced when we move `IDataMapping` interface into a separated library and hides its accessibility)
- Use `DataMappingConventionFixture` to ensure every `DataMapping` complies with our naming pattern e.g. **{SourceTypeClassName}To{TargetTypeClassName}DataMapping**

### Convention(s)

The following convention(s) are introduced for better discoverability and consistency:

- Naming Convention
- Inheritance Hierarchy Convention

They are currently enforced in [DataMappingConventionFixture](https://github.com/OctopusDeploy/OctopusDeploy/blob/f05d57024bd2a2e0db9c5dccf2b64c189aad0ca2/source/Octopus.Tests/Server/Web/Mapping/DataMappingConventionFixture.cs) within `Octopus.Server`.
However, how should we enforce these convention(s) when:

- `DataMapping` has been extracted to a common library
- And some concrete `DataMapping` types need to be defined somewhere else like in `Sashimi`

There are a few options that may resolve this issue:

- Scan all integrated assemblies in `Octopus.Server` for all `DataMapping(s)`
	- Pros:
		- Better maintenance/consistency as no duplicated convention tests in different projects
	- Cons:
		- Late failure as we will have to run unit tests in `Octopus.Server` to know something isn't right somewhere else
		- Breaks modularity
- Having the same convention tests in different projects where needed
	- Pros: 
		- Fail fast
		- Remains modularity
	- Cons:
		- Poor maintenance/consistency as duplicated convention tests everywhere 
- Introduce a new `Map` method in each `AccountTypeProvider` and `DeploymentTargetTypeProvider`, and inject the providers into the `DataMapping` in `Octopus.Server`
	- Pros:
		- No need for convention testing for this kind of mapping
		- Remains modularity
		- No need to extract `DataMapping` to a common library
	- Cons:
		- Poor discoverability as compared to what we are trying to propose here
		- Inconsistency 

**FEEDBACK NEEDED:** Is there a better option to tackle this issue? And are there any other flaws in the above option(s)?

### Testability

This is another benefit given by the inheritance hierarchy. 
To gain better testability, each `DataMapping` is forced to inherit from a generic type of `DataMapping<,>` which has a public abstract method `TTarget Map(TSource source)`. This abstract method will then make each concrete derived `DataMapping` type mockable.

e.g.
```
var mapping = Substitute.For<TestToTestResponseDataMapping>();
mapping.Map(Arg.Any<Test>()).Returns(new TestResponse("2", "2", "2"));
```

Note: While we can mock/stub each `DataMapping` implementation, whether to adopt this pattern in our tests may still vary depending on the type of tests we are trying to conduct.

## Decision

TBA

## Data Sources

Sources listed below might provide additional context but keep in mind that they can disappear at any time.

- [Initial Investigation of the existing ResourceMapper](https://docs.google.com/document/d/1rb-dUOMsf7R6B5KjPbnAggJSn3p_rcqk3KokXWuAiQE/edit?usp=sharing)

