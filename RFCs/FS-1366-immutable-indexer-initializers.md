# F# RFC FS-1366 - Indexer initializers for immutable collections

The design suggestion [Indexer initializers for immutable collections](https://github.com/fsharp/fslang-suggestions/issues/FILL-ME-IN) is not approved yet.
This RFC extends [FS-1364](https://github.com/fsharp/fslang-design/pull/869) to types that have no indexer setter.

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/FILL-ME-IN)
- [ ] Approved in principle
- [ ] Implementation (not started)
- [ ] Discussion

# Summary

FS-1364 calls the indexer setter on the result of a call. An immutable collection has no indexer setter.
This RFC gives the same syntax a meaning for a type that has a collection builder for `KeyValuePair` elements. The construction becomes one call of the builder:

```fsharp
let a = ImmutableDictionary<string, int>(StringComparer.Ordinal, ["a"] <- 1, ["b"] <- 2)
// ImmutableDictionary.CreateRangeWithOverwrite(StringComparer.Ordinal, span of the two pairs)
```

# Motivation

Today F# code must name the builder of an immutable collection, or build a list of tuples first:

```fsharp
let b = ImmutableDictionary.CreateBuilder<string, int>(StringComparer.Ordinal)
b["a"] <- 1
b["b"] <- 2
let a = b.ToImmutable()

let m = Map [ "a", 1; "b", 2 ]
```

With FS-1364 alone, the code still fills a mutable collection and converts it:
`ImmutableDictionary.CreateBuilder<string, int>(["a"] <- 1).ToImmutable()` or `Dictionary<string, int>(["a"] <- 1).ToFrozenDictionary()`.

C# creates these collections with collection expressions through `CollectionBuilderAttribute`. Library authors add this attribute for C#.
This RFC lets F# use the same attribute with the FS-1364 syntax.

# Detailed design

FS-1364 applies first. This RFC applies only to a call that has an indexer initializer and a result type without an accessible indexer setter.
It also requires all of these conditions:

1. The call constructs a named type `R`: `R(args)` or `new R(args)`.
2. `R` has a valid `CollectionBuilderAttribute` by the C# 12 rules, with the builder type `B` and the method name `N`.
3. The iteration type of `R` is `KeyValuePair<'K, 'V>`.

Then `R(a1, ..., an, [k1] <- v1, ..., [km] <- vm)` means one call `B.N(a1, ..., an, span)`.
`span` is a `ReadOnlySpan<KeyValuePair<'K, 'V>>` with the elements `KeyValuePair(k1, v1), ..., KeyValuePair(km, vm)` in source order.

1. Overload resolution selects `B.N` with the arguments `a1, ..., an` before the span, as for the C# 15 [collection expression arguments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-15.0/collection-expression-arguments). The constructors of `R` do not take part.
2. The compiler evaluates `a1, ..., an`, then `k1, v1, ..., km, vm`, and then calls `B.N` once.
3. Each initializer MUST have exactly one index. Named property setters are errors.
4. The builder method sets the result for a repeated key. `ImmutableDictionary.CreateRangeWithOverwrite`, `FrozenDictionary.Create` and the `Map` builder of FS-1365 keep the last value, as repeated `[k] <- v` does.
5. If the span parameter of `B.N` is `scoped`, the compiler MAY put the elements on the stack.

If a condition is false, the FS-1364 error stays.

```fsharp
FrozenDictionary<string, int>(["a"] <- 1)   // FrozenDictionary.Create(span)
Map(["a"] <- 1, ["b"] <- 2)                  // with FS-1365: one builder call, type Map<string, int>
```

## Interactions

- Quotations: this form is not valid in a quotation, because a quotation cannot contain a span.
- SRTP: there is no new constraint. The type `R` must be known at the call.
- Type providers: provided types work as other types do.
- [FS-1342](https://github.com/fsharp/fslang-design/pull/838), if accepted, uses the same collection builders for target-typed literals. For the same elements, both features give the same result.

# Changes to the F# spec

[§14.4 Method Application Resolution](https://github.com/fsharp/fslang-spec/blob/main/spec/inference-procedures.md#method-application-resolution), as FS-1364 changes it:

- Step 1: if this RFC applies, the candidate method group is the methods `B.N`, not the constructors of `R`.
- Step 2: if this RFC applies, the indexer initializers give one more unnamed actual argument after the other unnamed actual arguments. This argument is the span.

# Drawbacks

- The syntax has two meanings: setter calls or one builder call. Only tools such as Quick Info show which one applies.
- The constructors of `R` do not take part. For example, `Map(pairs, ["a"] <- 1)` is an error, because the builder has no such overload.
- A type without the attribute gets no support. For example, `ImmutableSortedDictionary` does not have it.

# Alternatives

- `ToBuilder()` and `ToImmutable()`. The result type of a call has `ToBuilder()`, and the builder has an indexer setter and `ToImmutable()`. Then the initializers go to the builder:

  ```fsharp
  let s = ImmutableSortedDictionary.Create<string, int>(["b"] <- 2, ["a"] <- 1)
  // let b = ImmutableSortedDictionary.Create<string, int>().ToBuilder()
  // b["b"] <- 2; b["a"] <- 1; b.ToImmutable()
  ```

  This covers `ImmutableSortedDictionary` and the results of methods, and it is valid in quotations.
  It does not cover `FrozenDictionary` or `Map`, because they have no `ToBuilder()`.
  It gives a language meaning to the method names of one library, and C# does not do that.
  It also adds no new behavior: with FS-1364 alone, `ImmutableSortedDictionary.CreateBuilder<string, int>(["a"] <- 1).ToImmutable()` does the same work.
- Both mechanisms. Then one syntax has three meanings and needs a priority rule.
- `SetItem` or `Add` for each item. Each item allocates a new collection, and `<-` no longer sets a value on one object.
- A literal with a target type (FS-1342) and a key-value syntax. It needs a known target type. This RFC takes the type from the call.
- No change. FS-1364 covers these types with one more call, as the motivation shows.

# Prior art

C# 12 collection expressions call the method of `CollectionBuilderAttribute`. C# 15 passes the `with(...)` arguments before the span, as this RFC does with positional arguments.

# Compatibility

- This is not a breaking change. These forms are errors today and under FS-1364 alone.
- The feature needs FS-1364 and a new language version.
- There is no new metadata.

# Interop

The feature uses the metadata of C# collection expressions. Other languages see an ordinary call of the builder method.

# Pragmatics

## Diagnostics

1. No indexer setter and no collection builder: the FS-1364 error. It SHOULD say that the type has no collection builder for `KeyValuePair` elements.
2. A collection builder with more than one index, or with a named property setter: a new error.
3. This form in a quotation: a new error.

## Tooling

Quick Info on the brackets shows the builder method. Go To Definition goes to that method.

## Performance

The compiled code calls the builder once with m elements and makes no intermediate collection. Code that compiles today does not change.

## Scaling

As in FS-1364. For a large m, the compiler SHOULD use heap storage for the span.

## Culture-aware formatting/parsing

Not applicable.

# Unresolved questions

1. Is the `ToBuilder()` and `ToImmutable()` alternative a better choice than the collection builder?
