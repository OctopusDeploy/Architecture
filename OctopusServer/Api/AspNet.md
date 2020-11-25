# Overview

- **Subject**: ASP.NET Controller endpoints for the Octopus Server Web API

# Executive Summary

Octopus Server is migrating from the legacy Nancy web framework to Microsoft's ASP.NET Core WebAPI web framework.

This document provides an _Overview_ of the new API's behaviour and architecture, and then provides some advice on _Developing_ new Controllers.

# Overview

## How Requests Are Served

![Octopus Request Pipeline](....\assets\OctopusWebServer.png)

The ASP.NET request processing pipeline (build via ASP.NET's `IHostBuilder`) was introduced in 2020.5, with initial ground broken in [this PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/6288).

Incoming requests are either served by HTTP.Sys if on Windows, or Kestrel if on Linux. They are processed through the middleware pipeline declared in `WebServerInitializer`, and proceed to a fork at the end of the pipeline.

If an ASP.NET Controller endpoint exists for the route + HTTP method pair coming in, the request will be sent to the Controller to handle. If an ASP.NET Controller endpoint does not exist, the request will be served by Nancy - this is also where unmatched routes will end up.

## Architecture

We have opted for a **One Controller Per Endpoint** structure, meaning for each HTTP Route + Verb, a Controller is created to serve that request.

There are only a couple of non-conforming controllers (See `SetTenantLogoController` for an example), in which `PUT` and `POST` implementations are identical, and so are served within the same Controller.

Controllers are located under the `Octopus.Server.Web.Controllers.` namespace, collected in sub-directories based on the primary domain model they serve interactions for - for example, Tenant related Controllers are collected under `Octopus.Server.Web.Controllers.Tenants`.

We exclusively use [Attribute Routing](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/routing?view=aspnetcore-5.0#attribute-routing-for-rest-apis) to declare routes for our Controllers.

## Mapping and Validation

We rely on [ASP.NET Model Binding](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-5.0) to map incoming request payloads to parameters and models we work with in Controllers. We have created a custom [MultiSource Model Binder](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Infrastructure/Binders/MultiSourceModelBinder.cs) to allow binding a single model from various request sources (Route, query, headers, body).

[ASP.NET Model Validation](https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-5.0) is _disabled_ (applied in [this PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/7627)), as we are relying on existing [mapping](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Mapping/ResourceMapper.cs) and [validation](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Validation/IDocumentValidator.cs) mechanisms within Octopus to validate our resources and models.

_TODO: Complete paragraph on incoming resource validation once an opinion is formed._

Domain model validation was previously performed with a combination of `Rules`, `DocumentValidators`, and code within `Responders`. This validation should now be centralized and exercised within domain models themselves, see [Tenant's Validate](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Core/Model/Tenants/Tenant.cs) for an example.

We anticipate how we shape and validate resources will continue to change, informed by our opinions on [Domain Modelling](https://github.com/OctopusDeploy/Architecture/blob/master/OctopusServer/DomainModel.md) and the progression of our API.

## OpenAPI / Swagger

We are using [Swashbuckle](https://github.com/domaindrivendev/Swashbuckle.AspNetCore) to generate our OpenAPI/Swagger documentation for our API.

> The Swagger document served from `~/api/swagger.json` is produced currently by combining the OpenAPI Document Swashbuckle generates for our ASP.NET Controllers with an OpenAPI Document we create for our existing Nancy endpoints, driven by `MergeNancyOpenAPIPathsFilter` and the `NancyOpenApiPathGenerator`. As we migrate to ASP.NET, the amount of documentation coming from these components will shrink, until we can eventually remove them.

Some of the information presented in the document is derived from attributes we use to mark up our Controller endpoints and parameters. See the [GetAllTenantsController](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Controllers/Tenants/GetAllTenantsController.cs) for an example of the attributes that are expected to adorn all of our Controller endpoints.

These attributes are **mandatory**, as they help our API be a first-class product. They can be considered as essential as labels and information on any page of the Octopus Web UI.

## Guiderails

To help developers remember the parts they need to put in place for each Controller, we have established a suite of [Controller Conventions](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests/Server/Web/Controllers/ControllerConventionsFixture.cs) that inspect all Controllers created and will not pass unless the required structures and attributes are in place.

# Developing

## Creating New ASP.NET Controllers

Creating a new ASP.NET Controller should be straight-forward.

Add a new Controller class under the appropriate namespace. Most endpoints will need to be space-scoped, so derive from `SpaceScopedApiController` to ensure your request is established in an appropriate Space Partition. Ensure you add the required ASP.NET and Swashbuckle attributes to the Controller endpoint and any incoming parameters. Running the [Controller Conventions](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Tests/Server/Web/Controllers/ControllerConventionsFixture.cs) will help you discover anything you may have missed.

When in doubt, the [Tenant Controllers](https://github.com/OctopusDeploy/OctopusDeploy/tree/master/source/Octopus.Server/Web/Controllers/Tenants) were migrated to act as _Lighthouse implementations_, and should be used as references when creating new Controllers if required.

## Migrating Existing Nancy Responders

### Safety Nets

During the migration of the Tenants Nancy Module, performance, wire-compatibility, and swagger-compatibility safety nets were used to ensure our new ASP.NET infrastructure performed as well or better than our existing Nancy infrastructure. We refurbished part of the [Performance Test Suite](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.E2ETests.Performance/Readme.md) to accomplish this testing.

We found:

- Performance of new ASP.NET Controllers was on average ~25% better (note that this is baseline performance on stable hardware, not under load / variable conditions)
- With correct attribution (enforced by convention tests), Swagger output was improved
- Wire compatibility of responses was as-expected, with some improvement on some existing Nancy-isms (particularly around the handling of chunked encoding)

For future migrations, implementors may want to consider the risk / complexity involved in their particular migration, and consider:

- Existing E2E test coverage that exercises the endpoint(s) in question
- Whether additional coverage utilizing [AssentTestScript] within the [Performance Test Suite](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.E2ETests.Performance/Readme.md) would provide additional confidence

We don't think additional performance testing for regression is required given our observed results during the Tenants migration, unless there is a particular reason to suspect performance might change as a result of the migration.

### Legacy Responders (Legacy Nancy Request Processing)

Will use one of the Responder registration methods declared on `OctopusNancyModule` (like `Load`, `Create`, or `Modify`), with responders implementing [Responder](https://github.com/OctopusDeploy/OctopusDeploy/blob/master/source/Octopus.Server/Web/Infrastructure/Api/Responder.cs), and may use `PersistenceRule`s to enforce validation or create other side-effects.

To migrate legacy responders, all `PersistenceRule` code should be re-implemented within the Controller or domain model, and the existing rules deleted ([Example PR](https://github.com/OctopusDeploy/OctopusDeploy/pull/7668)). It is important to consider where in the Responder lifecycle the rules were applied, so that the behaviour can be accurately replicated within the Controller.

### Custom Responders (Newer Nancy Request Processing)

Will use the `CustomAction` registration method on `OctopusNancyModule` and implement `Custom{Action/Create/Modify}Responder`. Rules utilized will implement `IResponderStep`, with a simple 'veto' pattern used for rule enforcement directly from the responder.

To migrate Custom Responders, the default recommendation is taking the `IResponderStep` rules as dependencies to the Controller, and exercising them within the new Controller created.

## Data Sources

Source listed below might provide additional context but keep in mind that they can disappear at any time.

[A Less Confusing API Framework Pitch](https://docs.google.com/document/d/1kZ5pOKBHvkJ2ahva2L_ReYS_TklqAO_679zJSlYYrno/edit)

[Domain Modelling](https://github.com/OctopusDeploy/Architecture/blob/master/OctopusServer/DomainModel.md)

[Domain Modelling Spike](https://github.com/OctopusDeploy/OctopusDeploy/pull/6834)
