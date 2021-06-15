# Index

- [Index](#index)
- [Inputs](#inputs)
  - [Problem](#problem)
  - [Solution: Structured Inputs and Input Schemas](#solution-structured-inputs-and-input-schemas)
- [Bounded Contexts](#bounded-contexts)
  - [Problem](#problem-1)
  - [Solution: Type and Data Mapping](#solution-type-and-data-mapping)
- [Bound Variables](#bound-variables)
  - [Problem](#problem-2)
  - [Constraints](#constraints)
  - [Design](#design)
    - [Guiding principle](#guiding-principle)
      - [Why?](#why)
    - [Our Step API should be unaware of bound variables when](#our-step-api-should-be-unaware-of-bound-variables-when)
    - [Our step API will need to be aware of bound variables when](#our-step-api-will-need-to-be-aware-of-bound-variables-when)
  - [Resources](#resources)
- [Validation](#validation)
  - [Problem](#problem-3)
  - [Solution](#solution)
    - [Configuration time validation](#configuration-time-validation)
      - [Schema-based validation](#schema-based-validation)
      - [Execution environment](#execution-environment)
    - [Execution time validation](#execution-time-validation)
    - [Validation Rules](#validation-rules)
- [Outputs](#outputs)

# Inputs

To do their job, steps need to be given inputs that define all of their relevant contextual information - cloud service details, package details, Octopus environmental details, and more. These inputs are usually supplied by a user via the step's UI.

For further detail on how the below decisions have been implemented, see the [Step Package Documentation]() hosted on the `step-api` repository.

## Problem

Prior to Step Packages, Octopus provided no explicit schema definition for the inputs a given step needed defined to do its job - it used a Namespace-keyed state bag approach to supplying steps with their inputs.

The problems with this approach are:

- It is difficult to find what inputs a step expects
- It is difficult to tell what type of information each input should capture (i.e. number, string, complex type) - at the moment the only place we enforce the type of info is within validators
- Complex types can only be expressed as flat sets of keys
- Input keys are redefined in up to three places for certain steps
- It is difficult to determine what keys within the state bag a given step might care about at runtime

## Solution: Structured Inputs and Input Schemas

Steps will define structured types for their inputs. These types may reference types defined by the step itself, and types defined by Octopus Server, such as Package References and Sensitive Values. So that Octopus Server can interpret and consume these types, a language-agnostic representation of the inputs will be made avilable, using [JSON Schema](https://json-schema.org/).

By using structured types, inputs are defined in a single place, and it is simple to understand what inputs a Step expects.

[Inputs Map](https://whimsical.com/steps-inputs-map-QyP5kQgsTtXSSStdDTAGVZ)

[Further Discussion](https://docs.google.com/document/d/19qz4U33sK_xwGJATBxJ52CdYNQM-hzSbULX2H8mxbBA)

# Bounded Contexts

## Problem

Different parts of the Step Package ecosystem may have different conceptual views of the same logical step input model.

Let's use **Package Reference** as an example.

The `Step UI API` enables Step Package authors to be able to specify an input property as a `PackageReference`, but not actually give them access to any of the internal properties of this type - the internal representation is owned by Octopus Server.

To the Step UI, the `PackageReference` type looks like so:

```ts
export type PackageReference = {
  readonly __packageReferenceBrand: unique symbol;
};
```

Octopus Server is responsible for managing package references - it is both responsible for providing the UI framework components that capture package reference details, and is responsible for coordinating package-related features, as described in the [Execution](https://github.com/OctopusDeploy/Architecture/blob/master/Steps/Concepts/Execution.md) documentation.

To Octopus Server, the `PackageReference` type looks like so:

```c#
public class PackageReference
{
  ...
  public string Id { get; }

  public string Name { get; set; } = string.Empty;

  public string PackageId { get; set; } = string.Empty;

  public FeedIdOrName FeedIdOrName { get; set; }

  public string AcquisitionLocation { get; set; } = string.Empty;

  ...
}
```

Finally, at execution time, our Step Executor is interested in package information, but in a different shape. It is less concerned about the feed Id of a package, or its name, and is more interested in details like _where was it extracted?_

To the Step Executor, the `PackageReference` type looks like so:

```ts
export type PackageReference = {
  extractedToPath: string;
};
```

Each of these views is a _Bounded Context_ - it is an interpretation of a given logical model (Package References) that is relevant to a specific context - UI, Server, and Executor.

## Solution: Type and Data Mapping

The Step UI will always own the canonical model for a step's inputs, and sometimes it will compose into that model other types whose canonical model is owned by Server (PackageReference in our example). When a canonical model needs to be used in another bounded context, we will employ a mixture of Type and Data Mapping to transform canonical types and their data into bounded-context specific projections, rather than attempting to extend the original types to cater for other contexts.

# Bound Variables

## Problem

The Step Inputs collection contains various properties that can be configured. These properties are often primitive values like strings or booleans.

Users can bind any of these values to variables, such that the value of these properties can either by their underlying primitive value type (string, boolean, etc), or a bound value.

## Constraints

The following constraints will apply to steps

- Variable binding should be disabled for properties that branches the form or control flow of the step.
  - If these properties could be bound, it would be difficult to
    - reason about what the step UI should look like when it was bound
    - use Union types to represent valid states of properties
  - Customers that want to vary this property can duplicate the step with each configuration they want.
- Every field in the UI has a single explicit property in the inputs object to which it corresponds.
  - There are no cases where multiple UI controls map to a single input property
  - There are no cases where multiple input properties map to a single UI control
- Both our step UI and our step functions should have input properties that are structured similarly
  - The exact types of some of the primitives may vary, but structurally they should be identical (e.g. the same nested objects, arrays, union types)
- Both our Step UI and our step functions should be able to use union types
  - This allows us to build an intuitive, nested, and idiomatic input object structure
- Discriminators should always be explicit properties in our inputs object, rather than implied by the presence or absence of other properties
  - This makes the structure of the input objects more consistent between steps
  - This constraint can drastically simplify the way we develop our Step UIs
- UI components that show or hide based on discriminators should always be shown immediately next to the discriminator selection
  - This could be a section immediately afterwards, or the controls could be inline with the discriminator UI control
- Only supported primitive values can be bound. Objects and lists cannot be bound.
  - Predictable type narrowing and the use of union types is much more difficult without this constraint.

## Design

### Guiding principle

**Remove knowledge of bound variables from our Step API where it would otherwise cause excessive boilerplate, repetition or be a source of inconsistencies.**

#### Why?

It would be nice if our Steps could be completely unaware of bound variables. This would allow Octopus Server to completely own and control this concept, and keep our steps as lean as possible. However, this is not feasible because there are some step-specific functions that need to take bound variables into account.

The next best thing we can do is remove knowledge of bound variables from the places where we would otherwise find ourselves repeating the same patterns and boilerplate in all of our steps. If all of our steps had to implement these same patterns, they would not be as simple as they could be, and there is a greater opportunity for mistakes if we don't follow these patterns consistently.

### Our Step API should be unaware of bound variables when

- Showing, hiding or displaying dynamic content in the UI
- Declaring validation rules
- Receiving inputs into a step function
- Binding UI controls to input properties
- Declaring the input schema (i.e. we should not need to declare which properties support variable binding)

### Our step API will need to be aware of bound variables when

- Serializing to and from an exported representation of the step (e.g. YAML for K8s steps)
- Writing expressive human-readable section summaries that may need to take the context of variable binding into account

## Resources

- [Decision document](https://docs.google.com/document/d/17TZBRvoIp9gHPvipdQSWQnSHtNHuFN6CpcOjzDeJRyo/edit?usp=sharing)

# Validation

## Problem

The types available for step authors to use to model their inputs are very broad (e.g. `string`), meanwhile the set of values that are valid for each input property may be more constrained (e.g. an email input might only accept `string`s that are also valid inputs).

We want to
- Give users feedback when they have provided an invalid value for an input property
- Give step authors confidence that by the time a step executor function is invoked, the inputs are valid
- Make it simple and easy for step authors to express validation rules
- Allow step authors to write debuggable and testable code using tools they are familiar with for their validation rules (e.g. not just regular expresions, not a DSL).

## Solution

Step authors will express validation rules using a validation framework.

There are two times at which we will validate inputs
- Configuration time
- Execution time

### Configuration time validation

At configuration time, we want to give users feedback if they have provided invalid values, as this is gives them the nicest experience and shortest feedback loop. 

The unique part about configuration time inputs is tha some of the values may be bound to variable expressions. The final resolved values of these expressions can't be known until execution time.

While we could allow step authors to possibly validate these inputs when they are bound, this would
- Add extra complexity when writing validation rules
- Add extra boilerplate (every validation rule needs to handle the case when it can be bound)
- Introduces sources of inconsistency - as a step author, how do I validate an input when it is bound?

For the reasons specified in the [Bound Variables - Design](#design) section, we instead don't want to expose knowledge of bound values to validation rule authors. Validation rule authors should only need to write validation rules for the case where an input value is _not_ bound.

If we try to validate an input property, and that property is currently bound to a variable expression, then we will skip any custom validation checks and assume the value is valid.

#### Schema-based validation

At configuration time, we will also run schema-based validation to ensure that the values provided match up with the expected types.

For example, if the inputs are

```typescript
type MyInputs = { someProperty: string; }
```

And as part of an API request, the following inputs were provided:

```json
{
  someProperty: undefined
}
```

Then because `undefined` is not assingable to the `string` type, we should receive a validation error.

However, if the type of `someProperty` was `string | undefined`, then this input would have been valid.

This type information is captured in the input schema, so this kind of validation will be performed automatically by the validation framework, based on the input schema.

#### Execution environment

Configuration time validation runs on Octopus Server, so that any API requests that are received (including those that don't come from our UI) can be validated.

We use [jint](https://github.com/sebastienros/jint) as the execution engine that invokes the javascript validation code included in the step package.

While using an environment like [node.js](https://nodejs.org/en/) would give us a more standard and consistent execution environment (this is the same environment within which our step executors run), there are some concerns with using it:
- It would introduce some additional complexities, like needing to bundle node with Octopus Server, store scripts on disk, and perform IPC
- There are security concerns with allowing step packages to execute javascript code on the machine hosting Octopus Server. This concerns may become more important if we allow third parties to develop step packages.

In contrast, most of the complexity from using jint comes from
- Marshalling data into and out of the jint execution engine
- Ensure we bundle validation code in a way that is compatible with jint

### Execution time validation

At execution time, we will have inputs in which all bound variable expressions have been resolved to values. At this point, we want to run our validation rules against these same inputs, and only if the inputs are valid should we invoke our step executor with these inputs.

This gives a step author a nice assumption: **The inputs will only be passed to the step executor function if they are valid**.

### Validation Rules

When step authors are specifying their validation rules, they need to specify:
- Which input property they want to validate
- A typescript function that accepts non-bound values and returns whether the input value is valid, or an error message if it is invalid

This same set of validation rules drives both the configuration time execution time validation. The main difference is that only some of these rules might be run at configuration time (because some of the inputs might have bound variable expressions), while they should all run at execution time.

# Outputs

TBA
