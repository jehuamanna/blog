---
title: Regex, Automata, and Computation Are the Same Thing
description: Automata, computation, and regex turn out to be three vocabularies for one underlying idea. - Study Notes
pubDate: 2026-09-07T12:00:00Z
tags:
  - automata
  - regex
  - computation
---

*Part 7 of a series on automata, computation, and why regex never lies to you.*

<div class="series-nav">
<p>Previously in this series</p>
<ol>
<li><a href="/blog/react-state-machine-is-a-dfa/">Your React State Machine Is Secretly a DFA</a></li>
<li><a href="/blog/what-is-a-language/">What Even Is a "Language" in Computer Science?</a></li>
<li><a href="/blog/elevator-state-machine/">Modeling an Elevator With a State Machine</a></li>
<li><a href="/blog/nfa-vs-dfa/">Nondeterminism Doesn't Make Machines More Powerful (And Here's the Proof)</a></li>
<li><a href="/blog/building-a-regex-engine/">Building a Regex Engine From Scratch: Thompson's Construction to DFA</a></li>
<li><a href="/blog/catastrophic-backtracking/">Why (a+)+b Can Freeze JavaScript But Not grep</a></li>
</ol>
</div>

The title of this post isn't a metaphor.

Six parts ago, we started with a `useReducer` call. Along the way we covered a formal definition of "language," an elevator that opens doors, a 1959 theorem about nondeterminism, a regex engine built from nothing, and a pattern that can hang a browser tab. It probably felt like six separate topics.

It wasn't. It was a single object, seen from three different angles.

Automata theory describes this object as a **machine**. Computation theory describes it as a **process**. Regex describes it as a **notation**. Three different vocabularies, from three different research traditions, and there are theorems proving that all three descriptions are interchangeable. You now have every piece needed to see this as one single idea.

## Angle one: automata are a memory ladder

Strip any abstract machine down to its most basic parts and you get: a set of states, a transition rule, a starting point, and a way to decide acceptance. This basic structure describes every automaton that exists. The finite automaton from Parts 1 through 4 isn't *the* automaton, it's the weakest one, the ground floor of the building.

What separates each floor from the next is exactly one variable: **how much extra memory you're allowed to have, beyond just knowing "which state am I in."**

| Machine | Extra memory | Recognizes | Example language |
|---|---|---|---|
| Finite automaton (DFA/NFA) | none | Regular | `a*b*` |
| Pushdown automaton | a stack | Context-free | balanced parens, JS syntax |
| Linear-bounded automaton | a tape, bounded by input length | Context-sensitive | (a middle rung, rarely load-bearing in practice) |
| Turing machine | an infinite read/write tape | Recursively enumerable | general computation |

This table describes the **Chomsky hierarchy** (named after Noam Chomsky, 1956, the same year as Kleene's theorem, and that's not a coincidence). Each level strictly contains the level below it: every regular language is also context-free, but not every context-free language is regular. This works in the other direction too. Each machine class recognizes *exactly* the language class next to it in the table. Not approximately, exactly. Giving a machine a stack doesn't simply make it a better finite state machine, it moves the machine into a class that can provably solve a larger set of problems.

This explains why the finding in [Part 2](/blog/what-is-a-language/) had to be true. `L(JavaScript)` isn't a regular language, because nesting depth has no fixed limit, and the word "finite" is literally part of the machine's name. Add a stack, though, push a value on `{`, pop a value on `}`, and the problem becomes trivial. The difference between your lexer and your parser is exactly one line in this table.

## Angle two: computation is the same recipe, repeated

Now, instead of thinking about the machines themselves, look at what they actually *do*.

Take a **configuration**: a complete snapshot of the machine at one moment during its run, containing everything needed to determine what happens next. Apply the transition function to this configuration and you get the next configuration. Repeat until the machine halts. Then check whether the result is accept or reject.

That's the entire idea. That's computation. Not about any single machine, just this repeating pattern.

As you move up the ladder, the only thing that changes is what's *inside* a configuration:

- **FSM:** `(current state, remaining input)`
- **PDA:** `(current state, remaining input, stack)`
- **Turing machine:** `(current state, tape contents, head position)`

The overall shape never changes: configuration, transition, configuration, repeated until halt. This loop is the complete meaning behind the phrase "what is computation," and every machine in this series has been running exactly this loop. Your form reducer ran this loop using the smallest configuration that still counts as valid. The elevator ran the same loop, with motors attached to the output. The DFA in [Part 5](/blog/building-a-regex-engine/) ran this loop once for every character, which is exactly why it ran in linear time. One configuration per symbol, no revisiting of previous states, no choice involved.

Computational power has nothing to do with speed or cleverness. It's really a question of how much information you're allowed to carry between steps. That's the entire subject.

## Angle three: regex names one point on the ladder

This is where everything connects.

> **Kleene's Theorem** (Stephen Kleene, 1956): A language is regular **if and only if** some regular expression describes it, **if and only if** some finite automaton recognizes it.

Pay close attention to the logical wording here. This isn't saying "regex is inspired by automata." It says *if and only if*, and that holds true in both directions. A regular expression, a DFA, and a regular language are three different descriptions of the same object: an algebraic string of symbols, a graph made of circles and arrows, and a set of strings. Have any one of these three descriptions and the other two can be derived from it mechanically, without losing any information.

This means the pipeline from [Part 5](/blog/building-a-regex-engine/) is more than just one possible way to implement regex. It's a mathematical necessity:

```
regex string
   │  Thompson's construction      (Part 5)
   ▼
NFA
   │  subset construction          (Part 4)
   ▼
DFA
   │  one step per input symbol    (Part 5)
   ▼
accept / reject
```

Every arrow in this diagram represents a theorem, not just a design choice someone made. Thompson's construction proves that every regex can be turned into an NFA. The subset construction is the Rabin-Scott result: it shows that the power of an NFA collapses down to exactly the power of a DFA, meaning nondeterminism gives you convenience but exactly zero extra computational power. The matching loop itself is simply angle two's recipe, applied at the bottom rung of the ladder. Nobody chose this architecture because they preferred it. It's the only architecture the mathematics allows.

[Part 6](/blog/catastrophic-backtracking/) is really just another view of this same fact. Catastrophic backtracking isn't a bug in regex, it's what happens when a regex engine steps outside the boundaries of this ladder. The moment a pattern is no longer a regular language, the guarantee of linear-time matching disappears along with it, and a pattern like `(a+)+b` makes you pay for that difference.

## What you actually did

Look back at everything that happened across the previous six posts.

You took a reducer you'd already written a hundred times before and extracted its formal mathematical structure. You defined precisely what a "language" is. You found the exact boundary where finite memory stops being enough. You watched a proof showing that nondeterminism doesn't add any real power. Then you built the regex to NFA to DFA pipeline by hand, yourself, and ran it.

That entire process is Kleene's theorem. You didn't just read about it, you walked through its full construction, from start to end, yourself. Kleene originally worked this out while modeling neural networks, before most people had ever touched an electronic computer. Chomsky developed his hierarchy while thinking about grammar, not compilers. Rabin and Scott later shared a Turing Award for the argument discussed in [Part 4](/blog/nfa-vs-dfa/).

Your `useReducer` transition function and their δ aren't merely similar, they aren't just an analogy. Your email-validation regex and their regular expressions aren't merely alike. They're the same formal object, and you now know exactly why.

## Where the ladder goes next

There are two directions you could go next, and both are open to you.

**Go up one rung.** This means pushdown automata and context-free grammars, the parser. This is the second stage of every compiler, and it's the topic I deliberately postponed, in both [Part 2](/blog/what-is-a-language/) and [Part 5](/blog/building-a-regex-engine/). You already know why it needs a stack. Build one yourself and you'll have the front end of a compiler.

**Go sideways, into unresolved territory.** This is the P vs NP question, from [Part 4](/blog/nfa-vs-dfa/): does nondeterminism actually help? At the level of finite automata, Rabin and Scott proved the answer is no, and you've now seen that entire proof built from the ground up. Further up the ladder, this same question has remained unanswered for fifty years. That puts you in an unusual position: you know exactly what a famous unsolved problem is really asking, because you've already seen its smaller, simpler version get fully resolved.

The theory ends here. The building doesn't.

Go up a rung.

---

*A note on how this is written: the ideas and the code are mine. I use AI to help edit the language, since English isn't my first language.*
