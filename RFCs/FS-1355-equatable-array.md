# F# RFC FS-1355 - `EquatableArray`, an immutable array with value equality

No design suggestion exists yet for an immutable array with value equality as a type of its own. [#619](https://github.com/fsharp/fslang-suggestions/issues/619), the suggestion of [FS-1094](FS-1094-immarray.md), is related. This RFC is a design for discussion.

- [ ] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/FILL-ME-IN): not filed
- [ ] Approved in principle
- [ ] [Implementation](https://github.com/fsprojects/FSharp.Collections.Immutable/pull/FILL-ME-IN): `EquatableArray<'T>`, a library prototype with the same name and these members, not the dotnet/fsharp change
- [ ] [Discussion](https://github.com/fsharp/fslang-design/discussions/FILL-ME-IN)

# Summary

FSharp.Core gets `EquatableArray<'T>` and the module `EquatableArray`: an immutable array that every comparer, F# or .NET, compares, hashes and orders by its elements, and whose default value is the empty array. It is a struct over one `ImmutableArray<'T>` field, so conversion to and from the .NET type is O(1). There is no syntax and no compiler change. Patterns are the subject of RFC FS-1357, collection patterns ([suggestion #1478](https://github.com/fsharp/fslang-suggestions/issues/1478)): `Length`, `Item` and `AsSpan` make `[| a; ...rest |]` apply, with a `ReadOnlySpan<'T>` rest.

# Motivation

Records and unions that hold arrays (syntax trees, models, cache keys) need those arrays to be values. `ImmutableArray<'T>` is not one (F# 10, .NET 10):

```fsharp
let a, b = ImmutableArray.Create (1, 2), ImmutableArray.Create (1, 2)
a = b                     // true, but boxes each element; hash reads the last 8 elements
a.Equals b                // false, as EqualityComparer.Default: Dictionary keys, C# records
compare a (a.Add 3)       // ArgumentException: lengths differ. Also <, List.sort, Set.ofList
let d = Unchecked.defaultof<ImmutableArray<int>>
d = ImmutableArray.Empty  // false; d.Length throws NullReferenceException
```

Its `equality` constraint holds for every element type, functions included. So programs wrap it by hand. [dotnet/runtime#118189](https://github.com/dotnet/runtime/issues/118189) asks .NET for that wrapper as `EquatableArray<T>`; it is open with milestone Future, and the answer of 2026-09-20 was that the .NET libraries are "not going to be adding dozens of extra collection types".

# Detailed design

## The type

```fsharp
namespace FSharp.Collections

[<Struct; CustomEquality; CustomComparison; CompiledName "FSharpEquatableArray`1">]
type EquatableArray<[<EqualityConditionalOn; ComparisonConditionalOn>] 'T> =
    new: ImmutableArray<'T> -> EquatableArray<'T>
    member Length: int
    member IsEmpty: bool
    member Item: int -> 'T with get
    member GetSlice: int option * int option -> EquatableArray<'T>  // bounds as for array slices
    member GetEnumerator: unit -> ImmutableArray<'T>.Enumerator     // a struct
    member AsSpan: unit -> ReadOnlySpan<'T>
    member AsMemory: unit -> ReadOnlyMemory<'T>
    member Equals: EquatableArray<'T> -> bool
    member CompareTo: EquatableArray<'T> -> int
    static member op_Implicit: EquatableArray<'T> -> ImmutableArray<'T>
    static member op_Implicit: ImmutableArray<'T> -> EquatableArray<'T>
    interface IEquatable<EquatableArray<'T>>
    interface IComparable<EquatableArray<'T>>
    interface IComparable
    interface IReadOnlyList<'T>
```

1. The type MUST have exactly one instance field, of type `ImmutableArray<'T>`. So it has the size of a reference, and conversion in both directions is O(1) without a copy.
2. Each member MUST treat a field in the `IsDefault` state as the empty array. So the default value is the empty array, and `toImmutableArray` never returns a default instance.
3. The names are those of the prototype and of the .NET proposal, with no lowercase abbreviation. Compiled type names take the `FSharp` prefix, as `FSharpList` does.

## Equality, hashing and comparison

4. Two values are equal when they have the same length and, at each index, elements that are equal by `EqualityComparer<'T>.Default`. The hash MUST combine the hashes of all elements.
5. The type MUST NOT implement `IStructuralEquatable` or `IStructuralComparable`. F# `=` and `hash` then give the result of rule 4, as each .NET comparer does.
6. Comparison is by length first, as for `'T[]`, then by the first index with different elements, in the order of F# generic comparison.
7. `EquatableArray<'T>` satisfies `equality` only when `'T` does, and `comparison` only when `'T` does.

```fsharp
let ea = EquatableArray.ofList
ea [nan] = ea [nan]           // true; [nan] = [nan] is false
ea [[|1|]] = ea [[|1|]]       // false, though compare gives 0: array elements are equal by reference; nest EquatableArray
compare (ea [9]) (ea [1; 2])  // -1: the shorter array is less
ea [nan] < ea [1.0]           // true: operators follow CompareTo; false for [|nan|] < [|1.0|]
ea [sin] = ea [sin]           // error FS0001: no 'equality' for a function type
```

## The `EquatableArray` module

`EquatableArray` (compiled name `EquatableArrayModule`, `RequireQualifiedAccess`) MUST have each function of the `Array` module, with the same name, parameter order and behavior, and with `EquatableArray<'T>` in place of each `'T[]`, also in results. The exceptions:

- Not included: `blit`, `fill`, `set`, `zeroCreate`, `sortInPlace*` and `randomShuffleInPlace*`, which mutate their argument or return uninitialized storage; `copy`; `Array.Parallel`.
- Added: `ofArray` and `toArray`, which copy; `ofImmutableArray` and `toImmutableArray` (rule 1).

## Construction, conversion and other features

The type has `CollectionBuilderAttribute`, as `'T list` has (FS-1145): a hidden non-generic static type `EquatableArray` (compiled name `FSharpEquatableArray`) with the one method `Create: ReadOnlySpan<'T> -> EquatableArray<'T>`. So target-typed `[ ... ]` ([fslang-design#838](https://github.com/fsharp/fslang-design/pull/838)) will apply: `let xs: EquatableArray<int> = [1; 2; 3]`.

`op_Implicit` follows FS-1093: no warning at a method argument, warning FS3391 elsewhere. At a method argument it also selects an overload that takes `ImmutableArray<'T>` over one that takes `seq<'T>`; an argument that is itself an overloaded call does not convert (error FS0193).

Quotations, type providers and SRTP see an ordinary struct; an implicit conversion is quoted as a `Call` of `op_Implicit`. No overload is added to an existing type or module.

# Changes to the F# spec

The F# Library FSharp.Core.dll, section "Immutable Collection Types (Map, Set)": add `EquatableArray`. The language chapters do not change.

# Drawbacks

- One more array-like type; with FS-1094, two immutable arrays. .NET APIs use `ImmutableArray<'T>`, so a conversion appears at each such boundary.
- FSharp.Core gets its first package dependency, which FS-1094 accepted: on `netstandard2.0` and `netstandard2.1`, System.Collections.Immutable with System.Memory and System.Runtime.CompilerServices.Unsafe. System.Memory supplies `ReadOnlySpan<'T>` on `netstandard2.0`, so all target frameworks have the same API.

# Alternatives

- **`immarray` itself as this wrapper** (a revision of FS-1094 under review): one type, with the element rules of `'T list` through `IStructuralEquatable`. It avoids boxing each element only with support inside FSharp.Core, which a library cannot have (warning FS1204): without it, `=` allocates as on the bare `ImmutableArray` (measured: 1584 B for 32 `int` elements). F# struct elements stay boxed as in `'T list`. This RFC needs no such support; its prototype is the type itself.
- **`Comparer<'T>.Default` in rule 6**: it orders strings by the current culture and throws for array elements.
- **A wrapper over `'T[]`**: no dependency, but no O(1) conversion to `ImmutableArray<'T>` in FSharp.Core: `ImmutableCollectionsMarshal` is in System.Collections.Immutable.
- **A library only** (the prototype): no normative type, and a library that exposes the type puts a dependency on its users.
- **Status quo**: each project keeps its own wrapper.

# Prior art

The Roslyn [incremental generators cookbook](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.cookbook.md) tells authors to "use a wrapper type around arrays to augment them with value-based equality"; dotnet/runtime, dotnet/maui and microsoft/testfx each keep a `struct EquatableArray<T>`.

# Compatibility

- Not a breaking change: an existing definition named `EquatableArray` shadows the new one.
- No F# metadata is added. An older compiler uses the type from a newer FSharp.Core as any struct.

# Interop

- C# sees the struct `FSharpEquatableArray<T>`: `foreach` over the struct enumerator, collection expressions (`FSharpEquatableArray<int> xs = [1, 2, 3];`) and implicit conversions to and from `ImmutableArray<T>`. As for other F# types, there is no `==`.
- System.Text.Json (.NET 10) writes the type as a JSON array but cannot read it (`NotSupportedException`). It knows F# lists, sets and maps by type name; this type needs the same, or a converter.

# Pragmatics

## Diagnostics

None are added.

## Tooling

FSharp.Compiler.Service sees an ordinary symbol. `%A` and F# Interactive MUST print the value as an array, `[|1; 2; 3|]`, not as the default `seq [0; 1; 2; 3; ...]`. The debugger shows the elements, as for `'T list`.

## Performance

Bytes allocated by one `a = b` on 32 elements (.NET SDK 10.0.401, optimized):

| Elements | `ImmutableArray` | `'T list` | `EquatableArray` |
|---|---:|---:|---:|
| `int` | 1584 | 0 | 0 |
| `string`, F# union | 48 | 0 | 0 |
| F# struct record | 2096 | 2048 | 0 |

- `hash` of an `EquatableArray` allocates nothing in these rows. `'T list` boxes struct records because they implement `IStructuralEquatable`.
- Rule 4 boxes a struct element without `IEquatable<'T>`; rule 6 boxes struct records, as `compare` on a list does.
- Use as `seq<'T>` boxes the value.

## Scaling

Equality, hashing and comparison are O(n) in the elements. No compiler input grows.

## Culture-aware formatting/parsing

None. `%A` is culture-invariant, and string elements compare ordinally (rule 6).

# Unresolved questions

1. Element equality: `EqualityComparer<'T>.Default` (rule 4), or the rules of `'T list` as in the FS-1094 revision.
2. Comparison: rule 6, or no comparison, which keeps the type out of `Set` and `Map` keys.
3. `op_Implicit` in both directions, or only `ofImmutableArray` and `toImmutableArray`.
4. C# `==` and `!=`, and a read-only `IList<'T>` for LINQ: the prototype and `ImmutableArray<'T>` have them, `'T list` does not.
5. Fable has no support for `ImmutableArray<'T>` and needs its own implementation of the type.
