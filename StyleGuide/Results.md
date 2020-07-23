# Results
There are a number of occasions where the caller of a method would benefit from knowing whether the method succeeded, and if not then some details as to why not. In these cases we use a result object pattern.

The Result object base implementation comes from the `Octopus.Data` package. A typical method signature for a method that results a result would be something like

``` csharp
public IResult<SomeObject> TryGetSomeObject()
{
    ...
}
```

and a call to it might look like

```csharp
var result = foo.TryGetSomeObject();
```

An `IResult` is, by design, not entirely useful in itself. To work out whether the method succeeded or failed you have to check what type of result you actually got back.

## Failures

To check if the method failed, you need to know if the result is a `IFailureResult`. So you might do something like

```csharp
var result = foo.TryGetSomeObject();
if (result is IFailureResult failure)
    log.Error($"Something went wrong! Error: {failure.ErrorString}")
```

An `IFailureResult` actually has 2 key properties. `Errors`, which is an array of error details. `ErrorString`, which is the `Errors` concatenated together, with newlines separating each.

## Successes

To check if the method succeeded, you need to know if the result is an `ISuccessResult<T>`. So in our example case you might have something like

```csharp
var result = foo.TryGetSomeObject();
if (result is ISuccessResult<SomeObject> success)
    DoSomethingWithSomeObject(success.Value)
```

The key property on an `ISuccessResult<T>` is the `Value` property, which will be of type T.

In the cases where only success or failure is of concern, and there is no object type to return, a success result is signified by an `ISuccessResult` being returned.

## Extensions

Extensions have some specialised Result objects, defined in the `Octopus.ServerExtensibility` package. They will return an `IResultFromExtension`/`IResultFromExtension<T>`, which inherit the `IResult` interfaces.

The main reason for this is for the failure conditions that can be specific to extensions, namely a failure to indicate that the extension is actually currently disabled. In these cases the methods will return an `IFailureResultFromDisabledExtension`/`IFailureResultFromDisabledExtension<T>`. These are `IFailureResults`, so will appear as failures, but these can be specifically targeted in cases where special treatment is required.

## Nullable types

There are some cases where a success might still yield a null result. In these cases use a nullable reference type in the generics setup for the method. For example,

```csharp
public IResult<SomeObject?> TryGetSomeObject()
{
    ...
}
```

and calls to it could look something like the following

```
var result = foo.TryGetSomeObject();
if (result is ISuccessResult<SomeObject?> success)
    if (success.Value != null)
        DoSomethingWithSomeObject(success.Value)    
```

## Try

The method in the examples above is an illustration of our preferred way of implementing "try" style methods. E.g. ones where you may or may not get an answer. Traditional these were implemented as Try methods that returned a bool along with some `out` parameters.

These new call semantics make the implementations clearer. As an example it wasn't always clear in some methods whether/which out parameters would be set if the method returned false.

## Implementing a method that returns a Result

So far we've looked at the caller, let's have a look at what the implementation for the `TryGetSomeObject` might look like.

```csharp
public IResult<SomeObject> TryGetSomeObject()
{
    if (SomeConditionIsMet)
        return Result<SomeObject>.Success(new SomeObject());
    return Result<SomeObject>.Failed("The condition wasn't met :(");
}
```

