
## Out
Avoid using `out` parameters except when there is a well established pattern in the .NET ecosystem, e.g. `Dictionary.TryGet`, `TryParse`. Even then the alternatives might be better.

## Ref
Avoid. At the time of writing, only 6 methods took `ref` parameters, so chances are your code doesn't need it. The .NET Framework only uses them in specific cases (e.g. `Interlocked`).

## Alternatives

The `Maybe<>` and `Result<>` types exist to indicate whether a result is returned and in the case of `Result<>` an error message if it failed.

Another alternative is to use a `Tuple`:

```
var (success, result) = DoSomething();
```

If the method needs to return more than two results, or a complex type a `class` may be more appropriate.