# Introduction

## Context-Free Grammar

Consists of

1) Terminal symbols (letters in the language) (`a`, `b`, ... `+`)

2) Non-terminal symbols (`A`, `B`, `S`...)

3) A distinguished start symbol `S`

4) Rules of the form `A -> X_1 ... X_n`


## Notation

- Single symbols

  - `a`, `b`, `c`... for terminals

  - `A`, `B`, `C`... for non-terminals

    - `S` for the start symbol

  - `X-Z` for terminals/non-terminals

- Strings of symbols

  - `v-z` for strings of terminals

  - `α`, `β`, `γ`... for strings of terminals/non-terminals

    - `ε` for the empty string

## Derivations

Derivation step: `βAγ => βαγ` if there is a rule `A -> α`

Leftmost derivation step: `wAγ =>l wαγ`

Rightmost derivation step: `βAz =>r βαz`

## Dyck Language Grammar Example

Contains all well bracketed sentences, e.g. `[[()[]]]` but not `[(])`

Rules:
1) `D -> [D]D`

2) `D -> (D)D`

3) `D -> ε`

## Ambiguity

- A grammar is ambiguous if there is more than one possible parse tree for a sentence

- Ambiguity != Non-determinism

## Definition of a Parser

Given a grammar and some string `w`:

- If `w` is in the language of the grammar, parser provides a derivation

- If `w` is not in the language of the grammar, parser rejects it

- Parser always terminates


# LL and LR Parsers
## LL Parser

### State

`<π, w>` where `π` is the stack of predictions, and `w` is the remaining terminal input symbols

### Rules

1) Predict: `<Aπ, w> -> <απ, w>` iff `A -> α`. If we can apply a rule on the head of the stack, apply it

2) Match: `<aπ, aw> -> <π, w>`. If the prediction on the stack matches the input, we have a successful parse and can remove it from the state

3) Start with state `<S, w>` where `w` is the complete input

4) Accept on `<ε, ε>`, when all predictions are gone and there is no more input

5) We fail on either `<ε, w>` or `<π, ε>`

### Example

Given the rules:
```
S -> Lb
L -> aL
L -> ε
```

Parse `aab`:
```
<S, aab>
predict -> <Lb, aab>
predict -> <aLb, aab>
match   -> <Lb, ab>
predict -> <aLb, ab>
match   -> <Lb, b>
predict -> <b, b>
match   -> <ε, ε> // reached accepting state!
```

Parse `ba`:
```
<S, ba>
predict -> <Lb, ba>
predict -> <b, ba>
match   -> <ε, a> // can't perform predict or match, so the input isn't accepted
```

## LR Parser

### State

`<π, w>` where `π` is the stack of reduced input, and `w` is the remaining terminal input symbols

### Rules

1) Shift: `<ρ, aw> -> <ρa, w>`. Move a terminal from the input into the stack

2) Reduce: `<ρα, w> -> <ρA, w>` iff `A -> α`. If we have seen the output from a rule, we can reduce it to the rule's non-terminal lsymbol.

3) Start with `<ε, w>`, so that we can start shifting onto the empty stack to eventually reduce

4) Accept on `<S, ε>`, when there's no input left and we've reduced all seen input to the start state

### Example

Given the rules:
```
S -> A
S -> B
D -> a
A -> ab
B -> ac
```

Parse `ab`:
```
<ε, ab>
shift  -> <a, b>
shift  -> <ab, ε>
reduce -> <A, ε>
reduce -> <S, ε> // reached accepting state!
```

# Making LL and LR Deterministic with LL(1) and LR(0)

- LL and LR are non-deterministic, i.e. at each step there are multiple actions to choose from

- By introducing some extra concepts, we can make them deterministic in most cases

- If the grammar is ambiguous, there will always be cases where these parsers are non-deterministic

## LL(1)

To make LL(1), we add three concepts, `FIRST`, `FOLLOW`, and `NULLABLE`.

`FIRST` and `FOLLOW` are sets of sets, and they are precomputed before runtime from the grammar. This means that they don't add much overhead to the parsing, as they don't need to be recomputed.

### `FIRST`

Formally:
```
b ∈ FIRST(α) iff α *=> bβ
```
This means that a terminal `b` is in `FIRST(α)` if it is possible for `α` to be parsed as some string that begins with `b`

### `FOLLOW`

Formally:
```
b ∈ FOLLOW(X) iff S *=> αXbβ
```
This means that a terminal `b` is in `FOLLOW(X)` if it is possible for `b` to follow `X` in the grammar

### `NULLABLE`

Formally:
```
X ∈ NULLABLE iff X *=> ε
```
This means that `X` is `NULLABLE` if you can apply a series of steps to reduce it to the empty string.

### New Rules

The predict rule now becomes two separate rules:

1) `<Aπ, bw> -> <απ, bw>` iff `A -> α` and `b ∈ FIRST(α)`. We only predict `α` if we know that `α` can begin with the next letter in the input (`b`).

2) `<Aπ, bw> -> <απ, bw>` iff `A -> α` and `α ∈ NULLABLE` and `b ∈ FOLLOW(A)`. We only predict `α` if we know that `α` can be eventually removed, and that it is possible for `b` to follow `A`.

All other rules are the same.

### Example

Rules:
```
S -> Lb
L -> aL
L -> ε
```

Parse `aab`:
```
<S, aab>
predict -> <Lb, aab> // because a ∈ FIRST(Lb)
predict -> <aLb, aab> // because a ∈ FIRST(aL)
match   -> <Lb, ab>
predict -> <aLb, ab> // because a ∈ FIRST(aL)
match   -> <Lb, b>
predict -> <b, b> // because ε ∈ NULLABLE and b ∈ FOLLOW(L)
match   -> <ε, ε>
```

### Conflicts

We can still get conflicts when using `FIRST`, `FOLLOW`, and `NULLABLE`

#### `FIRST`/`FIRST` Conflicts

Given:
1) `A -> α₁` and `A -> α₂`

2) Current state is `<Aπ, bw>`

3) Memberships `b ∈ FIRST(α₁)` and `b ∈ FIRST(α₂)`

4) `α₁ ≠ α₂`

We get a conflict on whether to predict `A -> α₁` or `A -> α₂`.

#### `FIRST`/`FOLLOW` Conflicts

Given:
1) `A -> α₁` and `A -> α₂`

2) Current state is `<Aπ, bw>`

3) Memberships `b ∈ FIRST(α₁)`, `α₂ ∈ NULLABLE` and `b ∈ FOLLOW(A)`

4) `α₁ ≠ α₂`

We get a conflict on whether to predict `A -> α₁` or `A -> α₂`.

### Closure Properties of `FIRST` and `FOLLOW`

Given a rule of the form `A -> αBβCγ`
1) `α ∈ NULLABLE => FIRST(B) ⊆ FIRST(A)` - anything that can be at the beginning of `B` can be at the beginning of `A` if `α` can disappear.

2) `β ∈ NULLABLE => FIRST(C) ⊆ FOLLOW(B)` - anything that can be at the beginning of `C` can follow `B` if `β` can disappear.

3) `γ ∈ NULLABLE => FOLLOW(A) ⊆ FOLLOW(C)` - anything that can come after `A` can come after `C` if `γ` can disappear.

### Calculating `FIRST`, `FOLLOW`, and `NULLABLE`

TODO: Check if we need to know this

## LR(0)

### Items Introduction

- We can make LR deterministic but using items

- An item is a rule with a "pointer" to how much of that rule we have seen so far

- For example `[A -> α∘β]` means that:
  - We have a rule `A -> αβ`

  - We've seen `α` in the input

  - We're expecting to see `β` in the input

- Once we have the item `[A -> γ∘]`, we can reduce with `A -> γ`

### Transitioning between items

We can transition using these rules:
1) `[A -> α∘Xβ] X-> [A -> αX∘β]`

2) `[A -> α∘Bβ] ε-> [B -> ∘γ]`

This is a non-deterministic finite automaton (NFA) - but we want deterministic

### Epsilon Closures

We can make it a deterministic finite automaton (DFA) much like how we make NFA for regular expressions into a DFA - by considering all routes and expanding ε-moves.

For a set of items `s`, we define a set of items `ε-closure(s)`:
- if `i ∈ s`, then `i ∈ ε-closure(s)` (everything in `s` is in `ε-closure(s)`)

- if `i₁ ∈ ε-closer(s)` and `i₁ ε-> i₂`, then `i₂ ∈ ε-closure(s)` (if we can move from something in the closure set to another item using an ε-move, then that item is in the `ε-closure`)

- Repeat this rules until you can't add any more

### New Stack

We now treat the stack of the LR(0) machine as a stack of sets of items. Each set has been expanded to contain all `ε-move`s.

### New Rules

1) Shift: `<σs, aw> -> <σss', w>` iff `[B -> α∘cβ] ∈ s` and `s a-> s'`. As long as there is something left to read in one of the states, and we can transition using `a`, make that transition

2) Reduce: `<σs₁s₂...sn, w> -> <σs₀s', w>` if `[B -> X₁...Xn∘] ∈ sn` and `s₁ B-> s'`. If we've seen the end of a rule, go back down the stack until we can make a transition using that rule, and make that transition

3) Accept: If `<σs, ε>` and `[Sт -> S∘] ∈ s`.

### Conflicts

TODO

### Prefix Property

No prefix of a string in the language is in the language itself.

For example, if `abcd` is in the language, `abc` must not be to satisfy this property.

For example, if `int x = 10;` is in the language, `int x = 10` should not be. This is why we often have semicolons to mark the end of a line in programming languages.

### Example

Given the rules:
```
S -> A
S -> B
A -> ab
B -> ac
D -> ac
```

We get the item states:
```
s₁ = {[Sт -> ∘S], [S -> ∘A], [S -> ∘B], [A -> ∘ab], [B -> ∘ac]}
s₂ = {[A -> a∘b], [B -> a∘c]}
s₃ = {[A -> ab∘]}
s₄ = {[S -> A∘]}
s₅ = {[Sт -> S∘]}
```

Parse `ab`:
```
<s₁, ab>
shift  a-> <s₁s₂, b>
shift  b-> <s₁s₂s₃, ε>
reduce A-> <s₁s₄, ε>
reduce S-> <s₁s₅, ε> // accept because [Sт -> S∘] ∈ s₅!
```
