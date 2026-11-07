# F# RFC FS-1366 - Indexer initializers for immutable collections

The design suggestion [Indexer initializers for immutable collections](https://github.com/fsharp/fslang-suggestions/issues/FILL-ME-IN) is not approved yet.
This RFC extends [FS-1364](https://github.com/fsharp/fslang-design/pull/869) to types that have no indexer setter.

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/FILL-ME-IN)
- [ ] Approved in principle
- [ ] Implementation (not started)
- [ ] Discussion

# Summary

FS-1364 calls the indexer setter on the result of a call. An immutable collection has no indexer setter. This RFC gives the same syntax two more meanings for such types:

```fsharp
// Path A: one call of the collection builder
let a = ImmutableDictionary<string, int>(StringComparer.Ordinal, ["a"] <- 1, ["b"] <- 2)
// Path B: ToBuilder, the setters, then ToImmutable
let s = ImmutableSortedDictionary.Create<string, int>(StringComparer.Ordinal, ["b"] <- 2, ["a"] <- 1)
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

With FS-1364 alone, the builder stays in the code: `ImmutableDictionary.CreateBuilder<string, int>(["a"] <- 1).ToImmutable()`.
C# creates the same collections with collection expressions. The libraries already have the metadata that this RFC uses.

# Detailed design

FS-1364 applies first. This RFC applies only when the call has an indexer initializer and the result type has no accessible indexer setter.
Path A has priority over path B. If no path applies, the FS-1364 error stays.

## Path A: collection builder

Path A applies when:

1. The call constructs a named type `R`: `R(args)` or `new R(args)`.
2. `R` has a valid `CollectionBuilderAttribute` by the C# 12 rules, with the builder type `B` and the method name `M`.
3. The iteration type of `R` is `KeyValuePair<'K, 'V>`.

Then `R(a1, ..., an, [k1] <- v1, ..., [km] <- vm)` means one call `B.M(a1, ..., an, span)`.
`span` is a `ReadOnlySpan<KeyValuePair<'K, 'V>>` with the elements `KeyValuePair(k1, v1), ..., KeyValuePair(km, vm)` in source order.

1. Overload resolution selects `M` with the arguments `a1, ..., an` before the span, as for the C# 15 [collection expression arguments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-15.0/collection-expression-arguments). The constructors of `R` do not take part.
2. The compiler evaluates `a1, ..., an`, then `k1, v1, ..., km, vm`, and then calls `M` once.
3. Each initializer MUST have exactly one index. Named property setters are errors.
4. The builder method sets the result for a repeated key. `ImmutableDictionary.CreateRangeWithOverwrite`, `FrozenDictionary.Create` and the `Map` builder of FS-1365 keep the last value, as repeated `[k] <- v` does.
5. If the span parameter of `M` is `scoped`, the compiler MAY put the elements on the stack.

```fsharp
FrozenDictionary<string, int>(["a"] <- 1)   // FrozenDictionary.Create(span)
Map(["a"] <- 1, ["b"] <- 2)                  // with FS-1365: one builder call, type Map<string, int>
```

## Path B: builder round trip

Path B applies when path A does not apply, and the result type `R` of the call has:

1. an accessible instance method `ToBuilder()` without parameters, with the result type `Bld`;
2. an accessible indexer setter on `Bld` for each initializer;
3. an accessible instance method `Bld.ToImmutable()` without parameters, with the result type `R`.

Then `call(args, [k1] <- v1, ..., [km] <- vm)` means:

```fsharp
let b = call(args).ToBuilder() in b[k1] <- v1; ...; b[km] <- vm; b.ToImmutable()
```

Each initializer follows FS-1364 on `b`. Named property setters are errors, because the result `R` is not the object that they change.

## Interactions

- Quotations: path A is not valid in a quotation, because a quotation cannot contain a span. Path B uses ordinary calls, so a quotation contains them.
- SRTP: there is no new constraint. Both paths need a known result type.
- Type providers: provided types work as other types do.
- [FS-1342](https://github.com/fsharp/fslang-design/pull/838), if accepted, uses the same collection builders for target-typed literals. For the same elements, path A gives the same result.

# Changes to the F# spec

[§14.4 Method Application Resolution](https://github.com/fsharp/fslang-spec/blob/main/spec/inference-procedures.md#method-application-resolution), as FS-1364 changes it:

- Step 4: for a construction with an indexer initializer, if path A applies, the candidates are the methods `B.M`. The span is the last unnamed argument.
- Step 10: if path B applies, check and elaborate the indexer initializers on `v.ToBuilder()`. The result is the value of `ToImmutable()` on that builder.

# Drawbacks

- One syntax has three meanings: setter calls, one builder call, or a builder round trip. Tools must show the selected meaning.
- Path B depends on two method names.
- In path A, the constructors of `R` do not take part. Thus `Map(pairs, ["a"] <- 1)` is an error, because the builder has no such overload.

# Alternatives

- Only path A. It does not cover `ImmutableSortedDictionary` or the results of methods.
- Only path B. It does not cover `FrozenDictionary` or `Map`, which have no `ToBuilder()`.
- `SetItem` or `Add` for each item. Each item allocates a new collection, and `<-` no longer sets a value on one object.
- A literal with a target type (FS-1342) and a key-value syntax. It needs a known target type. Path A takes the type from the call.

# Prior art

- C# 12 collection expressions call the method of `CollectionBuilderAttribute`. C# 15 passes the `with(...)` arguments before the span, as path A does with positional arguments.
- `System.Collections.Immutable` uses `ToBuilder()` and `ToImmutable()` as a convention. C# has no syntax for it.

# Compatibility

- This is not a breaking change. Under FS-1364 alone, these forms are errors.
- The feature needs FS-1364 and a new language version.
- There is no new metadata.

# Interop

Both paths use existing metadata and conventions of .NET libraries. Other languages see ordinary calls.

# Pragmatics

## Diagnostics

1. No indexer setter and no path: the FS-1364 error. It SHOULD say that the type has no collection builder and no `ToBuilder()`.
2. Path A with more than one index, or with a named property setter: a new error.
3. Path A in a quotation: a new error.

## Tooling

Quick Info on the brackets shows the builder method for path A and the indexer of the builder for path B. Go To Definition goes to the same member.

## Performance

Path A calls the builder once, with m elements. Path B makes one builder and one immutable result. Existing code does not change.

## Scaling

As in FS-1364. For a large m, the compiler SHOULD use heap storage for the span of path A.

## Culture-aware formatting/parsing

Not applicable.

# Unresolved questions

1. Keep both paths, or only one of them?
