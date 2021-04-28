
# Introduction
This document provides guidelines on how to approach logging when creating steps in Octopus.

# Guidelines

1. Users are the primary consumers of log information at the info and error levels.
2. Info level messages provide a story of what just happened. Each message should include enough information to make sense on its own.
3. Support teams and those debugging failed deployments are the primary consumers of log information at the verbose and error levels.
4. Verbose log messages do not have to be as refined, but should still have context without requiring a deep understanding of the source code.
5. External tool logs should be captured at the verbose level. Also watch for tools printing debug messages to the error stream - despite the name, these aren’t always errors!
6. Stack traces should only appear in logs for unforeseen errors. They should always be verbose.
7. Predictable and known errors should be listed with a unique code, e.g. `LambdaDeployment-Permissions-MissingDeployRole`, along with a concise error message. These codes are easy to Google and can be documented outside of the product to potentially allow users to resolve without a support ticket. The error code should be a link taking users to the documentation (via an octopurl).

# Error code format

Error codes have the format `StepId-Area-Identifier`.

## StepId

A unique id for the step. Should be unique enough that it won’t generate many false positives in a text search. 

So a step ID of `Lambda` is not suitable, as this will return unrelated tickets about Lambdas in general.

A step ID of `LambdaDeployment` is a better choice.

## Area

A list of standard error types. Examples might include:

* Permissions
* Configuration
* Networking
* Memory
* General (a catch all unpredictable error)

This list is not exhaustive. Try to use existing areas if possible, but use a new area if no suitable examples already exist.

## Identifier

A unique identifier like `MissingDeployRole`. If the underlying platform provides a convenient error code ([AWS provides some for example](https://docs.aws.amazon.com/lambda/latest/dg/troubleshooting-deployment.html)), that can be used here.

## Code Explanation

This code format allows support forums (internal and external like stack overflow) and emails to be searched for text like `LambdaDeployment`, `LambdaDeployment-Permissions`, or `LambdaDeployment-Permissions-MissingDeployRole` to quickly gauge the kinds of issues customers are encountering.

Because the codes are unique, they also allow customers the opportunity to search sites like stack overflow for resolutions to their issue.

The full error message starts with the error code and presents a concise error message, such as:

```
Lambda-Permissions-MissingDeployRole: The supplied account did not have permission IAM/LambdaDeployRole to deploy the application.
```

The error code will be presented as a link i.e. printed to the logs like:

```
[Lambda-Permissions-MissingDeployRole](https://g.octopushq.com/Lambda-Permissions-MissingDeployRole]): The supplied account did not have permission IAM/LambdaDeployRole to deploy the application.
```

The documentation can contain more details, like screenshots and external links. This external documentation can also be updated and maintained by teams like support and customer success to improve the context around error messages without having to update Octopus itself.
