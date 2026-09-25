---
title: "Tighten type compatibility rules within translation unit"
document: Nxxxx
date: 2026-09-24
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

# Motivation

TBD

# Approach

Wording for compatibility of types between TUs is basically reverted to C17 state while still handling completeness array types, but now formatted with bullets. Compatibility of types within the same TU now requires them to have the same tag (and have one in the first place), and for their declarations to consist of the same sequence of tokens, where corresponding identifiers denote the same entities.

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

> [6]{.pnum} EXAMPLE 1&nbsp;The following example shows allowed redeclarations of the same structure, union, or enumerated type in the same scope:
>
> > ```c
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
> > ```c
> > enum X { A = 1, B = 1 + 1 };
> > enum X { B = 2, A = 1 };
> >
> > enum Q { C = 1 };
> > enum Q { C = C }; // ok!
> > ```
> > :::
>
> [7]{.pnum} EXAMPLE 2&nbsp;The following example shows invalid redeclarations of the same structure, union, or enumerated type in the same scope:
>
> > ```c
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
> > ```c
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