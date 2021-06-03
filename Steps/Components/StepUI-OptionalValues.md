# Step UI - Optional Values

When defining our inputs, we need a way of expressing whether an input is optional or required.

- [Background](#Background)
- [Scenarios](#Scenarios)
  - [Required values](#required-values)
  - [Optional values](#optional-values)
- [Solutions](#Solutions)
  - [Preferred Solution](#preferred-solution)
  - [Rejected Solutions](#rejected-solutions)

# Background

## Schema Validation

Our validation framework should be helpful enough to validate that the types supplied for each input matches the expected types in the schema.

```ts
// If this is our inputs type...
type MyInputs = {
    foo: string;
}

// ...and this value is passed to the API...
{
    foo: 42
}

// ...then our validation framework should return an error like
// Error: A number (42) was supplied for the "foo" property, which only accepts string values
```

## Bounded contexts

There are a few useful bounded contexts to consider for this problem

- **UI Inputs**: The inputs used by the Step UI
- **API Inputs**: The inputs used by the Octopus Server backend and passed to the API layer
- **Execution Inputs**: The inputs used by the step executor

# Scenarios

## Required values

For these inputs
```ts
type MyInputs = {
    foo: number;
}
```

the `foo` value should be required for all bounded contexts, since it should be invalid to pass an empty value like `undefined` at any of these points in time. The Schema Validation should assert that any values provided through the API also conform to this requirement.

## Optional values

There are only two scenarios where optional values make sense
- A value can be optional within all bounded contexts, including at execution time
- A value can be optional in the UI inputs, but not in the API inputs, nor in the execution inputs.

This section provides more details about these scenarios, and also talks about a third scenario that is not required.

In these examples, consider a set of inputs like the following, where we want the `foo` property to be optional in some bounded contexts.

```ts
type MyInputs = {
    foo?: number;
}
```

### Optional at execution time

If we want a type to be optional at execution time, this also implies it should be optional at UI input and API input time.

|UI Input              | Api Input             | Execution Input       |
|----------------------|-----------------------|-----------------------|
|`number \| undefined` | `number \| undefined` | `number \| undefined` |

In this case, the `foo` property can be omitted in the UI or by someone crafting a raw API request, and this would be valid. At execution time, the value may not be on the inputs object, and the step executor is forced to handle the case where the `foo` value has not been provided.

### Optional at API configuration time

If we want a type to be required at execution time, but optional when you configure the props through the API or the UI, this implies the following shape:

|UI Input              | Api Input             | Execution Input       |
|----------------------|-----------------------|-----------------------|
|`number \| undefined` | `number \| undefined` | `number`              |

If you want this type to be required at execution time, you might implement a validation rule to validate that the type is not `undefined`. At this point, the validation rule would also execute when you try to configure this value through the API, which violates the design of our validators that assumes the same rules apply for the API Inputs as for the Execution Inputs.

**There are also *not* any useful cases where this applies, so we _don't_ need to support this case.**

### Optional at UI configuration time

If we want a type to be required at execution time and in the API inputs, but optional when you are configuring it in the UI, this implies the following shape:

|UI Input              | Api Input             | Execution Input       |
|----------------------|-----------------------|-----------------------|
|`number \| undefined` | `number`              | `number`              |

A validation rule can be implemented to ensure that this value is required and not `undefined`, and this validation rule can run both at execution time and at configuration time as part of the API.

However, when configuring the UI, the value can still be undefined. To see why this is a valid scenario, consider the following use cases:

1. A step has a set of radio buttons, but none of these make sense to be selected by default. If the user tries to save the step, they will receive a validation error because they have not yet picked a value for this radio button group. The initial value of the underlying input field should reflect that "nothing is selected".
2. A deployment target has a reference to an account. There is no value that makes sense to be selected by default for this account, so the Select control is initially empty. If the user tries to save the deployment target without selecting an account, they would receive a validation error. The initial value of the underlying input field should reflect that "nothing is selected".

# Solutions

## Preferred solution

For cases where the input value is required, or an input value is optional in all bounded contexts (including execution time), we will rely on typescript to represents this as follows

```ts
// Required in all bounded contexts
type MyInputs = {
    foo: number;
}
// Optional in all bounded contexts
type MyInputs = {
    foo?: number;
}
```

For cases where the input value can only be optional at UI configuration time, we lean on the fact that the only time this occurs is when the value is initially empty because there is no value we could reasonably default it to.

**To handle this, we introduce a special type (and corresponding special value) called `EmptyInitialValue` (naming TBC).**

This special type can be included as part of a union type, in the following way:

```ts
type MyInputs = {
    foo: number | EmptyInitialValue;
}
```

For some more concrete examples, consider the case where there is a set of radio buttons with no default value selected:

```ts
type MyInputs = {
    radioButtonValue: "first option" | "second option" | "third option" | EmptyInitialValue;
}
```

Also consider the case where an account needs to be selected from a dropdown, and there is no default value selected:

```ts
type MyInputs = {
    account: Account<AzureServicePrincipal | EmptyInitialValue>;
}
```

When constructing the initial inputs for this step, the empty initial value will be provided so that the step author can initialise any values to this empty value:

```ts
createInitialInputs: ({ emptyInitialValue }) => ({
    account: emptyInitialValue;
})
```

The mapped types for ExecutionInputs (and for validation too) will strip out the `EmptyInitialValue` type such that it is not a valid value that can be accepted in the step executor function or the validation functions.

That is

```ts
type MyInputs = {
    foo: number | EmptyInitialValue;
}

type MyExecutionInputs = ExecutionInputs<MyInputs>;
// MyExecutionInputs is equivalent to:
type MyExecutionInputs = {
    foo: number;
}
```

The `EmptyInitialValue` will also be treated differently in the input schema, such that it is ignored during Schema validation.

### Variant: `UIOnly`

If we discover that there are any special values for other cases other than the "empty initial value", we might generalise this pattern further. 

We could instead have a mapped type like `UIOnlyValue` that can be used to wrap any types, allowing them to be used in the UI, but stripped out in other bounded contexts and their mapped types.

```ts
type MyInputs = {
    foo: number | UIOnly<"empty initial value">
}

type MyExecutionInputs = ExecutionInputs<MyInputs>;
// MyExecutionInputs is equivalent to:
type MyExecutionInputs = {
    foo: number;
}
```

This might make setting up initial values feel more natural:

```ts
createInitialInputs: () => ({
    foo: "empty initial value";
})
```

## Rejected solutions

### Mapped type: `Required`

We could wrap types in a `Required` mapped type

```ts
type MyInputs = {
    foo: Required<number>;
}
```

Pros
- It makes it even more explicit that a property is required.

Cons
- This is highly ambiguous. If you omit this `Required` mapped type and just left it as `number`, what would this mean? I would guess this means it is optional because it is not required. How does this differ from `number | undefined` or `foo?: number`?
- It doesn't really solve the problem of distinguishing between the two flavours of optional types (optional at execution time, or optional at UI configuration time).

### Mapped type: `Optional`

We could wrap types in an `Optional` mapped type

```ts
type MyInputs = {
    foo: Optional<number>;
}
```

Pros
- Compared to `Required` this makes the distinction between required and optional types clearer: If it is lacking the `Optional` attribute, then it is a required type

Cons
- How do you distinguish between the two flavours of optional types (optional at executoin time, or optional at UI configuration time)? One idea might be using two different mapped types, like `OptionalDuringExecution<T>` and `OptionalDuringConfiguration<T>`.
- There still appears to be multiple ways of doing things. e.g. what is the difference between `Optional<number>` and `number | undefined`? Perhaps `number | undefined` should represent "optional during execution" while `Optional<number>` should only represent "optional during UI configuration". This still feels a bit confusing.
- What does it mean to do things like this: `Optional<number | undefined>`? Is the `| undefined` part redundant, or does it mean something?

### Separate execution and configuration inputs

Instead of having a single source of truth for our inputs that we map to our different bounded contexts, we could require step authors to provide different represenations of their inputs for different bounded contexts. For example

```ts
type UIInputs = {
    foo?: number;
}

type ExecutionInputs = {
    foo: number;
}
```

This feels like a non-starter because of the additional friction involved in authoring steps (which violates our fundamental goal of making step authoring simple), and other infrastructural pieces that might be needed (eg asserting at build time that the two types are compatible with each other).

### Parsing instead of validating

Instead of having a set of validators, we could have a set of parsers, following the [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) pattern.


For this set of inputs:

```ts
type UIInputs = {
    foo?: number;
}
```
This might look like

```ts
const parsers = (parse) => ({
    foo: parse((foo) => {
        if (foo === undefined) throw new Error("foo must be defined");
        return foo; // the type here has been narrowed to `number`
    })
})
```

We could then somehow infer (using mapped types and [ReturnType](https://www.typescriptlang.org/docs/handbook/utility-types.html#returntypetype)) the type of the execution and API configuration inputs based on the parsers.

We think this option is also a non-starter because of the massive increase in complexity (particularly when it comes to dealing with union types), the extra difficulty with implementing parsers, and the lack of explicitness about what the execution inputs will look like.