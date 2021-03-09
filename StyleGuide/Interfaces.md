# Interfaces

Interfaces are a level of indirection between the consumer and implementation. This indirection is often useful, but may sometimes be unnecessary. 

## Dependency Injection

Concrete implementations can be dependency injected. Therefore it is not necessary to create an interface for a type for the sole purpose of injection.

## Location

The default position is that an interface should live in the same file as its implementation. It should be moved out if
- the interface is implemented by multiple types (outside of tests)
- the interface should live in a different project
- it makes things clearer for it to live separately (e.g. size)

## Naming

Interfaces should begin with `I`. If there is a single implementation, the rest of the name should match. An exception is if the consumer may implement the interface (e.g. the single implementation is the default).
Clever names (e.g. `ICanModifyVariables`) are encouraged as long as they are still clear.
