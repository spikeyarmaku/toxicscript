# ToxicScript
## What's this?
A specification for an easy-to-implement, opinionated, safely embeddable scripting language.

A few key points:
- Strict
- Immutable
- Functional
- Curried by default
- Latently typed
- Mathematically unsound

Inspired by [Haskell](https://haskell.org) and John Shutt's [Kernel](https://web.cs.wpi.edu/~jshutt/kernel.html).

## A high-level implementation overview
To design a language, you need to come up with two things: syntax (how the code looks) and semantics (how it gets evaluated).
### Syntax
We are aiming for a syntax that hits the sweet spot between being easy to parse, and being human-friendly, so S-expressions. They are at the very least not human-antagonistic, and they parse straight into an AST.

The key datatype here is `Expr`, which is either an `Atom`, containing a symbol (e.g. a string), or a `List` of `Expr`s. (There are alternative solutions possible - e.g. one could use nested `Pair`s instead of a `List`.)

### Semantics
For the evaluation model, we use a slightly modified lambda calculus with the environment model (which is kind of a cached version of the substitution model). We will have user-define opaque values, variables, and lambda abstractions, but these abstractions get their argument unevaluated. This is useful because the language implementor can define their own syntax (such as `let`, `if`, `cond`...) with them.

There are two key datatypes here: `Term<T>`, which is a polymorphic type, indexed by the user-defined datatype, and `Env<V>`, which is the environment, assigning values of `V` to `Expr`s. (In our scripting language, we will use `Env<Term<T>>` without exception, but it is a good practice to not lock the implementation of `Env` to any particular datatype.)

The possible values of `Term<T>` are:
- `Val` - containing a user-define opaque value of type `T`
- `Exp` - containing an `Expr` (a variable name, or some code)
- `Abs` - containing a function, which takes an `Env<Term<T>>` and an `Expr`, and gives a `Term<T>` (This is a modified version of lambda calculus' lambda abstraction. The difference is that the original version takes a `Term<T>`, but here it takes an unevaluated expression. Think of it as a curried version of Kernel's $vau combiner.)

As for `Env<T>`, one could get away with a simple dictionary, but there is a problem with string and numeric literals. Most languages treat them as special values, but we don't. We *do* parse strings enclosed by quotes differently, because it's easier, but after parsing, every syntactically correct atom is just an identifier. We can define functions that will turn some identifiers to numbers and strings - provided that the language user actually wants to have such constructs - and to do that, we need the environment to support such mappings. Naturally, we could always model environment lookup as a partial function (if the expression is in the environment, it returns a value, otherwise it returns a terminal value, such as `null`, or `Nothing`), and environment extension would be just composing these lookup functions. But for debugging purposes, one could also choose to have a hybrid approach: use functions for the kind of bulk-lookup definition like strings and numbers (check if an identifier is a valid number, and if so, return the number), and for everything else, use a dictionary.

When it's done, the next step is evaluation. There are three major steps:
- `eval` takes an `Env<Term<T>>`, a `Term<T>` and returns a `Term<T>`. `Val`s and `Abs`s just return themselves, and for `Exp`s, it does a lookup in the environment. If it finds something, returns it, otherwise it just returns the original `Exp`.
- `apply` - takes an `Env<Term<T>>`, a `Term<T>`, an `Expr`, and returns a `Term<T>`. If the second parameter is an `Abs`, it applies it to the environment and the third parameter, and returns the result. Otherwise, it evaluates the second argument, and calls itself with the same environment, the evaluated argument and the original third argument.
- `evalExpr` - takes an `Env<Term<T>>` and an `Expr`, and returns a `Term<T>`. First it looks up if the expression has a value bound to it in the environment, and if so, returns it. Otherwise, it checks if the expression is not empty. If it is, signals an error. If it has exactly one element, it calls itself with the same environment and that element. But if it has more than one, then it takes all but the last, wraps them up in a `List`, calls itself on it with the same environment, and then calls `apply` with the same environment, the result, and the last element of the original list.

#### Standard library

When we are done with all of this, the only thing left is to provide a standard library. Some syntactic forms, such as `lambda` or `let` need to be implemented as primitives, while others, like `pair` can be encoded in the script itself. Some other primitives one might want to implement are equality tests, math operators, and string operations. Recursive let-binding can be done with the help of the Z-combinator (which is the strict version of the Y combinator).

Here is a list of things one might want to provide for a general-purpose language:
- A way to handle numbers and strings (it can be done by examining the variable name, and treating it differently if it is a valid number, or is enclosed by quotes)
- A way to create expressions, such as `lambda`
- A way to bind values to names, such as `let`
- A way to compare values, such as `eq?`
- True/false values (`true`, `false`)
- Branches (`if`)
- Some basic math operations and constants (e.g. `+`, `*`, `-`, `/`, `pi`, `e`)
- Some basic list operations (e.g. `empty?`, `list`)

True / false values can be encoded as functions in lambda calculus. If done that way, `if` becomes trivial - it basically just calls the condition with the two branches as its parameters. Pairs can also be encoded in the exact same way, in which case `fst = true` and `snd = false`.

In the provided implementations, a slightly different way is chosen, but the principle is the same.

## Benchmarks

TODO