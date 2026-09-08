---
title: What Even Is a "Language" in Computer Science?
description: A precise definition of "language" in computer science, and why JavaScript isn't a regular one. - Study Notes
pubDate: 2026-09-07T07:00:00Z
tags:
  - automata
  - formal-languages
---

*Part 2 of a series on automata, computation, and why regex never lies to you.*

<div class="series-nav">
<p>Previously in this series</p>
<ol>
<li><a href="/blog/react-state-machine-is-a-dfa/">Your React State Machine Is Secretly a DFA</a></li>
</ol>
</div>

[Part 1](/blog/react-state-machine-is-a-dfa/) ended with a claim aimed directly at your intuition: a `useReducer` call and a regex are the same kind of machine, and the whole job of that machine is deciding **membership** in something called a *language*. That word carried a lot of meaning without ever being explained. Time to explain it properly.

Here's a word that has been quietly confusing your intuition for years: **language**.

When a computer scientist says "language," they don't mean JavaScript, and they don't mean English. They mean something much simpler, so simple it barely seems to deserve the word. Once you understand the real definition, the rest of this series gets much easier to follow. It explains why regex can't match nested parentheses, why compilers work in stages, and why `(a+)+$` is actually a security bug. All of it comes from the definition below.

Getting there takes three steps.

## Step one: an alphabet

An **alphabet** Σ is a finite set of symbols. That's the entire requirement.

```
Σ = { 0, 1 }
Σ = { a, b, c }
Σ = { IDENT, EQUALS, NUMBER, SEMICOLON }
```

Symbols are treated as atoms; you never look inside them to see what they're made of. The only rule is that the set has to be finite.

## Step two: Σ\*, every possible string

**Σ\*** is the set of every finite string you can build using symbols from Σ, including the **empty string ε**, a string of zero symbols.

For Σ = {0, 1}:

```
Σ* = { ε, 0, 1, 00, 01, 10, 11, 000, 001, ... }
```

Σ itself is finite, but Σ\* is infinite. With just two symbols, you get every binary string that could ever exist. Every file, every integer, every JPEG image is, at some level, one of the strings inside this set. Σ\* is the entire universe of possible strings, and nothing is excluded from it, because at this stage nothing has been judged yet.

## Step three: a language is a subset

A **language** L over Σ is any subset of Σ\*.

```
L ⊆ Σ*
```

That's the whole definition. No grammar involved, no meaning attached, no additional rules. You simply pick some strings out of Σ\*, and that selection is, by definition, a language.

This means all of the following count as languages over Σ = {0, 1}:

| Language | Contains | Notes |
|---|---|---|
| Strings with an even number of `0`s | ε, `1`, `00`, `1001`, … | Infinite, and describable |
| Strings of only `0`s | ε, `0`, `00`, `000`, … | Also infinite |
| `{ ε }` | just the empty string | One element. Still a language |
| `∅` | nothing at all | Zero elements. **Also a language** |
| `Σ*` | everything | The whole universe. Yes, a language |

A language that contains nothing at all is still a valid language. A language that contains only the empty string is a *different* valid language. Notice how different this is from how you normally use the word "language" in everyday speech. Here, "language" doesn't mean a communication system. It simply means a **set of strings**, chosen using whatever rule you like.

## Where the machine comes in

[Part 1](/blog/react-state-machine-is-a-dfa/) ended by introducing the fifth part of a DFA, the set of accepting states F, which seemed useless at the time. Here's why it matters.

Take a machine M and run it on a string. Feed the symbols in one at a time. Once the input is finished, the machine is sitting in some particular state. If that state belongs to F, M **accepts** the string. Otherwise it rejects. Now collect every string the machine accepts:

```
L(M) = { w ∈ Σ* : M accepts w }
```

That collected set is called **the language of M**. Every machine defines exactly one language.

This changes how you should think about what a finite automaton actually is. It isn't a calculator. It doesn't return numeric values, perform arithmetic, or produce output. It answers exactly one question: *is this string part of my set?* A DFA is a **membership test**, a classifier that answers yes or no, one string at a time.

That gives us the key term the rest of this series depends on: a language is called **regular** if some finite automaton recognizes exactly that language, not approximately but exactly. The automaton accepts every string that belongs to L and rejects every string that doesn't.

Some languages are regular. Most aren't. That gap is where things get interesting.

## Is JavaScript a language in this sense?

Yes, genuinely, in the formal sense described above.

Let Σ be the set of characters your source files are allowed to contain. Then:

```
L(JavaScript) = { w ∈ Σ* : w is a syntactically valid JS program }
```

This is a real formal language, a genuine subset of Σ\*. The string `const x = 1;` belongs to this set; `const = ;; x 1` doesn't. Every syntax error you've ever seen in your life was really just a membership test returning false.

However, `L(JavaScript)` is **not regular**. No finite automaton can recognize it. The reason is nesting: to check whether braces are properly balanced, the machine has to track how deeply nested it currently is, and that depth has no upper limit. A finite automaton, by definition, has only a finite number of states, so it can't count up to an arbitrarily large number. To handle this you need a machine with a stack, called a **pushdown automaton**. A later post in this series will explain, with proof, exactly why this limitation is unavoidable. For now, just remember the fact.

## This is why compilers split the job

This is exactly why your compiler toolchain works in separate stages.

| Stage | Job | Machine | Language class |
|---|---|---|---|
| Lexer / tokenizer | Group characters into tokens: `IDENT`, `EQUALS`, `NUMBER` | Finite automaton | Regular |
| Parser | Check token structure, balanced braces, nesting | Pushdown automaton | Context-free |

Recognizing an identifier, something matching `[a-zA-Z_][a-zA-Z0-9_]*`, is an ordinary job for a finite state machine. It runs in linear time and needs no memory beyond its current state. Understanding the structure of an entire program is a different kind of problem, though. Babel doesn't separate tokenizing from parsing as a style choice. It separates them because the two tasks genuinely require *different classes of machine*.

That has a very practical consequence. When you find you can't match balanced HTML tags or nested parentheses using a regex, you haven't failed at using regex correctly. Regex is simply a notation for describing finite automata, finite automata can only recognize regular languages, and balanced nesting isn't a regular language. That's a proven mathematical fact, not a gap in your skill.

That's enough abstract theory for now. Next, we'll look at a machine you've personally used this week, one where transitions don't just accept or reject input but actually move motors and open doors.

*Next: [Modeling an Elevator With a State Machine](/blog/elevator-state-machine/)*

---

*A note on how this is written: the ideas and the code are mine. I use AI to help edit the language, since English isn't my first language.*
