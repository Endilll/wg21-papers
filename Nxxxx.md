---
title: "Tighten type compatibility rules within translation unit"
document: Nxxxx
date: 2026-10-15
audience: WG14
author:
  - name: Vlad Serebrennikov
    email: <serebrennikov.vladislav@gmail.com>
  - name: Aaron Ballman
    email: <aaron@aaronballman.com>
toc: true
toc-depth: 2
---

# Abstract

[N3037](https://open-std.org/JTC1/SC22/WG14/www/docs/n3037.pdf) "Improved Rules for Tag Compatibility" applied cross-translation-unit rules of type compatibility within the same translation unit.
We believe that those rules doesn't need to be as permissive within a translation unit to support the motivation of N3037, as that causes a number of side effects we consider undesireable.

# Why is N2366 being referred to?

While N3037 was the paper that was voted into C23, this paper quotes an earlier revision of it, [N2366](https://open-std.org/JTC1/SC22/WG14/www/docs/n2366.pdf), because that was the last revision that expanded on the motivation and background. We realize that not all of N2366 made it into C23, so we're quoting only part of it that we think are relevant for N3037.

# Motivation

## Cross-TU compatibility rules are just admitting ABI reality

N2366: _"While all other types follow basic structural typing rules for identity and compatibility, tagged types have much more complicated rules which are different when applied to types declared inside the same or in different translation units. [. . .] While it is a common and expected property of type systems that mutual compatibility of types is transitive, this is not the case in C: The sets of types which are mutually compatible to each other do not form equivalence classes."_

C17-era compatibility rules across TUs were not designed from first principles — they were admitting the reality of how translation phase 8 is implemented between compilers and linkers, and they were shaped by that reality. Linkers perform no semantic checks: if the resulting program happens to work, it works.

N2366: _"It should be noted that many existing compilers already implement a version of this algorithm, because they need to to be able to determine compatibility of types across translation units when performing global optimization – removing the difference between compatibility of types declared in the same translation unit and types declared between translation unit would simplify such compilers."_

That missed two important aspects for at least one implementation (Clang).

First, intra-TU type compatibility rules have to be enforces by the compiler frontend during translation phase 7, whereas cross-TU compatibility rules during LTO are enforced by the middle-end during translation phase 8. Front-end and middle-end are different parts of the compiler that operate on vastly different program representations (AST vs LLVM IR), which makes it quite hard to share code between them, including the proposed algorithm for evaluating structural equivalence.

Second, the structural equivalence check that LTO can (and need to) perform is severely limited by the representation it operates on. The following C structure:

```c
struct S {
  int a;
  const volatile _Atomic int b;
  _BitInt(17) c;
};
```

is transformed into the following LLVM IR type:

```llvm
%struct.S = type { i32, i32, i32 }
```

As you can see, the pre-existing type compatibility check for the purposes of global optimization ignores a lot of semantics important for the Standard C. Given that this has been deployed for years, if not for a decade, this loose check is sufficient to perform global optimizations.

The example above is based on Clang implementation details, but we expect that other implementations that support LTO share the properties mentioned above.

## Global static analysis works in C++ despite the One Definition Rule

It should be noted that C++ always had a rule that you can have only one definition of an entity in a program, which is then relaxed to accomodate for entities defined in headers. As far as we know, this never prevented static analyzers from performing analysis even on big codebases, and we're not aware of any intents to relax that rule for C++ in the same vein as N3037 did.

## It is not trivial to support different definitions in the same TU

N2366: _"This is not expected to be a significant burden to implementors."_

Clang front-end, by the virtue of implementing C++20 modules, already had a structural equivalence check not too dissimilar to what N3037 needs. However, switching it on in C23 mode was just a start of a long trail of bugs ([#141724](https://github.com/llvm/llvm-project/issues/141724), [#149965](https://github.com/llvm/llvm-project/issues/149965), [#150594](https://github.com/llvm/llvm-project/issues/150594), [#151394](https://github.com/llvm/llvm-project/issues/151394), [#186196](https://github.com/llvm/llvm-project/issues/186196), [#188476](https://github.com/llvm/llvm-project/issues/188476), [#201647](https://github.com/llvm/llvm-project/issues/201647)) and fixes ([#138516](https://github.com/llvm/llvm-project/pull/138516), [#141783](https://github.com/llvm/llvm-project/pull/141783), [#150282](https://github.com/llvm/llvm-project/pull/150282), [#150946](https://github.com/llvm/llvm-project/pull/150946), [#168769](https://github.com/llvm/llvm-project/pull/168769)).

N3037 affects the semantic analysis of the code that goes into headers, which might be consumed by a C++ compiler. Given the amount of trouble we went through so far to make N3037 work in C23 language mode, it seems unlikely that Clang is going to support this feature in any C++ mode, because C++ has even more features that interact with N3037. This means that C projects that would like their headers to be working in C++ compilers will have to abandon a significant chunk of C++ users which use Clang.

## Go to Definition

Before N3037, when tools like clangd wanted to show users the definition of a type, there was no question what to show, because there was only one definition of it in the source code, even if included into multiple TUs. With N3037, there can be multiple definitions of a type in the same TU which doesn't have to be tokenwise identical, so it matters which one the tool picks.

In the example below, there is no single definition that contains all the information about the type:

```c
struct S {
  [[deprecated("don't use 'a'")]] int a;
  int b;
};

struct S {
  int a;
  [[deprecated("don't use 'b'")]] int b;
};
```

# Approach

Wording for compatibility of types between TUs is basically reverted to C17 state while still handling completeness of array types, but now formatted with bullets. Compatibility of types within the same TU now requires them to have the same tag (and have one in the first place), and for their declarations to consist of the same sequence of tokens, where corresponding identifiers denote the same entities.

# Proposed wording

Wording changes are relative to [N3886](https://open-std.org/JTC1/SC22/WG14/www/docs/n3886.pdf).

## 6.2.7 Compatible type and composite type

> [1]{.pnum} Two types are _compatible types_ if they are the same. Additional rules for determining whether two types are compatible are described in 6.7.3 for type specifiers, in 6.7.4 for type qualifiers, and in 6.7.7 for declarators.^33)^
> Moreover, two [complete]{.rm} structure, union, or enumerated types declared [with the same tag]{.rm} [in different translation units]{.add} are compatible if [members satisfy]{.rm} the following requirements [are satisfied]{.add}:
>
> - [if one is declared with a tag, the other is declared with the same tag,]{.add}
> - [and, if both are completed somewhere in their respective translation units:]{.add}
>   - there [shall be]{.rm} [is]{.add} a one-to-one correspondence between their members such that each pair of corresponding members are declared with compatible types;
>   - if one member of the pair is declared with a complete array type, the other is declared with a complete array type;
>   - if one member of the pair is declared with an alignment specifier, the other is declared with an equivalent alignment specifier;
>   - and, if one member of the pair is declared with a name, the other is declared with the same name.
>
> For two structures, corresponding members shall be declared in the same order.
> [For two unions declared in the same translation unit, corresponding members shall be declared in the same order.]{.rm}
> For two structures or unions, corresponding bit-fields shall have the same widths.
> For two enumerations, corresponding members shall have the same values; if one has a fixed underlying type, then the other shall have a compatible fixed underlying type.
> For determining type compatibility, anonymous structures and unions are considered a regular member of the containing structure or union type, and the type of an anonymous structure or union is considered compatible with the type of another anonymous structure or union, respectively, if their members fulfill the preceding requirements.
>
> Furthermore, two structure, union, or enumerated types declared in [separate]{.rm} [the same]{.add} translation units are compatible [in the following cases]{.rm} [if]{.add}:
>
> - [both are declared without tags and they fulfill the preceding requirements;]{.rm}
> - both have the same tag [and are completed somewhere in their respective translation units and they fulfill the preceding requirements]{.rm};
> - [both have the same tag and at least one of the two types is not completed in its translation unit.]{.rm}
> - [their declarations consist of the same sequence of tokens;]{.add}
> - [and, corresponding identifiers in their declarations denote the same entity.]{.add}
>
> Otherwise, [the]{.rm} [it is implementation-defined whether two]{.add} structure, union, or enumerated types are [incompatible]{.rm} [compatible]{.add}.^[34)]{.rm}^
>
> ::: add
> ::: example
> ```cpp
> constexpr int n = 4;
> void f() {
>   struct S { int arr[n]; };
>   constexpr int n = 4;
>   struct S { int arr[n]; }; // error: n denotes a different entity
> }
> ```
> :::
> :::
>
> [2]{.pnum} All declarations that refer to the same object or function shall have compatible type; otherwise, the behavior is undefined.
>
> [footnote33]{.pnum} Two types are not expected to be identical to be compatible.
>
> [footnote34]{.pnum} [A structure, union, or enumerated type without a tag [or an incomplete structure, union or enumerated type]{.rm} is not compatible with any other structure, union or enum type declared in the same translation unit.]{.rm}

## 6.7.2.3 Tags

### Constraints

> [1]{.pnum} Where two declarations that use the same tag declare the same type, they shall both use the same choice of `struct`, `union`, or `enum`.
> [If two declarations of the same type have a member-declaration or enumerator-list, one shall not be nested within the other and both declarations shall fulfill all requirements of compatible types (6.2.7) with the additional requirement that corresponding members of structure or union types shall have the same (and not merely compatible) types.]{.rm}
>
> [2]{.pnum} [. . .]

::: draftnote
The removed restriction was only working within a TU, and is now subsumed by much stricter restriction of token sequences being the same.
:::

### Semantics

Remove two examples at the end of paragraph 6:

> [6]{.pnum} EXAMPLE 1&nbsp;The following example shows allowed redeclarations of the same structure, union, or enumerated type in the same scope:
>
> > ```
> > struct foo { struct { int x; }; };
> > struct foo { struct { int x; }; };
> >
> > union bar { int x; float y; };
> > union bar { int x; float y; };
> >
> > typedef struct q { int x; } q_t;
> > typedef struct q { int x; } q_t;
> >
> > void foo(void)
> > {
> >     struct S { int x; };
> >     struct T { struct S s; };
> >     struct S { int x; };
> >     struct T { struct S s; };
> > }
> > ```
> > ::: rm
> > ```
> > enum X { A = 1, B = 1 + 1 };
> > enum X { B = 2, A = 1 };
> >
> > enum Q { C = 1 };
> > enum Q { C = C }; // ok!
> > ```
> > :::

Add two examples to the end of paragraph 7:

> [7]{.pnum} EXAMPLE 2&nbsp;The following example shows invalid redeclarations of the same structure, union, or enumerated type in the same scope:
>
> > ```
> > struct foo { int (*p)[3]; };
> > struct foo { int (*p)[]; };    // member has different type
> >
> > union bar { int x; float y; };
> > union bar { int z; float y; }; // member has different name
> >
> > union purr { int x; float y; };
> > union purr { float y; int x; }; // members have different order
> > // purr only valid if each union purr is in
> > // a different translation unit
> >
> > typedef struct { int x; } q_t;
> > typedef struct { int x; } q_t; // not the same type
> >
> > struct S { int x; };
> > void foo(void)
> > {
> >     struct T { struct S s; };
> >     struct S { int x; };
> >     struct T { struct S s; }; // struct S not the same type
> > }
> >
> > enum X { A = 1, B = 2 };
> > enum X { A = 1, B = 3 };     // different enumeration constant
> >
> > enum R { C = 1 };
> > enum Q { C = 1 };            // conflicting enumeration constant
> > ```
> > ::: add
> > ```
> > enum O { A = 1, B = 1 + 1 };
> > enum O { A = 1, B = 2 };     // different token sequence
> >
> > const int i;
> > void bar(void)
> > {
> >     struct P { typeof_unqual(i) x; };
> >     const int i;
> >     struct P { typeof_unqual(i) x; }; // i denotes a different entity
> > }
> > ```
> > :::