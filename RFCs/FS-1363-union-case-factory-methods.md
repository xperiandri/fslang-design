# F# RFC FS-1363 - Union case factory methods

The design suggestion [Allow methods with name New* on DUs as long as they have different parameters](https://github.com/fsharp/fslang-suggestions/issues/532) has been marked "approved in principle". This RFC covers the detailed proposal for this suggestion.

- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/532)
- [x] Approved in principle
- [ ] [Implementation](https://github.com/dotnet/fsharp/pull/FILL-ME-IN)
- [ ] [Discussion](https://github.com/fsharp/fslang-design/discussions/FILL-ME-IN)

# Summary

A union type can declare a static method `New<Case>` with the new attribute `[<UnionCaseFactory>]`. This method is a factory: one more way to construct the case. F# code calls the factory through the case name, `Case(args)`. Other .NET languages see the factory as an overload of the generated method `New<Case>`. The compiler verifies that a factory returns the case that it names. Patterns do not change.

# Motivation

One case can have more than one natural set of inputs. In the suggestion, a duct element has a profile and a length, and a profile has a width and a height:

```fsharp
type Profile = Profile of width: int * height: int

type ElementDimensions =
    | Inherited of length: int
    | CustomDimensions of profile: Profile * length: int
```

Today, a factory from the width, the height and the length needs another name, for example `MakeCustomDimensions`. The name `NewCustomDimensions` gives error FS0023 for all signatures, because the [compiled form](https://fsharp.github.io/fslang-spec/type-definitions/#854-compiled-form-of-union-types-for-use-from-other-cli-languages) already has a method with this name. So each language gets two names for one case: `CustomDimensions` and `MakeCustomDimensions` in F#, `NewCustomDimensions` and `MakeCustomDimensions` in C#.

# Detailed design

```fsharp
type ElementDimensions =
    | Inherited of length: int
    | CustomDimensions of profile: Profile * length: int

    [<UnionCaseFactory>]
    static member NewCustomDimensions(width: int, height: int, length: int) =
        CustomDimensions(Profile(width, height), length)

let a = CustomDimensions(Profile(2, 3), 10)   // the case
let b = CustomDimensions(2, 3, 10)            // the factory

match b with
| CustomDimensions(p, l) -> ()                // the case only
| Inherited _ -> ()
```

```csharp
var a = ElementDimensions.NewCustomDimensions(new Profile(2, 3), 10); // generated
var b = ElementDimensions.NewCustomDimensions(2, 3, 10);              // factory
```

## Method names

In this RFC, `U` is a union type and `C` is a case of `U`. The rules apply to the static methods of `U`, in its definition and in its intrinsic extensions.

| Method name | `[<UnionCaseFactory>]` | Result |
| --- | --- | --- |
| `NewX`, where `X` is not a case of `U` | no | An ordinary method, as today. |
| `NewX`, where `X` is not a case of `U` | yes | Error. |
| `NewC` | no | An ordinary method, and a new warning: add the attribute or use another name. |
| `NewC` | yes | A factory of `C`. |

A method `NewC` MUST NOT have the compiled signature of the generated method `NewC`. The compiled signature is the number of method type parameters and the parameter types after erasure. The return type is not part of it. A case without fields has no generated method `NewC`. A union with `[<DefaultAugmentation(false)>]` has no generated method `NewC` for any case.

Properties and instance members with the name `NewC` keep error FS0023, because a case constructor is a static method. Optional extensions keep their current rules.

## Factories

A factory MUST be a static method of `U` with the name `NewC`. Its parameters MUST be in one tupled group. Its return type MUST be `U`, with any type arguments. The attribute on another member is an error. In all other respects, a factory is an ordinary method.

## Construction

In an application `C(args)` or `U.C(args)`, the candidates are the case and the factories of `C`. The case is a candidate with one parameter for each field, and the field names are the parameter names. [Method application resolution](https://fsharp.github.io/fslang-spec/inference-procedures/#144-method-application-resolution) selects one candidate. A call to a factory is a method call.

The case name without arguments, as in `List.map C xs`, refers to the case only. So a new factory does not change code that uses the case as a first-class function.

F# code calls a factory through the case name only. The name of a factory does not resolve in F#, the same as the generated method today.

## Patterns

Patterns do not change. The pattern `C pat` matches the case only, and a factory is never a pattern. Because of the body check, a value from `C(args)` always matches `C _`.

## Body check

The compiler checks the body of each factory after type inference. The body is in a result position. These parts of an expression in a result position are also in a result position:

- the last expression of `let`, `use` and sequential expressions;
- the branches of `if` and the targets of `match`;
- the body and the handlers of `try/with`, and the body of `try/finally`.

Each expression in a result position MUST be one of these:

- a construction of the case `C`;
- a call to a factory of `C`;
- a call to `raise`, `reraise`, `failwith`, `invalidArg`, `invalidOp` or `nullArg`.

Any other expression is an error, also when it returns `C` at run time:

```fsharp
    [<UnionCaseFactory>]
    static member NewCustomDimensions(side: int) =
        if side > 0 then CustomDimensions(Profile(side, side), side)
        else Inherited side   // error: not a construction of 'CustomDimensions'
```

## Reflection

`FSharpValue.MakeUnion`, `PreComputeUnionConstructor` and `PreComputeUnionConstructorInfo` find the generated method by its name only. A factory makes this lookup ambiguous. FSharp.Core MUST select the method that has `CompilationMappingAttribute` with `SourceConstructFlags.UnionCase`. If no method has this attribute, FSharp.Core uses the name, as today.

## FSharp.Core

```fsharp
namespace Microsoft.FSharp.Core

[<AttributeUsage(AttributeTargets.Method, AllowMultiple = false)>]
[<Sealed>]
type UnionCaseFactoryAttribute =
    inherit Attribute
    new: unit -> UnionCaseFactoryAttribute
```

The feature needs a new language version. With an older language version, FS0023 does not change.

# Changes to the F# spec

- [§8.5.1 Members in Union Types](https://fsharp.github.io/fslang-spec/type-definitions/#851-members-in-union-types): add [Method names](#method-names), [Factories](#factories) and [Body check](#body-check).
- [§8.5.4 Compiled Form of Union Types](https://fsharp.github.io/fslang-spec/type-definitions/#854-compiled-form-of-union-types-for-use-from-other-cli-languages): the method `U.NewC` can have overloads from factories. The generated method has `CompilationMappingAttribute` with `SourceConstructFlags.UnionCase`.
- [§14.2.2 Item-Qualified Lookup](https://fsharp.github.io/fslang-spec/inference-procedures/#1422-item-qualified-lookup): a union case with factories, applied to arguments, resolves as in [Construction](#construction).

# Drawbacks

- `C(args)` can call user code. A reader must look at the type to know the candidates.
- The construction `C(args)` and the pattern `C(pats)` accept different argument lists.
- The body check rejects some correct factories, for example a factory that returns a cached value.
- Reflection code outside FSharp.Core that finds `NewC` by name only fails on a type with a factory.

# Alternatives

- Another name, such as `MakeC` or `CreateC`: the status quo, with two names for one case in each language.
- A module of functions next to the type: no overloads, and C# sees a separate type.
- A static member with the name of the case: the case name then has a member meaning next to its pattern meaning. The suggestion discussion rejects this.
- Factories without the attribute or without the body check: a value from `C(args)` can then fail to match `C _`.
- `U.NewC(args)` in F#: a second name for the case in F#.

# Prior art

- Scala: the companion object of a case class can overload `apply`. `Foo(args)` selects an overload, and the pattern `Foo(pats)` uses `unapply` only.
- Kotlin: a data class can have secondary constructors. Destructuring uses the properties of the primary constructor only.
- [FS-1011](../FSharp-8.0/FS-1011-warn-on-recursive-without-tail-call.md): `[<TailCall>]` is an FSharp.Core attribute that makes the compiler check the body of a method.

# Compatibility

- This is not a breaking change. The warning and the factories apply only to methods that give FS0023 today.
- An older compiler reports FS0023 for the new source code.
- The compiled binary is valid IL: a factory is an overload of the generated method. An older compiler that references this binary sees an ordinary static method `NewC`.
- A program that uses the attribute references a version of FSharp.Core that has the reflection change.

# Interop

- C# and VB see a factory as an overload of the generated method `NewC`, with `[UnionCaseFactory]`.
- A factory does not change the `Deconstruct` methods of [FS-1360](https://github.com/fsharp/fslang-design/pull/865).
- A quotation of `C(args)` that selects a factory contains a `Call` of the factory, not `NewUnionCase`.
- Member constraints (SRTP) see a factory as the static member `NewC`.
- Struct unions and single-case unions follow the same rules.
- Type providers cannot provide union types, so there is no interaction.

# Pragmatics

## Diagnostics

- Warning: a static method `NewC` without `[<UnionCaseFactory>]`.
- Error: `[<UnionCaseFactory>]` on a member that is not a factory.
- Error: an expression in a result position that is not a construction of `C`.
- Error: a method `NewC` with the compiled signature of the generated method.
- FS0041 when overload resolution cannot select one of the case and its factories.

## Tooling

- A use of `C(args)` that selects a factory is a use of the factory symbol. Hover, Go to Definition and Find All References show the factory.
- Signature help on `C(` shows the case and its factories.
- Completion shows the case one time and does not show `NewC`.
- Breakpoints and stepping in a factory work as in other methods.

## Performance

- Existing code does not change.
- A construction that selects the case compiles as today. A factory call is a static method call.
- Overload resolution runs only for a case that has factories.

## Scaling

- Factories for one case: up to 5 in hand-written code. The compiler accepts as many as the method overloads of one name.

## Culture-aware formatting/parsing

None.

# Unresolved questions

1. The name of the attribute: `UnionCaseFactory` or `UnionCaseConstructor`.
