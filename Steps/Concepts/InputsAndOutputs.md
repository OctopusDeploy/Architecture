# Index

- [Inputs](#inputs)
  - [Problem](#problem)
  - [Solution: Input and Output Schemas](#solution-structured-inputs-and-input-schemas)
- [Bounded Contexts](#bounded-contexts)
  - [Problem](#problem-1)
  - [Solution: Mapped Types](#solution-type-and-data-mapping)
- [Bound Variables](#bound-variables)
  - [Problem](#problem-2)
  - [Constraints](#constraints)
  - [Design](#design)
- [Validation](#validation)
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

TBA

# Outputs

TBA
