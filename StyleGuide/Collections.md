# Collections

See also  [performance of collections](https://www.c-sharpcorner.com/UploadFile/0f68f2/comparative-analysis-of-list-hashset-and-sortedset/)

In general:
- Default to using the collection interfaces, they support a wider variety of concrete collection types. Exception `ReferenceCollection`.
- When handling a set of IDs, use `ReferenceCollection` and `ReferenceCollection<>`
- When accepting a collection, use the most generic collection interface that still captures the intent
- When returning a collection, use the most specific collection interface that matches the contract you wish to provide
- Default to the `ReadOnly` collection types. They indicate that the method will not modify the collection during it's execution or after it returns.

## Commonly Used

| Collection | Description |
| - | - |
| I[ReadOnly]Collection<> | Unordered collection |
| I[ReadOnly]List<> | Ordered collection that can be indexed |
| ReferenceCollection | Unordered collection of unique strings |
| ReferenceCollection<> | Unordered collection of unique tiny types |
| IHashSet<> | Unordered collection of unique items where the consumer is likely to look up many items |

## ReferenceCollection

`ReferenceCollection` and `ReferenceCollection<>` are `HashSet<>` with case-insensitive comparison under the covers. They ensure the items are unique and you do not need to worry about case-sensitivity when doing lookups.

Lookups in `HashSets<>` are typically `O(1)`, so using `ReferenceCollection` will change comparison of two collections from `O(n^2)` to `O(n)`.

## ToArray/ToList

For the most part collection sizes in octopus are in the order of `100,000` or less. The performance impact of creating collections of that size is negligible compared to data access. We prefer the known behaviour that comes from a copied collection over the risk that a collection might be modified elsewhere.

Internally `ToArray` and `ToList` use `ICollection.Count` to optimise the copy.

The performance of `ToArray` and `ToList` are similar at our scale. `ToList` does not have to do a double copy of `ICollection.Count` is not available, however `Array`s have faster indexing times.

## Any and None

Internally `Any()` (and by extensions `None`) have `O(1)` performance if the underlying collection is `ICollection` even if the variable type is `IEnumerable`

## IEnumerable

Avoid accepting and returning `IEnumerable` unless the method is part of a filter chain. In particular if the method needs to iterate over the enumerable multiple times, or the method converts it to a collection, let the caller do the work of converting to a `ICollection` derived type.

Avoid lazy collections (i.e `IEnumerable`) as they are hard to debug when they 💥 due a filter that is several steps removed from the place they error. Again we prefer developer comprehension over small performance/memory gains.
