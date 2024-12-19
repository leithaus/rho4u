# MeTTa Architecture Proposal

# Introduction

## Early MeTTa implementations

## Problems

### Leaking Secrets

### All paths

## GSLTs as a way forward

### Well-defined semantics

### Compiles to Rholang

### Rholang is optimal

# Theories of Operational Semantics

Our approach to operational semantics is to define a category Th whose generating objects are "sorts"—the basic atoms out of which the state of the machine is assembled—and whose generating morphisms are term constructors that produce terms of those sorts.

Bill Lawvere pioneered the use of categories to capture grammars modulo equations for structural congruence, so such categories are often called "Lawvere theories".  Lawvere proved that his theories correspond to finitary monads.  Our theories are Lawvere theories equipped with extra structure and stuff; specifically, they are graph-structured magmal multi-sorted lambda theories, or GSLTs for short.

The extra structure and stuff allow us to talk about reductions between terms, bind variables, and define modal types.

## Multi-sorted

Lawvere only considered categories with one generating sort. We need at least two generating sorts, one for terms representing processes, and another for terms representing rewrites between processes.  Todd Trimble gave a [definition of multisorted Lawvere theories](https://ncatlab.org/toddtrimble/published/multisorted+Lawvere+theories) on the nLab and proved a similar monadicity theorem.

## Graph-structured

We distinguish one of these sorts, say P, as the sort of complete machine states, which we call processes.  Rewrites are represented as a different sort R equipped with source and target maps from R to P.  Let Th(Gph) to be the category with two objects and two parallel morphisms between them.  A graph-structured theory Th is one equipped with a functor from Th(Gph) to Th.

## Magmal

We restrict our attention to those programming languages where the source of any rewrite rule is an occurrence of a binary process constructor ⊙.  For example:

* in lambda calculus, beta reduction is of the form ((λx.T) U) \~\> T\[U/x\], whose source is an application of one process to another  
* in pi calculus, the comm rule is of the form for(y \<- x)P | x\!z \~\> P\[z/y\], whose source is a juxtaposition of two processes

Consider the type of terms \<⊙(\[\], A)\>B that when placed into a term context ⊙(\[\], x), where x:A, may reduce to a term of type B. In lambda calculus, this corresponds to the arrow type A \=\> B.  In pi calculus, this is a "possibility" modal type; the dual "necessity" modal type is Caires' rely-guarantee type A ▷ B \[citation\].  If B can depend on x, these are dependent product types.

## Lambda theories

Lawvere's theories were cartesian categories; that is, the theories had finite products.  While it is possible to define the machinery of bound variables and substitution in a Lawvere theory, it is a hassle and not particularly interesting. Instead, we use cartesian closed categories, which handle bound variables and substitution automatically.

## Presentation of finitely generated GSLTs

A presentation of a GSLT consists of

* a finite set of **generating sorts**, including two distinguished sorts P (mnemonic for processes) and R (mnemonic for rewrites) from the graph structure  
* a choice of a distinguished binary morphism ⊙:X x Y \-\> P, called the **interaction**, for some pair of sorts X, Y  
* a finite set of **generating morphisms**, including the two built-in **graph structure morphisms** s, t: R → P. We call generating morphisms with codomain P **term constructors** and those with codomain R **rewrite constructors**.  A rewrite constructor   
                                                r: ∏\_i Z\_i \-\> R   
  whose codomain does not contain a factor of R must factor through the interaction:  
                                     ∃\!f: ∏\_i Z\_i \-\> X x Y. ⊙ ⚬ f \= s ⚬ r  
* a finite set of **equations** between morphisms. We often write r: A \~\> B, where A and B use the same free variables zi, as shorthand for the equations s(r(zi...)) \= A, t(r(zi...)) \= B.

Rewrite term constructors usually come in two flavors:

* the **top-level rewrites** are those whose codomain does not contain a factor of R, and  
* the **in-context rewrites** are those whose codomain does contain a factor of R.

For example, here is a GSLT for the untyped lambda calculus where the evaluation strategy is to reduce to head normal form:

* no generating sorts  
* generating morphisms  
  * Graph structure morphisms s, t: R \-\> P  
  * Term constructors  
    * App: P x P \-\> P  
    * Lam: (P \-\> P) \-\> P  
  * Rewrite constructors  
    * Beta: (P \-\> P) x P \-\> R  
    * Head: R x P \-\> R  
* the interaction is App  
* equations  
  * Beta(A, B): App(Lam(A), B) \~\> ev(A, B)   (top level)  
  * Head(A, B): App(s(A), B) \~\> App(t(A), B)   (in context)

# Generating typed GSLTs from untyped ones

Hypercube functor

# GSLT to Rholang compilation

The general idea is that we send the current state on a channel (say, @0) and then do  
`Interpreter = ∑ for(LHS_Pattern <- @0) { 0!(RHS) | Interpreter }`  
where LHS and RHS mean left- and right-hand side, respectively, and the sum indicates mutually exclusive options.  Rholang does not directly support \+, but it can be added as syntactic sugar. The term above picks one rewrite from among the matching possibilities, then proceeds.

One problem with this approach is that there are potentially an unbounded number of patterns that the term needs to be checked against, and we don't want the interpreter term to be infinitely large.  But if we allow many internal reductions by the interpreter for each reduction by the original GSLT, then we can enumerate the possible left-hand sides:  
`Interpreter = PatternGen(0)`  
`PatternGen(n) = for(LHS_Pattern(n) <- @0) { @0!(RHS(n)) | Interpreter } + PatternGen(n+1)`

A smarter enumeration would do something like  
`Interpreter = for(state <- @0) { PatternGen'(0, state) | @1!(state) }`  
`PatternGen'(n, state) = for(LHS_Pattern'(n, state) <- @1) { @0!(RHS(n)) | Interpreter } + PatternGen(n+1, state)`  
where `LHS_Pattern'(n, state)` uses the structure of state to enumerate only those patterns that state might match.

# RSpace

The Rholang interpreter uses a very efficient data structure called RSpace for storing continuations and matching sends with receives.

## RSpace on Mork

