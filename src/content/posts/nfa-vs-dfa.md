---
title: Nondeterminism Doesn't Make Machines More Powerful (And Here's the Proof)
description: Why nondeterministic finite automata are no more powerful than DFAs, proved by subset construction. - Study Notes
pubDate: 2026-09-07T09:00:00Z
tags:
  - automata
  - nfa
  - dfa
---

*Part 4 of a series on automata, computation, and why regex never lies to you.*

<div class="series-nav">
<p>Previously in this series</p>
<ol>
<li><a href="/blog/react-state-machine-is-a-dfa/">Your React State Machine Is Secretly a DFA</a></li>
<li><a href="/blog/what-is-a-language/">What Even Is a "Language" in Computer Science?</a></li>
<li><a href="/blog/elevator-state-machine/">Modeling an Elevator With a State Machine</a></li>
</ol>
</div>

Every machine in this series has followed one rule without exception: given a state and an input symbol, there's **exactly one** next state. This is δ : Q × Σ → Q from [Part 1](/blog/react-state-machine-is-a-dfa/). One arrow out, per state, per symbol. No ambiguity, no choice, no gaps.

Now remove that rule.

## The NFA

A **nondeterministic finite automaton** keeps all five parts of a DFA, but weakens one of them. The transition function no longer returns a single state. It returns a *set* of states:

```
δ : Q × Σ → P(Q)
```

So for any (state, symbol) pair, you can now have **many** transitions, **one** transition, or **zero** transitions. A dead end is allowed now. There's also a new type of move: the **ε-transition**, an arrow that consumes *no input*. The machine can move along it for free. (This post won't need ε-transitions, but [Part 5](/blog/building-a-regex-engine/)'s construction is built almost entirely out of them.)

How does a machine with multiple options actually run? There are two equivalent ways to think about it.

**Parallel exploration.** The NFA isn't in a single state, it's in a *set* of states at once. When it reads a symbol, every state in the set fires all of its transitions, and the union of the results becomes the new set. In this view the machine is genuinely in several places at the same time. If any state in the final set is an accepting state once the input ends, the machine accepts.

**Guessing.** An equivalent way to describe this: imagine the machine has an oracle that always guesses correctly. At every fork it guesses the right branch, and it only needs *one* accepting path to exist somewhere. Wrong guesses cost nothing. The machine rejects only when no path could have worked at all.

The guessing framing is where the extra power seems to come from. Let's look closely at what that power actually gives us.

## The example: "the third-from-last symbol is a 1"

Σ = {0, 1}. We want to accept exactly the strings whose third-from-last symbol is `1`. So `100`, `1101`, and `0000111` should be accepted. `011`, `1`, and `000` should be rejected.

As an NFA this is very simple to build. It only needs four states:

```
        ,--- 0, 1 ---.
        |            |
        v            |
  ---> (q0) ---------'
         |
         | 1     <-- "I'm guessing THIS one is 3rd-from-last"
         v
       (q1) --0,1--> (q2) --0,1--> [[q3]]
```

| State | on `0` | on `1`       |
|-------|--------|--------------|
| q0    | {q0}   | **{q0, q1}** |
| q1    | {q2}   | {q2}         |
| q2    | {q3}   | {q3}         |
| q3    | **{}** | **{}**       |

You can read this table as a strategy. `q0` loops on any input, simply waiting. At some point it sees a `1` and *guesses* that this particular `1` is the one that's third from the end, moving into `q1`. From there it must read exactly two more symbols, of any value, which lands it in the accepting state `q3`. State `q3` has no outgoing transitions, so if more input keeps arriving after that, this particular path dies.

Both of the weakened rules get used here: `q0` on input `1` has two possible targets, and `q3` has no targets at all. Notice the machine never actually *checks* anything directly. It commits to a guess early, and the structure of the machine verifies afterward whether that guess was correct.

## The same language, built deterministically

Now let's build the same language without any guessing. A DFA gets exactly one arrow per symbol, with no chance to go back and redo a choice. That means at every position in the input, it has to already know everything it could possibly need to know later. It can't ask, after the fact, "was the symbol three positions back a `1`?" It has to have *remembered* the answer already.

Remembered what, exactly? The last three symbols it has seen. All three, at all times, updated every time it reads a new symbol. That's a sliding window, and there are 2³ = 8 possible values for it. So the DFA needs eight states. We can name each state after the window it represents, with the oldest bit written first:

| State | on `0` | on `1` | Accepting? |
|-------|--------|--------|------------|
| `000` | `000`  | `001`  | no  |
| `001` | `010`  | `011`  | no  |
| `010` | `100`  | `101`  | no  |
| `011` | `110`  | `111`  | no  |
| `100` | `000`  | `001`  | **yes** |
| `101` | `010`  | `011`  | **yes** |
| `110` | `100`  | `101`  | **yes** |
| `111` | `110`  | `111`  | **yes** |

The machine starts in state `000`. Every transition does the same simple thing: drop the oldest bit, shift the remaining bits left, and append the symbol just read. A state is accepting whenever its oldest bit is `1`, because if the input stopped right now, that bit would sit exactly three positions from the end.

The number eight isn't something you could reduce further with a smarter design, either. Feed in zero, one, or two more symbols after being in a given state, and you're effectively asking about the first, second, or third bit of the window. Since any two distinct windows differ in at least one bit position, there's always some suffix of input that tells them apart. That means all eight states are genuinely distinguishable from each other. Eight is the minimum possible number of states.

This is the real payoff, and the pattern generalizes: "the *k*-th symbol from the end is a 1" needs only *k+1* states as an NFA, but needs 2^k states as a DFA. Nondeterminism gave us the ability to **guess instead of remember**.

## Rabin-Scott: it's only a convenience

That convenience, it turns out, is *all* nondeterminism actually buys you.

**Theorem (Rabin and Scott, 1959).** For every NFA, there exists a DFA that recognizes exactly the same language.

Rabin and Scott proved the two models are equivalent in recognizing power, and they later received the Turing Award for this and related work. Nondeterminism adds **zero** extra computational power to finite automata. Not "a small amount." Zero. A machine that can see the future and always guesses correctly can't recognize a single language that a plain, step-by-step table-lookup machine can't also recognize.

The proof is constructive, and you've already seen the core idea. It's the "parallel exploration" view, taken literally and turned into an actual construction.

## The subset construction

If an NFA is really just tracking a *set* of states at every step, we can build a DFA where each state directly represents one such set. This gives us one DFA state for every possible subset of NFA states. We start with the set containing the NFA's start state, plus every state reachable from it using only ε-transitions. To transition on a symbol `a`, we take the union of everywhere reachable on `a` from every state currently in our set. That union is itself just another subset, so it becomes a DFA state too. A DFA state is accepting if *any* NFA state inside its corresponding set is an accepting state. Since there are no choices left anywhere in this construction, the result is deterministic by definition.

An NFA with *n* states has 2^n possible subsets, so in the worst case the resulting DFA can have up to 2^n states. That potential increase in size is the real cost of removing nondeterminism. It doesn't make the conversion impossible, only potentially much larger. In practice the result is usually much smaller than this worst case, because most subsets are never actually reachable. But sometimes the blowup really is as bad as the theory predicts. (In our example, the 4-state NFA became an 8-state DFA. That's not a coincidence, it matches the 2^n pattern exactly, since here n was effectively related to the window size.)

## The version of this question nobody has answered

Step back, and this whole discussion is really one example of a much bigger question: **does the ability to guess make a machine fundamentally more capable?**

For finite automata, this question was settled back in 1959. The answer is no.

Now ask the same question one level up, about *time* rather than memory. Is every problem that a nondeterministic machine can solve in polynomial time also solvable by a deterministic machine in polynomial time? That question is known as **P vs. NP**. It's been open for more than fifty years, has a million-dollar Clay Millennium Prize attached to it, and is arguably the most famous unsolved problem in computer science.

The shape of the question is identical, and so is the underlying intuition. One version was resolved with a proof that fits on a single page. The other has remained unresolved after five decades of serious effort. Nothing in this series will resolve P vs. NP, but the next time someone tells you "P vs. NP is about whether guessing helps," you'll know exactly what that means, because you've already seen the simpler version of the question proved.

## Next

The Rabin-Scott theorem isn't just a piece of theory. It's the actual compilation pipeline inside every fast regex engine. First, the engine parses the pattern into an NFA, which is easy, because nondeterminism makes patterns easy to express. Then it applies the subset construction to turn that NFA into a DFA, which is fast, because determinism turns matching into a simple table lookup. These are two theorems working together, and together they're what powers `grep`.

In the next post, we build this ourselves. Real JavaScript, a regex string as input, a matcher as output.

*Next: [Building a Regex Engine From Scratch: Thompson's Construction to DFA](/blog/building-a-regex-engine/)*

---

*A note on how this is written: the ideas and the code are mine. I use AI to help edit the language, since English isn't my first language.*
