---
title: "Uniform Call Syntax for explicit-object member functions"
document: D2818R0
date: today
audience:
  - EWG
author:
  - name: Gašper Ažman
    email: <gasper.azman@gmail.com>
toc: true
toc-depth: 2
---

# Introduction

This paper introduces a unification of hidden friends and explicit-object
member functions to allow a limited, but hopefully uncontroversial Uniform Call
Syntax for them.

Unlike the previous proposals on this topic, this one avoids pretty much all controversy.

# Motivation and Prior Art

Why we might want to have UFCS in the language has been covered extensively
already, and Barry Revzin classified all approaches and issues in
[@TaxonomyRevzin].

The short summary boils down to a few points:

- generic code wants to use free-function notation because it's more general
- users with IDEs want to use member-function call notation because it's easier for autocomplete
- it makes it possible to write idiomatic code is more contexts because one has more choices.

While the above list may be a bit short, at least the first point is *so
incredibly important* that the list of papers on the subject is staggering.

**Note:** the **CS:** and **OR:** labels refer to semantic options in the
above-linked article. Please read it, it's short.

The post very helpfully lists much prior art by many of WG21's esteemed
members: Glassborow, Sutter, Stroustrup, Coe, Orr, and Maurer; specifically
[@N1585], [@N4165], [@N4174], [@N4474], [@P0079R0], [@P0131R0], [@P0251R0],
[@P0301R0] and [@P0301R1].

With regards to the taxonomy proposd in [@TaxonomyRevzin], this paper is sortish
in the CS:FreeFindsMember category, but with CS:MemberFindsFree and
CS:ExtensionMethods left as a possible future extensions, as they aren't mutually exclusive.

This paper proposes OR:OneRound, but with ambiguities being impossible
(ill-formed) due to the way this is done.

## Related Work

There have been thoughts around `operator..FQN()`, Kirk Shoop and Ville
Voutilainen seem to have a reasonable grasp of how far that got.

The pipeline operator in its various incarnations also is roughly related - the
people at the head of that one seem to be Barry Revzin, Colby Pike [@P2011R1]
and Isabella Muerte [@P1282R0].

# Proposal

We propose that marking an explicit-object member function as a `friend` (to
parrot inline friend function declarations, specifically hidden friends) would
also make it callable via free-function argument-dependent-lookup.

Example:

```cpp
struct S {
  friend int f(this S) {
    return 42;
  }
};

int g() {
  S{}.f();  // OK, returns 42
  f(S{});   // OK, same
  (f)(S{}); // Error; f can only be found by ADL
}

int f(S);   // ill-formed, int f(S) conflicts with S::f(S)
int f(int); // OK
```

That's pretty much it.

## Alternatives to syntax

While `friend` communicates exactly what this actually does, it only does so if
one knows about hidden friends.

We *could* use a context-sensitive keyword here, like `adl`, or
`associated(type-id-or-class-template-id, ...)`, as Lewis Baker's paper
[@D2823R0] proposes, and just say that these functions are callable via ADL for
the classes mentioned in the parentheses using this free-function notation. We
still need this paper's wording to get that accomplished.

The main issue with `associated(type-id-or-class-template-id, ...)` is that
type-ids cannot be dependent, as that would be equivalent to adding the
overload to every namespace.

`friend` does not introduce this complexity, but the additional power might be
useful enough to choose as preferred syntax for the semantics this paper
proposes.

# Ruminations from certain past EWG chairs on UFCS design feasibility

(very slightly paraphrased)

In general, concerns arise from the existing guarantee that member functions
simply don't mix with ADL. Using a member call syntax means ADL doesn't happen,
but declaring a function a member also makes it not dance with ADL overloads in
an overload set. The first guarantee is paramount, the second is Really Nice To
Have.

Granted, the proposal still allows having the second guarantee. Don't mark your
functions `adl`, done.

# How is this different from prior art?

## There can be no confusion about which function is preferred

There is only one function in the first place.

The `friend` syntax signals the behavior exactly. The declaration of the member
function is *also* injected into the type's "hidden" namespace as a hidden
friend after notionally removing the keyword `this` from the argument list.

This is OK, because explicit-object member functions have free-function type,
and their bodies behave as if they were free functions, so we're not lying.
We're doing exactly what it looks like.

## It's precise

You opt-in to UFCS on a per-declaration basis. This matters because UFCS is
primarily about enabling generic code to use a given type, and gives precise
control about both the free-function and member-function interfaces of a given
class. When both interfaces should provide a given signature, this is the only
proposal that lets you _just do that and only that_, without impacting other
parts of either overload set.

## It's simple and minimal

It just merges two things we already have - hidden friends, and explicit-object
member functions. No need to remember which comes first or how a given function
is defined - both syntaxes always dispatch to the _only_ implementation.

## It's modular

It does not propose, but does not preclude, future extensions into, well, extension methods.
See the Future Extensions chapter.

# Future Extensions

While the author of this proposal is of a mild opinion that Extension Methods
(CS:ExtensionMethods) would not carry their weight in C++, this paper is
specifically neutral on this topic and reserves the only plausible syntax for
them:

```cpp
// Disclaimer: NOT PROPOSED IN THIS PAPER
struct E {};
int h(this E) { return 42; } // look ma, I'm not a member of E
int main() {
   h(E{}); // ok, obviously, since h is declared outside of E
   E{}.h(); // also OK, h found by ADL and specifies where to put `this`.
}
```

There is one caveat - in this case, if `E` declares an `h(this E)`, it would
conflict at declaration time, since this proposal already specifies that
behavior.

## Ruminations of certain past EWG chairs on the design of the extension

(very slightly paraphrased)

We can do a double-requirement. The extension needs to be declared with an explicit
object parameter **and** *the class needs to say that it allows such
extending*. Then we're good with the extension too.

# Questions for EWG

1) are we OK choosing the _OR:OneRound (+no conflicting declarations)_
   approach, knowing that it eliminates OR:TwoRoundsPreferAsWritten,
   OR:TwoRoundsMemberFirst and OR:OneRoundPreferMembers for all UFCS-related
   features in the language?
2) Do we want a different syntax from `friend` to signal exactly what `friend`
   does in this context, like a context-sensitive keyword `associated(type-id, ...)`?
3) Do we find UFCS eliminates a significant-enough portion of library
   boilerplate in the cases where a class needs to provide both interfaces for
   this feature to be worth the implementation cost?

# FAQ

## Why are you writing another paper about UFCS?

Because this is a novel direction that might actually fit the language and
pass.

## Has this been implemented?

No, but given that it uses a syntax that is ill-formed in C++23, and that it
only inserts an alias to the same function that otherwise works exactly like a
hidden friend, I *really* don't have implementation concerns. Any compiler that
implements [@P0847R7] will have zero issues implementing this paper.

## Are you going to bring the extension methods paper too?

No. I don't need them, and injecting functions into the space of member
functions that is explicitly given to the class designer is wrong unless
properly scoped. I don't know how to properly scope it. If you do, the only
reasonable syntax is above.

## Can I put `this` not on the first argument?

Not yet. I might bring that paper if this one passes, but separately.


---
references:
  - id: TaxonomyRevzin
    citatation-label: TaxonomyRevzin
    title: "What is unified function call syntax anyway?"
    author:
      - family: Revzin
        given: Barry
    URL: https://brevzin.github.io/c++/2019/04/13/ufcs-history/
  - id: D2823R0
    citation-label: D2823R0
    title: "Declaring hidden non-friend functions to be found by argument-dependent-lookup"
    author:
      - family: Baker
        given: Lewis
    URL: https://wg21.link/D2823R0
  - id: D2822R0
    citation-label: D2822R0
    title: "Providing user control of associated entities of class types"
    author:
      - family: Baker
        given: Lewis
    URL: https://wg21.link/D2822R0
---
