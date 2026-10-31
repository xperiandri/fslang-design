# F# RFC FS-1359 - Interop with C# 14 extension members

The design suggestion [Interop with C# 14 extension members in both directions](https://github.com/fsharp/fslang-suggestions/issues/NNNN) has not been marked "approved in principle" yet.
This RFC covers the detailed proposal for this suggestion.

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/NNNN)
- [ ] Approved in principle
- [ ] Implementation (not started)
- [ ] Discussion

Related: [FS-1043 extension members in SRTP](FS-1043-extension-members-for-operators-and-srtp-constraints.md), [dotnet/fsharp#20035](https://github.com/dotnet/fsharp/issues/20035), [dotnet/roslyn#84629](https://github.com/dotnet/roslyn/issues/84629).

# Summary

C# 14 compiles an `extension` block to a documented metadata shape. This RFC makes F# read that shape and write it.

- F# resolves every member of a C# 14 extension block: instance and static methods, instance and static properties, and operators. Today F# resolves only the instance methods.
- A type extension with `[<Extension>]` compiles to that shape. C# then sees F# extension properties, static extension members and extension operators.

```fsharp
[<Extension>]
type Thing with
    member x.Tripled = x.Value * 3                                     // C#: thing.Tripled
    static member Create v = Thing v                                   // C#: Thing.Create(5)
    static member (+) (a: Thing, b: Thing) = Thing(a.Value + b.Value)  // C#: a + b
```

The feature is `CSharpExtensionMembers` (preview). Consumption is not gated.

# Motivation

- .NET 11 `System.Private.CoreLib` declares the static properties `E`, `Pi`, `Tau`, `Epsilon`, `NaN`, `NegativeOne`, … of `Vector64/128/256/512<T>` in extension blocks. `Vector128<float32>.Pi` is FS0039 in F#. The only call today is `Vector128.get_Pi<float32>()`.
- ASP.NET Core 10 `Microsoft.AspNetCore.Identity` ships one block with 42 members. More libraries will follow.
- The unspeakable nested types of a block resolve in F#. A call to their members throws `NotSupportedException` at run time.
- F# compiles `member x.Tripled` of a type extension to a static method named `Thing.get_Tripled`. C# cannot name it ([dotnet/fsharp#20035](https://github.com/dotnet/fsharp/issues/20035), [dotnet/fsharp#16362](https://github.com/dotnet/fsharp/issues/16362)). Users write C# shims.
- The C# team will not add an attribute for other languages ([dotnet/roslyn#84629](https://github.com/dotnet/roslyn/issues/84629)). The metadata shape is the contract.

# Detailed design

## The metadata shape

Terms are from the [C# specification](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-14.0/extensions.md#metadata-for-declarations). Roslyn checks this structure only. It does not check the type names.

- **Container**: a top-level, non-generic, abstract sealed class with `[Extension]`.
- **Grouping type**: nested in the container. Public, sealed, not abstract, `specialname`, `[Extension]`, base `Object`, no interfaces. One for each receiver type and set of CLR constraints. Its type parameters are the type parameters of the block.
- **Marker type**: nested in the grouping type. Public, abstract sealed, `specialname`, base `Object`, no interfaces. It redeclares the type parameters of the grouping type with the same constraints. It holds exactly one method `static void '<Extension>$'(receiver)`: `specialname`, not generic, one parameter. The parameter gives the receiver type, its ref kind and its name.
- **Skeleton members**: methods, properties and their accessors in the grouping type. Each carries `[ExtensionMarker("<marker type name>")]`. Instance members have `this`. Their bodies throw. Operators are `public specialname` with `op_` names.
- **Implementation methods**: static methods in the container with the name and accessibility of the skeleton. Type parameters: those of the block, then those of the member. Parameters: the receiver first for an instance member, then the parameters of the member. Return type and constraints match the skeleton. An instance method also carries `[Extension]`.
- `System.Runtime.CompilerServices.ExtensionMarkerAttribute(string)` is in `System.Runtime` since .NET 10. Roslyn matches it by name and embeds a copy when the target lacks it.

## Consumption

Name resolution treats the container as a classic extension class, with these additions:

- For each grouping type it reads each marker type. The marker method gives the receiver type `R` over the type parameters of the marker type.
- For each skeleton member whose attribute names the marker type, it finds the implementation method with the rules above. A skeleton without an implementation is not visible.
- An instance member becomes an extension member of `R`. A static member becomes a static extension member of `R`, as an F# static extension member. A property becomes an extension property. Its accessors are the implementation methods `get_P` and `set_P`.
- Member lookup, overload resolution, accessibility, nullness and attributes use the skeleton signature. SRTP accepts an extension operator from a block under the rules of FS-1043.
- Code generation calls the implementation method. The receiver is the first argument.
- Completion and `open type` hide grouping and marker types. Name resolution still accepts their names, as C# does.
- Priority: an intrinsic member wins over an extension member. F# and C# extension members in scope follow the existing `open` order.

## Emission

`[<Extension>]` is permitted on a type extension in a module. Today this is FS3246. For such an extension the compiler emits the shape:

- The enclosing module is the container. It gets `[Extension]` if it does not have it.
- Members group by receiver type and constraints into grouping types. The grouping and marker type names are content-based hashes of the signature. They are stable between builds and unspeakable.
- Each member gets a skeleton and an implementation method with a speakable name: `get_Tripled(Thing)`, `set_Tripled(Thing, int)`, `Create(int)`, `op_Addition(Thing, Thing)`. Today the compiled names are `Thing.get_Tripled` and `Thing.Create.Static`.
- An instance method also carries `[Extension]`, so F# and C# 13 see a classic extension method.
- The compiler embeds `ExtensionMarkerAttribute` when the target framework lacks it, as for `IsReadOnlyAttribute`.
- F# callers use the F# signature data as today. The compiled names change only for members under `[<Extension>]`.

Members with no C# counterpart compile as today and get a warning: curried members, conversion operators and events.

A `[<Extension>]` type extension in a nested module gets a warning. A nested module compiles to a nested class, and C# reads extension members only from a top-level container.

# Changes to the F# spec

- [§8.13 Type Extensions](https://fsharp.github.io/fslang-spec/type-definitions/#813-type-extensions): permit `[<Extension>]` on an optional type extension; add the compiled form of [Emission](#emission).
- [§14.1 Name Resolution](https://fsharp.github.io/fslang-spec/inference-procedures/#141-name-resolution): add the members of [Consumption](#consumption) to the extension members of a type, with the receiver from the marker method.

# Drawbacks

- An opted-in extension emits two members for each source member. Assemblies grow.
- F# assemblies get unspeakable nested types.
- F# depends on the structural checks of Roslyn. A change there needs a change in F#.

# Alternatives

- **`[<Extension>]` on the module**: opts in every type extension in the module. Less precise; it can be added later.
- **A new attribute** such as `[<CSharpExtension>]`: a second attribute for one concept.
- **Emit the shape for every type extension**: changes compiled names and breaks binary compatibility.
- **Consume only**: F# reads the BCL members; F# libraries stay invisible to C#.
- **An attribute contract from C#**: rejected in [dotnet/roslyn#84629](https://github.com/dotnet/roslyn/issues/84629).

# Prior art

- C# 14 extension members: [proposal](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-14.0/extensions.md). Roslyn recognises the shape in `PENamedTypeSymbol` and accepts hand-written IL with plain type names in its tests.
- Classic extension methods: the `[Extension]` contract since .NET 3.5. F# consumes and emits it.
- VB.NET consumes the instance methods as classic extension methods only.

# Compatibility

- Not a breaking change. `[<Extension>]` on a type extension is FS3246 today. Extensions without it compile as today.
- Older F# compilers reading source with `[<Extension>]` on a type extension: FS3246.
- Older F# compilers reading new binaries: they use the F# signature data and call the implementation methods by compiled name. They see grouping types as plain nested types, as the current compiler sees C# blocks.
- FSharp.Core does not change.

# Interop

- C# 14 sees the members as extension members. C# 13 and VB.NET see the instance methods as classic extension methods and the rest as static methods of the module.
- F# sees C# 14 extension members with the same syntax as F# extension members.

# Pragmatics

## Diagnostics

| Number | Condition | Severity |
|---|---|---|
| new | `[<Extension>]` type extension in a nested module | warning |
| new | a member of a `[<Extension>]` type extension has no C# counterpart: curried, conversion operator, event | warning |
| FS3246 | an attribute other than `[<Extension>]` on a type extension | error, unchanged |
| FS3350 | `[<Extension>]` on a type extension and the feature is off | error |

## Tooling

Completion, tooltips and Go to Definition show C# 14 extension members as extension members of the receiver type. Completion hides grouping and marker types. FCS reports them through `IsExtensionMember` with the receiver as the apparent enclosing entity. Debugging and colorization do not change.

## Performance

The compiler reads a container once per assembly and caches the result, as for classic extension classes. Emission adds one pass over opted-in extensions.

## Scaling

Not applicable.

## Culture-aware formatting/parsing

Not applicable.

# Unresolved questions

- The hash scheme for the generated names. Roslyn uses XxHash128 of a raw signature string. The same scheme gives the same XML doc IDs from C# and F#.
- Indexers: extension indexers are preview in C#.
- Receivers with `byref`/`inref`, which C# encodes as `ref`/`ref readonly` on the marker parameter.
- The feature name `CSharpExtensionMembers` is a placeholder.
