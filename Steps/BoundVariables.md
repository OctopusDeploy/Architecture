# Bound Variables

## Background

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