---
title: "Building a Regex Engine From Scratch: Thompson's Construction to DFA"
description: Building a working regex engine by hand, Thompson's construction, subset construction, and a linear-time matcher. - Study Notes
pubDate: 2026-09-07T10:00:00Z
tags:
  - automata
  - regex
  - algorithms
---

*Part 5 of a series on automata, computation, and why regex never lies to you.*

<div class="series-nav">
<p>Previously in this series</p>
<ol>
<li><a href="/blog/react-state-machine-is-a-dfa/">Your React State Machine Is Secretly a DFA</a></li>
<li><a href="/blog/what-is-a-language/">What Even Is a "Language" in Computer Science?</a></li>
<li><a href="/blog/elevator-state-machine/">Modeling an Elevator With a State Machine</a></li>
<li><a href="/blog/nfa-vs-dfa/">Nondeterminism Doesn't Make Machines More Powerful (And Here's the Proof)</a></li>
</ol>
</div>

[Part 4](/blog/nfa-vs-dfa/) ended with a promise that looked like a side note: subset construction isn't just a way to *prove* that an NFA and a DFA are equivalent. It's the actual algorithm used inside every fast regex engine. Time to build it.

Four earlier posts covered the theory. Now let's put that theory to use.

States, transitions, languages, NFAs, and the subset construction weren't just material for an exam. Together they form a build pipeline. Put in a pattern string, and you get out a matcher with a provably linear running time, through three stages. Each stage is a classical, named algorithm from the 1950s and 1960s, and each one is still running right now inside `grep`.

```
"a(b|c)*d"  ──Thompson──►  NFA  ──subset construction──►  DFA  ──run──►  true/false
```

That diagram is the whole post, in short. Now let's build it.

## Stage 1: regex to NFA (Thompson's construction, 1968)

Ken Thompson published this insight while building the editor that later became `ed`, and eventually `grep`. His key idea is that you never need to reason about a whole regex at once. Each operator maps to a small **fragment**: a tiny NFA with exactly one start state and one accept state. These fragments can be combined with each other. That's the whole idea.

The rule that each fragment has exactly one start state and one accept state is what makes this combining possible, and **ε-transitions are what give you this rule**. Without a "free move" edge like this, joining two machines together would mean rewriting their internal structure: merging sets of states, rewiring transitions, checking for conflicts. With an ε-transition you just draw one new arrow instead. Every fragment looks the same from the outside, so you can drop a fragment into any slot without knowing anything about what's inside it.

| Regex | Fragment |
|---|---|
| `a` | Two states, one edge labeled `a`. Start → accept. |
| `AB` | Build `A`, build `B`, add `A.accept ─ε→ B.start`. Start is `A.start`, accept is `B.accept`. |
| `A\|B` | New start with `ε` into both `A.start` and `B.start`; new accept, with `ε` from both `A.accept` and `B.accept`. |
| `A*` | New start/accept pair. `start ─ε→ A.start` (enter), `start ─ε→ accept` (skip entirely), `A.accept ─ε→ A.start` (loop), `A.accept ─ε→ accept` (exit). |

Each rule adds **at most two states**. So a regex of length *m* compiles into an NFA with O(*m*) states, using a single recursive pass over the parse tree, one function call per operator, no backtracking and no fixpoint iteration needed. Compilation time is linear in the length of the pattern, no exceptions.

Here's `a(b|c)*d`, built from the bottom up:

```
                    ┌───────────── ε (skip) ─────────────┐
                    │                                    ▼
(0) ─a→ (1) ─ε→ (8) ─ε→ (6) ─ε→ (2) ─b→ (3) ─ε→ (7) ─ε→ (9) ─ε→ (10) ─d→ ((11))
                         │  ▲                     │
                         │  └───── ε (loop) ──────┘
                         └──ε→ (4) ─c→ (5) ─ε→ (7)
```

States 2 through 5 are the symbol fragments for `b` and `c`. States 6 and 7 are the new start and accept states created for the union (`|`). States 8 and 9 are the ones created for the star (`*`). The `ε` transitions out of state 1 and into state 10 represent the two concatenation steps. In total, twelve states, all produced mechanically, no clever tricks needed.

As data:

```javascript
const nfa = {
  start: 0,
  accept: 11,
  alphabet: new Set(["a", "b", "c", "d"]),
  edges: new Map([
    ["0,a", new Set([1])],
    ["2,b", new Set([3])],
    ["4,c", new Set([5])],
    ["10,d", new Set([11])],
  ]),
  epsilonEdges: {
    1: [8],
    3: [7],
    5: [7],
    6: [2, 4],
    7: [6, 9],
    8: [6, 9],
    9: [10],
  },
};
```

## Stage 2: NFA to DFA (subset construction)

This is the same algorithm from [Part 4](/blog/nfa-vs-dfa/). Now we turn it into code.

A DFA state here is a **set** of NFA states, all the places the NFA could be at the same time. First we compute everything reachable using only ε-transitions:

```javascript
function epsilonClosure(nfa, states) {
  const closure = new Set(states);
  const stack = [...states];
  while (stack.length) {
    for (const next of nfa.epsilonEdges[stack.pop()] ?? []) {
      if (!closure.has(next)) {
        closure.add(next);
        stack.push(next);
      }
    }
  }
  return closure;
}

function move(nfa, states, symbol) {
  const out = new Set();
  for (const s of states) {
    for (const next of nfa.edges.get(`${s},${symbol}`) ?? []) out.add(next);
  }
  return out;
}
```

There's one small complication in JavaScript: a `Set` can't be used as a `Map` key by its contents. Two sets with exactly the same contents would still count as different keys. So we need a canonical form instead. Sort the ids, join them with commas, and the resulting string *becomes* the identity of the set:

```javascript
const key = (states) => [...states].sort((a, b) => a - b).join(",");
```

Now for the worklist algorithm. We start from the closure of the start state and keep discovering new subsets until no new ones appear:

```javascript
function subsetConstruction(nfa) {
  const startSet = epsilonClosure(nfa, [nfa.start]);
  const start = key(startSet);

  const transitions = new Map(); // dfaKey -> Map<symbol, dfaKey>
  const accepting = new Set();
  const seen = new Map([[start, startSet]]);
  const worklist = [start];

  while (worklist.length) {
    const stateKey = worklist.pop();
    const current = seen.get(stateKey);

    // Accepting if ANY NFA state in the set accepts.
    if (current.has(nfa.accept)) accepting.add(stateKey);

    const row = new Map();
    for (const symbol of nfa.alphabet) {
      const nextSet = epsilonClosure(nfa, move(nfa, current, symbol));
      if (nextSet.size === 0) continue; // dead end, no entry
      const nextKey = key(nextSet);
      if (!seen.has(nextKey)) {
        seen.set(nextKey, nextSet);
        worklist.push(nextKey);
      }
      row.set(symbol, nextKey);
    }
    transitions.set(stateKey, row);
  }

  return { start, accepting, transitions };
}
```

At this point the ε-transitions are gone entirely. Every call to `epsilonClosure` has already absorbed them into the state sets. What's left is a plain lookup table.

The problem [Part 4](/blog/nfa-vs-dfa/) warned about is real: *n* NFA states can produce up to 2ⁿ subsets, and some difficult patterns actually reach this limit. That's why many production engines never build the full table at all. Instead they run this same loop *lazily*, keeping only the current subset in memory and computing the next one as each input character arrives. RE2 goes even further: it caches DFA states as needed and drops them again when memory gets tight. The underlying math is the same, only the tradeoff between memory use and speed differs.

## Stage 3: running it

```javascript
function matches(transitions, start, accepting, text) {
  let state = start;
  for (const ch of text) {
    const next = transitions.get(state)?.get(ch);
    if (next === undefined) return false; // dead end: no path can recover
    state = next;
  }
  return accepting.has(state);
}
```

Look closely at this loop and count the amount of work being done: one hash lookup per character. No recursion, no stack. No character is ever read twice. If the loop reaches a dead end it returns immediately, because in a DFA there's no alternative branch left to try. That's exactly what determinism *means*.

So the result is **O(n) in the size of the input, no matter what the pattern is**. Not just "fast in practice," and not an average-case or amortized bound either. It's a worst-case guarantee that the structure of the machine makes impossible to break. Any complexity in the pattern gets paid for once, at compile time, and never again after that.

## Two kinds of regex engine

Here's something most people who write regex every day never learn: the term "regex engine" actually refers to two fundamentally different kinds of machines.

**DFA-based engines**, like `grep`, RE2, and Rust's `regex` library. This is exactly what you just built, or a related lazy NFA-simulation approach. These engines are linear, predictable, and safe even with hostile input. They **cannot support backreferences**, though. The pattern `(\w+) \1` isn't something an engineer simply forgot to implement. The concept of "the same substring appeared twice" isn't a regular language at all, so no finite automaton can recognize it, and there's no place in this pipeline where it could fit.

**Backtracking engines**, like PCRE, Python's `re`, and **JavaScript's built-in `RegExp`**. These skip stages 2 and 3 completely. Instead they walk through the NFA directly, using recursion: try one branch, and if it fails, undo that choice and try the next branch instead. This approach makes backreferences, lookahead, and lookbehind possible, features that are genuinely more powerful than what regular languages allow. The cost shows up at runtime: certain patterns can cause the search tree to grow exponentially.

So the word "regex" is slightly misleading. **True** regular expressions, the ones provably equivalent to a finite automaton and therefore covered by the O(n) guarantee, are only the subset of patterns without backreferences. Add something like `\1` and you've silently stepped outside every theorem discussed in this series.

And the engine running inside the browser tab you're reading this in right now is a backtracking engine.

This exponential blowup isn't just a thought experiment. In the next post we'll actually measure it, directly in your own JavaScript engine, with a stopwatch.

*Next: [Why (a+)+b Can Freeze JavaScript But Not grep](/blog/catastrophic-backtracking/)*

---

*A note on how this is written: the ideas and the code are mine. I use AI to help edit the language, since English isn't my first language.*
