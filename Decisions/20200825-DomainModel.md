# Overview

- **Subject**: Implementing A Strict Domain Model (and related concerns)
- **Decision date**: [when was the decision made]

# Executive Summary

TBA

# Detail

## Assumptions

- all new endpoints created in Octopus Server will be ASP.NET Core controllers, and we will be actively migrating away from our legacy Nancy responder infrastructure. Nancy is no longer actively maintained, so this is a desirable direction for Octopus Server to take.

## Principals

- We want our core model to be simpler to reason about
- We want our core model to be safer to interact with
- We want less bugs that were caused by misunderstanding of what state the core model might be in
- We want less magic in how operations flow from the API boundary to the database and back
- We want it to take less time for new developers to be able to discover and understand server code paths

## Exclusions

**ASP.NET Core**
New components introduced in the worked example provided in this document sit on new ASP.NET Core infrastructure. Specifics around the best abstractions for controllers, the migration strategy from Nancy to ASP.NET Core, or other ASP.NET Core pipeline specific considerations, are outside of the scope of this document.

**Resource Mapping**
This document sees us move from utilising the ResourceMapper to directly facilitating Resource to Domain Model mapping, but it does not directly address moving away from the mapper, or moving away from it when mapping from Domain Model => Resource when translating back to wire format on "the way out". This will be the focus of [another decision record]() TBA.

**Domain Events**
Domain event infrastructure helps us create rich, decoupled domain models that can evolve and scale in healthy ways. This PR does not introduce any new domain event infrastructure or considerations, although it is a topic we want to tackle soon after agreeing on the core domain approach and abstractions.

## Overview

[This PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/6834) works through an end-to-end example of refactoring the `Tenants` `PUT` and `POST` endpoints to use new components that align with the above principals.

Our objective is to have code that is more easily verified as being correct, and to reduce the number of invariants developers need to consider for any code path between our API surface and our database.

The tools we are going to use to achieve this are:

- [C# 8 Nullable Reference Types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/nullable-reference-types)
- [Tiny Types](https://github.com/OctopusDeploy/TinyTypes/)
- [Fluent Validation](https://docs.fluentvalidation.net/en/latest/aspnet.html)
- Domain Driven Design

These tools are applied across two primary problem areas:

### Verifying correctness of client payloads

We want our Resource model's types to correctly reflect the information they contain.

To achieve this, we use:

- ASP.NET Core's nullable reference type awareness, which will validate that information has been supplied for non-nullable fields during model binding.

- Tiny Types, which we have a [custom serializer](https://github.com/OctopusDeploy/TinyTypes/blob/main/source/TinyTypes.Json/TinyTypeJsonConverter.cs) for, to ensure recieved information is in the expected format.

- Fluent Validation, for when we have more complex validation needs that are still restricted to validating the structure of the information in a single model, [example](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Validation/DaysPerWeekScheduledTriggerFilterValidator.cs).

Once the model is bound and validated and is available to our controller to consume, we have verified the integrity of the data it contains, and that that data aligns with the type that is defined.

### Verifying correctness of domain operations

Once we have verified the client has supplied us a correct payload for a given operation, we want to facilitate the interaction with the core domain model.

The rule here is the _domain model must enforce its own invariants_.

To achieve this, we use:

- Nullable reference types, to ensure we only store information that is appropriate for the model.

- Tiny types, which help strongly align requirements for data between our resources and domain model.

- Encapsulation within our core models, to ensure they remain internally consistent.

## Considerations

### Resource Validation

A number of options for resource validation were considered, considering the following constraints:

- We want to use nullable reference types and TinyTypes throughout Octopus Server
- We want to work with ASP.NET Core model binding, as we will be working with ASP.NET Core controllers
- We want to keep using JSON.NET for serialization
- We want our resource types to reflect the data we _expect_ them to contain so that we can confidently utilize them as soon as they are available to our endpoints

The options considered are [documented here](https://docs.google.com/document/d/1JVrLNPUf9h5TK9L6jxbL5KaoY3gGWUqp-flycQCFDsY/edit).

We settled on the approach detailed above in **Verifying correctness of client payloads**, which leverages ASP.NET's awareness of nullable reference types, tiny types, and fluent validation to ensure resources can be validated to a point that any consumption of them once they are presented to an endpoint is safe.

**FEEDBACK WANTED** are there scenarios in which the data may "pass the keeper" and we end up with a resource that has data that isn't congruent with the defined type (i.e. nulls in non-nullable fields)?

### Resource and Endpoint Modelling

Our existing resource model lends itself to being "progressively enhanced" with data within our Nancy request responders.

Typically, 'most' of the resource model would be bound and validated from the request body, but we would also add information that arrived on the request path or query string, such as the spaceId, or the resourceId, driven by the hypermedia we issue. This would be done step-by-step in a responder.

This progressive enhancement does not gel with nullable reference types, which have a strict requirement to initialize non-null fields in constructors (or explicitly set them to null, which is yuck).

Given our ideal approach to resource modelling uses nullable reference types and ASP.NET's awareness of them during model binding, we have taken a step toward _Asymmetric Resource Models_.

We have broken the resource model for `Tenant` into two components:

- `TenantWriteResource` is used to create or update tenants, and does not contain Id or SpaceId properties (as these are available as query parameters), or hypermedia links
- `TenantReadResource` is used to provide tenant information to clients - it contains all tenant information, along with identifiers and hypermedia links

Example: [TenantResource](https://github.com/OctopusDeploy/OctopusDeploy/blob/0047fb8c59f47c9661c911329fbbdd624f7cc026/source/Octopus.Core/Resources/TenantResource.cs#L14-L36) vs [TenantReadResource](https://github.com/OctopusDeploy/OctopusDeploy/blob/0047fb8c59f47c9661c911329fbbdd624f7cc026/source/Octopus.Core/Resources/TenantResource.cs#L38-L76).

There are some basic convention tests established in server within `ResourceConventionsFixture` to ensure `WriteResource`s remain true subsets of `ReadResource`s.

This aligns with the direction our web client is taking, which currently has the concept of a `TNewResource`. This uses typescript's `Omit` to remove properties from the full resource definition that are not needed when creating resources. We could align our client resources by providing changing `TNewResource` to `TWriteResource`. Modify methods could accept `TWriteResource & TLinks`, which read resources retrieved from the server should match anyway thanks to the ✨of duck-typing, making round-tripping resource changes simple.

**FEEDBACK WANTED** is this a sensible first step, or should we jump in with both feet and introduce seperate models for each endpoint to completely decouple their concerns?

### Domain Modelling

We have taken the approach of attempting to use the compiler and type system as much as possible to enforce correctness, leaving the domain to focus on domain invariants (business rules!).

This means we are not militant in applying precondition checks to incoming information - nullable reference types and tiny types should take care of the bulk of this type of "infrastructure concern" validation - things should not be null if we do not expect them to be, and things should not be malformed or empty.

The other type of precondition check we are typically required to do relates to enforcing relational consistency when linking documents to other documents, which is still a necessary requirement in the new world.

When interacting with the domain, if a domain invariant is unsatisifed [example](https://github.com/OctopusDeploy/OctopusDeploy/pull/6834/files#diff-29fae3c0e603cce39e381b742617dc85R120), we will throw a domain exception from the domain model, which can be captured and handled centrally in our `ErrorHandlingMiddleware`.

**FEEDBACK WANTED** are we going to get an acceptable amount of correctness with this approach? What could bite us if we aren't applying precondition checks like `!= null` and `string.isNullOrEmpty` within our domain model?

### Client Feedback

When something goes wrong processing a request, we need to return feedback to clients so they might act upon it.

Feedback will be provided by two mechanisms:

- _Model Binding_ - when verifying the correctness of client payloads a response will be returned that contains all of the errors detected within the payload as a single response. This will allow clients to address all of these errors before proceeding. This is particularly useful in the Octoups web app.

- _Domain Exceptions_ - when a domain invariant is unsatisfied, an exception will be thrown immediately, and processing of the given request will cease. This will mean these types of errors will be revealed "one per request". Typically we would not expect sets of this type of error. The client will have to correct the invariant, and then can attempt to process their request again.

_Client Feedback and User Experience_
There is an implication on user experience here if field names within our domain diverge from those in our resource model - exceptions thrown from the domain may contain information regarding fields that does not line up with resource field names. We anticipate most field-specific validation should be taken care of in the _Model Binding_ mechanism, meaning most exceptions thrown from the domain should not be about specific fields, but about rule violations of various kinds. Hopefully this means the impact on user experience should be minimal.

There is also a strong likelihood that at some point we may want to evaluate a set of invariants all at once, so that we can provide more information back to a user as to what might need to be corrected for an operation to proceed. Throwing exceptions means our core domain model will only ever reveal the first unsatisfied invariant, but also gives us strong guarantees that it will never be persisted in an incorrect state.

If we do want to evaluate a set of invariants for an operation, we will create coordinator domain services that have the responsibility for evaluating the invariants across a set of domain models for an operation - much like our `ExecutionFactory` class does at the moment - that will provide feedback back to the user and prevent domain operations proceeding if they are not satisfied.. This does not mean the core domain models that are being coordinated for the operation will not enforce their own invariants too, they will - by throwing exceptions to ensure no invalid state can be persisted.

**FEEDBACK WANTED** do we think this will provide an adequate client experience? Are there areas of the application where there could be a series of domain invariants that would be unsatisifed where this approach would not be sufficient?

## Decision

TBA

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

- [Nullable reference types and ASP.NET Core model binding](https://octopusdeploy.slack.com/archives/C015Z4DH6R2/p1597989201007900)
