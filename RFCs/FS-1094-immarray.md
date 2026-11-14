# F# RFC FS-1094 - `immarray`, an immutable array with value semantics

The design suggestion [A normative immutable array type](https://github.com/fsharp/fslang-suggestions/issues/619) has been marked "approved in principle". This RFC covers the detailed proposal for this suggestion.

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/619)
- [x] Approved in principle
- [ ] [Implementation](https://github.com/fsprojects/FSharp.Collections.Immutable/pull/FILL-ME-IN): `EquatableArray<'T>`, a library prototype of this design under a library name, not the dotnet/fsharp change
- [x] [Discussion](https://github.com/fsharp/fslang-design/discussions/528)

# Summary

FSharp.Core gets `'T immarray`, an immutable array that is a value as `'T list` is: equality, hashing and comparison are element-wise, and the default value is the empty array. It is a struct over one `System.Collections.Immutable.ImmutableArray<'T>` field, so conversion to and from the .NET type is O(1). The module `ImmArray` has the functions of `Array`. No syntax is added.

This changes the central decision of the earlier text, where `immarray` abbreviated `ImmutableArray<'T>`; [Motivation](#motivation) gives the reasons.

# Motivation

The suggestion asks for a default immutable array for F# models and APIs. The earlier text states that with `ImmutableArray<'T>` F# equality, hashing and comparison "are respected all the way down". They are not (F# 10, .NET 10):

```fsharp
let a, b = ImmutableArray.Create (1, 2), ImmutableArray.Create (1, 2)
a.Equals b                // false: Equals, EqualityComparer.Default and Dictionary keys compare references
compare a (a.Add 3)       // exception: lengths differ. Also <, max, List.sort, Set.ofList
let f = ImmutableArray.Create<int -> int> id
f = f                     // compiles: 'equality' holds for every element type
let d = Unchecked.defaultof<ImmutableArray<int>>
d = ImmutableArray.Empty  // false
d.Length                  // NullReferenceException
a[0..1]                   // error FS0039: no GetSlice
sprintf "%A" a            // "seq [1; 2]"
```

`a = b` is `true`, but `=` boxes each element, and `hash` reads only the last 8 elements. FSharp.Core cannot repair a type that it does not own.

# Detailed design

## The type

```fsharp
namespace FSharp.Collections

[<Struct; CustomEquality; CustomComparison; CompiledName "FSharpImmArray`1">]
type ImmArray<[<EqualityConditionalOn; ComparisonConditionalOn>] 'T> =
    interface IEquatable<ImmArray<'T>>
    interface IStructuralEquatable
    interface IComparable
    interface IComparable<ImmArray<'T>>
    interface IStructuralComparable
    interface IReadOnlyList<'T>
    member Length: int
    member IsEmpty: bool
    member Item: index: int -> 'T with get
    member GetSlice: startIndex: int option * endIndex: int option -> 'T immarray
    member GetEnumerator: unit -> ImmutableArray<'T>.Enumerator   // a struct
    member AsSpan: unit -> ReadOnlySpan<'T>
    static member op_Implicit: 'T immarray -> ImmutableArray<'T>
    static member op_Implicit: ImmutableArray<'T> -> 'T immarray

and 'T immarray = ImmArray<'T>
```

1. The type MUST have exactly one instance field, of type `ImmutableArray<'T>`, so conversion in both directions is O(1) without allocation.
2. Each member MUST treat a field in the `IsDefault` state as `ImmutableArray<'T>.Empty`. So `Unchecked.defaultof<'T immarray>` is the empty array, and `toImmutableArray` never returns a default instance.
3. `GetSlice` has the bounds rules of array slices.
4. FSharp.Core references System.Collections.Immutable. On `netstandard2.0` its dependency System.Memory supplies `ReadOnlySpan<'T>`, so all target frameworks have the same API.

## Equality, hashing and comparison

5. Two values are equal when they have the same length and equal elements at each index. The hash MUST combine the hashes of all elements.
6. Elements are compared and hashed as the elements of a `'T list` are: `=` uses PER equality, `Equals` uses ER equality, and the `IStructuralEquatable` and `IStructuralComparable` members use the comparer that they receive.
7. Comparison is by length first, then by the first index with different elements: the rule of `'T[]`, not the lexicographic rule of `'T list`.
8. `'T immarray` satisfies `equality` only when `'T` does, and `comparison` only when `'T` does.

```fsharp
let imm = ImmArray.ofList
let a, b = imm [1; 2], imm [1; 2]
a = b && a.Equals b && hash a = hash b  // true, so b finds a in a Dictionary
imm [nan] = imm [nan]                   // false, and Equals gives true, as for [nan]
imm [[|1|]] = imm [[|1|]]               // true: elements are compared structurally
compare (imm [9]) a                     // -1: the shorter array is less
imm [sin] = imm [sin]                   // error FS0001: no 'equality' for a function type
```

## The `ImmArray` module

`ImmArray` (compiled name `ImmArrayModule`, `RequireQualifiedAccess`) MUST have each function of the `Array` module, with the same name, parameter order and behavior, and with `'T immarray` in place of each `'T[]`, also in results. The exceptions:

- Not included: `blit`, `fill`, `set`, `zeroCreate`, `sortInPlace*` and `randomShuffleInPlace*`, which mutate their argument or return uninitialized storage; `copy`; `Array.Parallel`.
- Added: `ofArray` and `toArray`, which copy; `ofImmutableArray` and `toImmutableArray`, which are O(1).

## Construction, conversion and patterns

The `[: ... :]` form of the earlier text is withdrawn in favour of target-typed `[ ... ]` ([FS-1342](https://github.com/fsharp/fslang-design/pull/838)), which applies to types with `CollectionBuilderAttribute`. `ImmArray<'T>` has it, as in [FS-1145](../FSharp.Core-9.0/FS-1145-csharp-collection-expression-support-for-lists-and-sets.md): a non-generic static type `ImmArray` (compiled name `FSharpImmArray`, hidden from F# code) has `Create: ReadOnlySpan<'T> -> 'T immarray`.

The `op_Implicit` members follow [FS-1093](../FSharp-6.0/FS-1093-additional-conversions.md):

```fsharp
let xs = ImmArray.ofList [1; 2; 3]
let ys: int immarray = [1; 2; 3]      // with FS-1342: no intermediate list

type Api = static member Take (items: ImmutableArray<int>) = items.Length
Api.Take xs                           // at a method argument: no warning
let zs: ImmutableArray<int> = xs      // elsewhere: warning FS3391
```

This RFC adds no pattern syntax. Patterns are the subject of RFC FS-1357, collection patterns ([suggestion #1478](https://github.com/fsharp/fslang-suggestions/issues/1478)): `Length`, `Item` and `AsSpan` make `[| a; b |]` and `[| a; ...rest |]` apply to `'T immarray`, with a `ReadOnlySpan<'T>` rest.

## Other language features

- Quotations, type providers and SRTP see an ordinary struct with ordinary members. No quotation node, provider API or built-in constraint solution is added.
- Inference: no overload is added to an existing type or module.

# Changes to the F# spec

[Immutable Collection Types (Map, Set)](https://github.com/fsharp/fslang-spec/blob/main/spec/the-f-library-fsharpcoredll.md#immutable-collection-types-map-set): add `immarray`. The language chapters do not change, because the type is an ordinary struct.

# Drawbacks

- F# gets a second immutable array type. .NET APIs use `ImmutableArray<'T>`, so a conversion appears at each such boundary.
- FSharp.Core gets its first package dependency. On `netstandard2.0` and `netstandard2.1`, System.Collections.Immutable brings System.Memory and System.Runtime.CompilerServices.Unsafe; .NET Framework applications can need binding redirects.

# Alternatives

- **The abbreviation** of the earlier text, which the suggestion thread assumed (2020-05-22: "a type semantically equivalent to" `ImmutableArray`). It needs no conversion and keeps every defect in Motivation.
- **A wrapper over `'T[]`**: the same layout, no dependency. FSharp.Core then cannot convert to `ImmutableArray<'T>` in O(1), because `ImmutableCollectionsMarshal` is in System.Collections.Immutable.
- **`EqualityComparer<'T>.Default` for elements**, in place of rule 6. It does not box struct records, which rule 6 boxes as `'T list` does. But array elements then compare by reference and `=` is `true` for NaN elements, unlike every F# collection. The prototype does this: a library reaches rule 6 only through `IStructuralEquatable` with boxed elements.
- **A separate `EquatableArray<'T>` in FSharp.Core**, next to an `immarray` that stays the abbreviation: a separate RFC is drafted for it. It is the prototype unchanged: no compiler support and no boxing of F# struct elements, but the element rules of the previous alternative. FSharp.Core then has two immutable arrays, and `immarray` keeps the defects in Motivation.
- **A library only**: no normative type, and a library that exposes the type puts a dependency on its users.
- **The `[: ... :]` syntax**: a third bracket form, for one type. Discussion #528 (2022-10-27, 2023-04-12) and [dotnet/fsharp#12859](https://github.com/dotnet/fsharp/pull/12859) (2023-12-22) prefer type-directed construction.

# Prior art

- [dotnet/runtime#118189](https://github.com/dotnet/runtime/issues/118189) asks .NET for this type as `EquatableArray<T>`, the prototype's name and the name of the copies that Roslyn incremental generators keep, such as [ComputeSharp's](https://github.com/Sergio0694/ComputeSharp). The answers (2026-09-20): "It's trivial to create a no-cost wrapper around ImmutableArray that has sequence-equality semantics", and the .NET libraries are "not going to be adding dozens of extra collection types". So a wrapper is the accepted shape, and .NET leaves it to libraries.
- The OCaml 5.4 standard library has `'a iarray` and the module [`Iarray`](https://ocaml.org/manual/5.4/api/Iarray.html); its `compare` also orders by length first.

# Compatibility

- Not a breaking change: existing definitions named `ImmArray` or `immarray` shadow the new ones.
- No syntax or F# metadata is added. An older compiler uses the type from a newer FSharp.Core as any struct.

# Interop

- C# sees the struct `FSharpImmArray<T>` with an indexer, `Length`, `foreach` without allocation, collection expressions (`FSharpImmArray<int> xs = [1, 2, 3];`), implicit conversions to and from `ImmutableArray<T>`, and `default` as the empty array. C# records and tuples that hold it get value equality through `IEquatable<T>`. As for other F# types, there is no `==` operator.
- System.Text.Json (.NET 10) writes the type as a JSON array but cannot read it (`NotSupportedException`), even with `CollectionBuilderAttribute`. It knows F# lists, sets and maps by type name; this type needs the same, or a converter.

# Pragmatics

## Diagnostics

None are added.

## Tooling

- FSharp.Compiler.Service: no syntax or typed tree node is added; the type and its abbreviation are ordinary symbols.
- `%A` and F# Interactive MUST print `immarray [1; 2; 3]`, by the rule that prints `set [1; 2; 3]`.
- The debugger shows the elements through a `DebuggerTypeProxy`, as for `'T list`.

## Performance

- One `=` on bare `ImmutableArray<int>` values of 32 elements allocates 1584 B; on `int list` values, nothing (.NET SDK 10.0.401).
- Equality, hashing and comparison SHOULD NOT allocate more than the same operation on a `'T list` with the same elements. FSharp.Core can use the typed comparers of the generated equality of records and unions; a library cannot (warning FS1204).
- Use as `seq<'T>` boxes the value, unlike a list or an array.

## Scaling

Elements in one value: up to the CLR array limit. Equality, hashing and comparison are O(n). No compiler input grows.

## Culture-aware formatting/parsing

None. String elements compare ordinally, as in F# generic comparison.

# Unresolved questions

1. Element equality: rule 6, or `EqualityComparer<'T>.Default`.
2. Comparison: length first as for arrays (rule 7), or lexicographic as for lists.
3. `op_Implicit` in both directions, or only `ofImmutableArray` and `toImmutableArray`.
4. Patterns: dsyme (2022-06-13) left them to active patterns unless list patterns are extended to indexable types. RFC FS-1357 proposes that extension; until it is approved, users write active patterns.
5. A function `immarray` (as `set`) or a builder `immarray { ... }`: the suggestion sketches both (2022-06-13).
6. Fable, named by the suggestion, does not support `ImmutableArray<'T>` and needs its own implementation.
