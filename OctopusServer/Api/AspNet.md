# Overview

- **Subject**: ASP.NET Controller endpoints for the Octopus Server Web API

# Executive Summary

Octopus Server is migrating from the legacy Nancy web framework to Microsoft's ASP.NET Core WebAPI web framework.

This document provides an _Overview_ of the new API's behaviour and architecture, and then provides some advice on _Developing_ new Controllers.

# Overview

## How Requests Are Served

![Octopus Request Pipeline](https://github.com/OctopusDeploy/Architecture/blob/master/assets/OctopusWebServer.png)

The ASP.NET request processing pipeline (build via ASP.NET's `IHostBuilder`) was introduced in 2020.5, with initial ground broken in [this PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/6288).

Incoming requests are either served by `HTTP.Sys` if on Windows, or `Kestrel` if on Linux.

They are processed through the middleware pipeline declared in the [WebServerInitializer](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/WebServerInitializer.cs), and proceed to a fork at the end of the pipeline.

If an ASP.NET Controller endpoint exists for the HTTP Route + Verb pair coming in, the request will be sent to the Controller to handle. If an ASP.NET Controller endpoint does not exist, the request will be served by Nancy - this is also where unmatched routes will end up.

## Architecture

We have opted for a **One Controller Per Endpoint** structure, meaning for each HTTP Route + Verb, a Controller is created to serve that request. The name of the controller should contain the name of the resource concerned, with a verb that explains what is being done to it (e.g. `Get`, `Create`, `Update`, `Delete`). The public action method on the controller should be named using this same verb.

We have found many benefits of this approach (when compared to resource-centric controllers containing methods for many endpoints). Specifically:

* Controllers do only one thing each
* Controllers only take the dependencies required to do the one thing they need to do
* Potentially unit testable endpoints (if we want them) thanks to fewer dependencies per-controller.
* We make a deliberate break away from controllers _having_ to be resource-aligned, giving more flexibility for controllers to exist that might model aggregate domain operations

Because we are making controllers more explicit and flattening that inheritance heirarchy, it allows them to contain more code without losing clarity. Things would get messy quickly if we had many endpoints per controller, each including some additional amount of code.

Lastly, we feel that this approach aligns well with the **S** in **SOLID** - the Single Repsonsibility Principle - where each controller is responsible for a single thing: servicing a single endpoint.

There are only a couple of non-conforming controllers (See [SetTenantLogoController](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Controllers/Tenants/SetTenantLogoController.cs) for an example), in which `PUT` and `POST` implementations are identical, and so are served within the same Controller.

Controllers are located under the `Octopus.Server.Web.Controllers.` namespace, collected in sub-directories based on the primary domain model they serve interactions for - for example, Tenant related Controllers are collected under `Octopus.Server.Web.Controllers.Tenants`.

We exclusively use [Attribute Routing](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/routing?view=aspnetcore-5.0#attribute-routing-for-rest-apis) to declare routes for our Controllers.

## Mapping and Validation

We rely on [ASP.NET Model Binding](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-5.0) to map incoming request payloads to parameters and models we work with in Controllers. We have created a custom [MultiSource Model Binder](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Infrastructure/Binders/MultiSourceModelBinder.cs) to allow binding a single model from various request sources (Route, query, headers, body).

[ASP.NET Model Validation](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-5.0) is _disabled_ (applied in [this PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/7627)), as we are relying on existing [mapping](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Mapping/ResourceMapper.cs) and [validation](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Validation/IDocumentValidator.cs) mechanisms within Octopus to validate our resources and models.

Domain model validation was previously performed with a combination of `Rules`, `DocumentValidators`, and code within `Responders`. This validation should now be centralized and exercised within domain models themselves, see [Tenant's Validate method](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Model/Tenants/Tenant.cs) for an example.

We anticipate how we shape and validate resources will continue to change, informed by our opinions on [Domain Modelling](https://github.com/OctopusDeploy/Architecture/blob/master/OctopusServer/DomainModel.md) and the progression of our API.

## OpenAPI / Swagger

We are using [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle.AspNetCore) to generate our OpenAPI/Swagger documentation for our API.

> The Swagger document served from `~/api/swagger.json` is produced currently by combining the OpenAPI Document Swashbuckle generates for our ASP.NET Controllers with an OpenAPI Document we create for our existing Nancy endpoints, driven by `MergeNancyOpenAPIPathsFilter` and the `NancyOpenApiPathGenerator`. As we migrate to ASP.NET, the amount of documentation coming from these components will shrink, until we can eventually remove them.

Some of the information presented in the document is derived from attributes we use to mark up our Controller endpoints and parameters. See the [GetAllTenantsController](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Controllers/Tenants/GetAllTenantsController.cs) for an example of the attributes that are expected to adorn all of our Controller endpoints.

These attributes are **mandatory**, as they help our API be a first-class product. They can be considered as essential as labels and information on any page of the Octopus Web UI.

## Guiderails

To help developers remember the parts they need to put in place for each Controller, we have established a suite of [Controller Conventions](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests/Server/Web/Controllers/ControllerConventionsFixture.cs) that inspect all Controllers created and will not pass unless the required structures and attributes are in place.

# Testing

Before migrating an endpoint, we create Integration Tests that exercise the endpoint, using [Assent](https://www.nuget.org/packages/Assent/) to assert on the output. These have been shown to force us to do the (sometimes time-consuming) analysis of endpoints which is necessary for correct migration to ASP.NET. [dotCover](https://www.jetbrains.com/dotcover/) should be used to verify that the test cover all appropriate code paths in the relevant Nancy responder, rules and other related code.

## Scrubbing

Because certain values in an endpoint response (GUIDs, timestamps etc.) can change per invocation we scrub these values automatically before the output is checked. The scruber methods are all in [`ApprovalTestExtensions`](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests.Common/Support/ApprovalTestExtensions.cs).

## Header Testing

Our Assent-based testing approach by default includes HTTP headers. While our tests are not specifically concerned with headers, we've chosen to leave these in, scrubbing values as appropriate. We may in the future want to remove all of the headers from our tests, and extend [`HeadersFixture`](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.IntegrationTests/Server/Web/HeadersFixture.cs) to test all headers.

# Developing

## Creating New ASP.NET Controllers

Creating a new ASP.NET Controller is straight-forward.

Add a new Controller class under the appropriate namespace. Most endpoints will need to be space-scoped, so derive from `SpaceScopedApiController` to ensure your request is established in an appropriate Space Partition. Ensure you add the required ASP.NET and Swashbuckle attributes to the Controller endpoint and any incoming parameters. Running the [Controller Conventions](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests/Server/Web/Controllers/ControllerConventionsFixture.cs) will help you discover anything you may have missed.

When in doubt, the [Tenant Controllers](https://github.com/OctopusDeploy/OctopusDeploy/tree/master/source/Octopus.Server/Web/Controllers/Tenants) were migrated to act as _Lighthouse implementations_, and should be used as references when creating new Controllers if required.

## Migrating Existing Nancy Responders

### Safety Nets

During the migration of the Tenants Nancy Module, performance, wire-compatibility, and swagger-compatibility safety nets were used to ensure our new ASP.NET infrastructure performed as well or better than our existing Nancy infrastructure. We refurbished part of the [Performance Test Suite](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.E2ETests.Performance/Readme.md) to accomplish this testing, along with leveraging the existing [Swagger Fixture](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.IntegrationTests/Server/Web/Api/Swagger/SwaggerFixture.cs).

We found:

- Performance of new ASP.NET Controllers was on average ~25% better (note that this is baseline performance on stable hardware, not under load / variable conditions)
- With correct attribution (enforced by convention tests), Swagger output was improved
- Wire compatibility of responses was as-expected, with some improvement on some existing Nancy-isms (particularly around the handling of chunked encoding)

For future migrations, implementors may want to consider the risk / complexity involved in their particular migration, and consider:

- Existing E2E test coverage that exercises the endpoint(s) in question
- Whether additional coverage utilizing [AssentTestScript] within the [Performance Test Suite](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.E2ETests.Performance/Readme.md) would provide additional confidence

We don't think additional performance testing for regression is required given our observed results during the Tenants migration, unless there is a particular reason to suspect performance might change as a result of the migration.

### Migrating Legacy Responders (Legacy Nancy Request Processing)

Legacy Responders will use one of the Responder registration methods declared on `OctopusNancyModule` (like `Load`, `Create`, or `Modify`), with responders implementing [Responder](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Infrastructure/Api/Responder.cs), and may use `PersistenceRule`s to enforce validation or create other side-effects.

To migrate legacy responders, all `PersistenceRule` code should be re-implemented within the Controller or domain model, and the existing rules deleted ([Example PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/7668)). It is important to consider where in the Responder lifecycle the rules were applied, so that the behaviour can be accurately replicated within the Controller.

### Migrating Custom Responders (Newer Nancy Request Processing)

Custom Responders will use the `CustomAction` registration method on `OctopusNancyModule` and implement `Custom{Action/Create/Modify}Responder`. Rules utilized will implement `IResponderStep`, with a simple 'veto' pattern used for rule enforcement directly from the responder.

To migrate Custom Responders, the default recommendation is taking the `IResponderStep` rules as dependencies to the Controller, and exercising them within the new Controller created.

### Considerations around `ResourceMapper`

TODO

### Routing Table Risks

Nancy and ASP.NET have separate routing tables, and incoming requests are handed to ASP.NET first, and then to Nancy if ASP.NET decides it did not have a route that could handle the request.

This presents the opportunity for a bug, in the following scenario.

```
GET /resource/{id} // migrated to ASP.NET
GET /resource/all // still in Nancy
```

A request to `GET /resource/all` would be mistakenly handled by ASP.NET, as this matches the pattern `GET /resource/all`.

To workaround this issue, we created a custom ASP.NET [route constraint](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-5.0#route-constraint-reference), named [`ExcludeValuesRouteConstraint`](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/RouteConstraints/ExcludeValuesRouteConstraint.cs). This is used in the `HttpGet` (for example) attribute for an ASP.NET endpoint to let us specify values to "exclude" from handling by that ASP.NET endpoint.

A usage example would be:

```
[HttpGet("workers/{id:excludevalues(all, discover)}")]
```

This means that while `GET /workers/foo` would be handled by this endpoint, `GET /workers/all` and `GET /workers/discover` would not be, and would instead fall through to Nancy for handling.

include ExcludeValuesRouteConstraint in docs

### Use of `CancellationToken` in ASP.NET Controllers

There are two ways to get hold of a `CancellationToken` in an ASP.NET Core controller.

* Via the static `HttpContext.RequestAborted` property
* By adding a `CancellationToken` argument to your async public action method(s).

These are [functionally identical](https://odetocode.com/blogs/scott/archive/2018/09/12/cancellationtokens-and-aborted-asp-net-core-requests.aspx), but we decided to mandate use of the `CancellationToken` parameter approach as this leads to easier unit testing of controllers in the future.

A convention test - [`ControllerConventionsFixture.AllAsyncEndpointsMustTakeACancellationToken`](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests/Server/Web/Controllers/ControllerConventionsFixture.cs#L86) - has been added to enforce this.

### Design of builders

To support the creation of test data for our endpoint Assent tests we have created a suite of **resource builders**. These live in the [`Octopus.IntegrationTests.Helpers.ResourceBuilders`](https://github.com/OctopusDeploy/OctopusDeploy/tree/master/source/Octopus.IntegrationTests/Helpers/ResourceBuilders) namespace.

The resource builders are build on the principle that tests should own their own test data, which is set on a builder using `With...` methods. We have created just the methods we need for our tests, with the expectation that future developers can extend the builders as they see fit. If property value is not specified using a `With...` method then the builder will create a random (wherever possible) value for the property.

All builders should have a `Build()` method which returns the resource, and also stores it in the appropriate context. Builders should be designed so that the `Build()` method can be called on a new builder, and a valid resource will be saved and returned.

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

[A Less Confusing API Framework Pitch](https://docs.google.com/document/d/1kZ5pOKBHvkJ2ahva2L_ReYS_TklqAO_679zJSlYYrno/edit)

[Domain Modelling](https://github.com/OctopusDeploy/Architecture/blob/master/OctopusServer/DomainModel.md)

[Domain Modelling Spike](https://github.com/OctopusDeploy/OctopusDeploy/pull/6834)
