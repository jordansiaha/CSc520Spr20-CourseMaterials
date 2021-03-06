# Getting Started
 * Note credits: Kathleen Fisher and Norman Ramsey,
   https://www.cs.tufts.edu/comp/105-2017f/notes.html

 * [Handout: 105 Impcore Semantics, Part 1](https://www.cs.tufts.edu/comp/105/handouts/ImpcoreSemantics1.pdf)

 * Today: Abstract Syntax and Operational Semantics

 * Discussion: Two things you learned last class.

# Programming-language semantics

Semantics means meaning.

## What problem are we trying to solve?

  *Know what’s supposed to happen when you run the code*

Ways of knowing:

* People learn from examples

 * You can build intuition from words
   (Book is full of examples and words)

 * To know exactly, *unambiguously*, you need more precision

**Q:** Does anyone know the beginner exercise “make a peanut butter 
and jelly sandwich”? (Videos on YouTube)

 * You can watch and learn, but a computer can’t.

 * "Put the peanut butter on the bread"

## Why bother with precise semantics?

Same reason as other forms of math:

 * Distill understanding

 * Express it in sharable way

 * Prove useful properties. For example:
   * private information doesn’t leak
   * device driver can’t crash the OS kernel
   * compiler optimizations preserve program meaning
   * Most important for you: *things that look different are actually the same*

Plus, needed to build language implementation and tests

The programming languages you encounter after 520 will certainly look different 
from what we study this term. But most of them will actually be the same. 
Studying semantics helps you identify that.

The idea: **The skills you learn in this class will apply**

## Behavior decomposes
We want a computational notion of meaning.  What happens when we run `(* y 3)`?

We must know something about `*`, `y`, `3`, and function application.

Knowledge is expressed inductively

 * Atomic forms: Describe behavior directly (e.g., constants, variables)

 * Compound forms: Behavior specified by composing behaviors of parts

(Non)-Example of compositionality: Spelling/pronunciation in English

 * `fish` vs `ghoti`
 * Both composed from letters, but no rules of composition for pronunciation.

*By design, programming languages more orderly than natural language.*

## Review: Concrete syntax for Impcore

Definitions and expressions:
```
def ::= (define f (x1 ... xn) exp)
     |  (val x exp)                
     |  exp
     |  (use filename)            
     |  (check-expect exp1 exp2)
     |  (check-error exp)

exp ::= integer-literal      ;; atomic forms
     |  variable-name
     |  (set x exp)          ;; compound forms
     |  (if exp1 exp2 exp3)
     |  (while exp1 exp2)
     |  (begin exp1 ... expn)
     |  (function-name exp1 ... expn)
```

## How to define behaviors inductively

Expressions only

 * Base cases (plural): numerals, names

 * Inductive steps: compound forms

   To determine behavior of a compound form, look at behaviors of its parts
 
## First, simplify the task of definition

What’s different? What’s the same?
```
 x = 3;               (set x 3)

 while (i * i < n)    (while (< (* i i) n)
   i = i + 1;            (set i (+ i 1)))
```
**Abstract away** gratuitous differences

(See the bones beneath the flesh)

## Abstract syntax

Same inductive structure as BNF

More uniform notation

Good representation in computer

Concrete syntax: sequence of symbols

Abstract syntax: ???

## The abstraction is a tree

The abstract-syntax tree (AST):
```
Exp = LITERAL (Value)
    | VAR     (Name)
    | SET     (Name name, Exp exp)
    | IFX     (Exp cond, Exp true, Exp false)
    | WHILEX  (Exp cond, Exp exp)
    | BEGIN   (Explist)
    | APPLY   (Name name, Explist actuals)
```
One kind of "application" for both user-defined and primitive functions.

## ASTs
Question: What do we assign behavior to?

Answer: The Abstract Syntax Tree (AST) of the program.

 * An AST is a data structure that represents a program.

 * A parser converts program text into an AST.

Question: How can we represent all while loops?
```
  while (i < n && a[i] < x) { i++ }
```
Answer:

 * Tag code as a while loop
 * Identify the condition, which can be any expression
 * Identify the body, which can be any expression

As a data structure:

 * `WHILEX(exp1, exp2)`, where
 * `exp1` is the representation of `(i < n && a[i] < x)`, and
 * `exp2` is the representation of `i++`

In class: what about all function applications?

## In C, trees are a bit fiddly
```
typedef struct Exp *Exp;
typedef enum {
  LITERAL, VAR, SET, IFX, WHILEX, BEGIN, APPLY
} Expalt;        /* which alternative is it? */

struct Exp {  // only two fields: 'alt' and 'u'!
    Expalt alt;
    union {
        Value literal;
        Name var;
        struct { Name name; Exp exp; } set;
        struct { Exp cond; Exp true; Exp false; } ifx;
        struct { Exp cond; Exp exp; } whilex;
        Explist begin;
        struct { Name name; Explist actuals; } apply;
    } u;
};
```

In class: Draw the while loop example tree and apply example tree.

## Let’s picture some more trees

An expression:
```
  (f x (* y 3))
```
(Representation uses `Explist`)

A definition:
```
  (define abs (n)
    (if (< n 0) (- 0 n) n))
```

## Behaviors of ASTs, part I: Atomic forms

Numeral: stands for a value

Name: stands for what?

## In Impcore, a name stands for a value

**Environment** associates each **variable** with one **value**

Written `\rho = \{ x_1 \mapsto n_1, ... x_k \mapsto n_k\}`
 * <img src="02-semantics-intro/environment.jpeg">
 * associates variable `x_i` with value `n_k`.

Environment is a **finite map**, aka **partial function**

`x \in dom \rho`
 * <img src="02-semantics-intro/x-in-dom-rho.jpeg">
 * means `x` is defined in environment `\rho`

`\rho(x)`
 * <img src="02-semantics-intro/rho-x.jpeg">
 * means the value of `x` in environment `\rho`

`\rho \{ x \mapsto v \}`
 * <img src="02-semantics-intro/extend-rho.jpeg">
 * means extends/modifies environment `\rho` to map `x` to `v` 


## Environment in C, abstractly

An abstract type:
```
    typedef struct Valenv *Valenv;
    
    Valenv mkValenv (Namelist vars, Valulist vals);
    bool isvalbound (Name name, Valenv env);
    Value fetchval  (Name name, Valenv env);
    void bindval    (Name name, Value val, Valenv env);
```

## Environment is pointy-headed theory

You may also hear:
 * Symbol table
 * Name space

Influence of environment is "scope rules"

In what part of code does environment govern?

## Find behavior using environment

Recall
```
  (* y 3)   ;; what does it mean?
```
Your thoughts?

## Impcore uses three environments

Global variables ξ (or `xi`)

Functions ϕ (or `phi`)

Formal parameters ρ (or `rho`)

There are no local variables
 * Just like awk; if you need temps, use extra formal parameters
 * For HW2, you'll add local variables

Function environment ϕ not shared with variables—just like Perl

## Syntax and Environments determine behavior

Behavior is called **evaluation**

 * Expression is **evaluated** in **environment** to produce **value**

 * "The environment" has three parts: globals, formals, functions

Evaluation is

 * **Specified** using **inference rules** (math)

 * Implemented using **interpreter** (code)

You know code. You will learn math.

## Key ideas apply to any language

Expressions

Values

Rules

## Rules written using operational semantics

Evaluation on an **abstract machine**

 * Concise, precise definition

 * Guide to build interpreter

 * Prove "evaluation deterministic" or "environments can be on a stack"

Idea: "mathematical interpreter", formal rules for interpretation

## Syntax and environments determine meaning

Initial state of abstract machine:
 * `\langle e, \xi, \phi, \rho \rangle`
 * <img src="02-semantics-intro/initial-state.jpeg">

State `\langle e, xi, phi, rho \rangle` is
 * `e`, Expression being evaluated
 * `xi`, Values of global variables
 * `phi`, Definitions of functions
 * `rho`, Values of formal parameters
 
Three environments determine what is in scope.

## Meaning written as "Evaluation judgement"

We write
 * `\langle e, \xi, \phi, \rho \rangle \Downarrow \langle v, \xi', \phi, \rho' \rangle`
 * <img src="02-semantics-intro/eval-judgement.jpeg">

(**Big-step** judgement form.)

Notes:
 * `xi` and `xi'` may differ
 * `rho` and `rho'` may differ
 * `phi` must equal `phi`

Question: what do we know about globals?  functions?

With that as background, we can now dive in to the semantics for Impcore!

## Impcore atomic form: Literal

"Literal" **generalizes** "numeral"

`\inferrule[LITERAL]{ }{\langle \mbox{LITERAL}(v),\xi,\phi,\rho \rangle \Downarrow \langle v,\xi,\phi,\rho \rangle}`

<img src="02-semantics-intro/literal-semantics.png">

Numeral converted to `LITERAL(v)` in **parser**

NOTE: for the `\inferrule` latex definition using mathpartir package

## Impcore atomic form: Variable name

`\inferrule[FormalVar]{x \in \mbox{dom } \rho}{\langle \mbox{VAR}(x),\xi,\phi,\rho \rangle \Downarrow \langle \rho(x),\xi,\phi,\rho \rangle}`

<img src="02-semantics-intro/formalvar-semantics.png">


`\inferrule[GlobalVar]{x \notin \mbox{dom } \rho \\  x \in \mbox{dom } \xi}{\langle \mbox{VAR}(x),\xi,\phi,\rho \rangle \Downarrow \langle \xi(x),\xi,\phi,\rho \rangle}`

<img src="02-semantics-intro/globalvar-semantics.png">

Parameters hide global variables.

## Impcore compound form: Assignment

In `SET(x,e)`, `e` is any expression

`\inferrule[FormalAssign]{x \in \mbox{dom } \rho \\
\langle e, \xi, \phi, \rho \rangle \Downarrow \langle v, \xi', \phi, \rho' \rangle}
{\langle \mbox{SET}(x,e),\xi,\phi,\rho \rangle \Downarrow \langle v,\xi',\phi,\rho'\{x \mapsto v\} \rangle}`

<img src="02-semantics-intro/formalassign-semantics.png">


`\inferrule[GlobalAssign]{x \notin \mbox{dom } \rho \\  x \in \mbox{dom } \xi \\
\langle e, \xi, \phi, \rho \rangle \Downarrow \langle v, \xi', \phi, \rho' \rangle}
{\langle \mbox{SET}(x,e),\xi,\phi,\rho \rangle \Downarrow \langle v,\xi'\{x \mapsto v\},\phi,\rho' \rangle}`


<img src="02-semantics-intro/globalassign-semantics.png">


Impcore can assign only to **existing** variables
