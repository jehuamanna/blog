---
title: Why (a+)+b Can Freeze JavaScript But Not grep
description: Why the exact same regex pattern is instant in grep but can freeze JavaScript for minutes. - Study Notes
pubDate: 2026-09-07T11:00:00Z
tags:
  - regex
  - performance
  - security
---

*Part 6 of a series on automata, computation, and why regex never lies to you.*

<div class="series-nav">
<p>Previously in this series</p>
<ol>
<li><a href="/blog/react-state-machine-is-a-dfa/">Your React State Machine Is Secretly a DFA</a></li>
<li><a href="/blog/what-is-a-language/">What Even Is a "Language" in Computer Science?</a></li>
<li><a href="/blog/elevator-state-machine/">Modeling an Elevator With a State Machine</a></li>
<li><a href="/blog/nfa-vs-dfa/">Nondeterminism Doesn't Make Machines More Powerful (And Here's the Proof)</a></li>
<li><a href="/blog/building-a-regex-engine/">Building a Regex Engine From Scratch: Thompson's Construction to DFA</a></li>
</ol>
</div>

[Part 5](/blog/building-a-regex-engine/) ended with a specific claim: the engine reading this sentence right now, your browser's or Node's, is a backtracking engine, and it can cause an exponential blowup in runtime. That's not just a theoretical claim. Here's a way to measure it directly.

Five posts of theory have been building toward one fact you can verify yourself in the next sixty seconds.

The "regex" in JavaScript and the "regex" inside `grep` aren't the same kind of machine. Same syntax. Same pattern string. Same input. But one returns a result in microseconds. The other doesn't return at all. It's not just slower, it *doesn't finish*.

Here's the pattern that proves this.

## The pattern

```javascript
const pattern = /^(a+)+b$/;
```

Read it informally: one or more `a`s, that whole group repeated one or more times, followed by a `b`. This looks almost redundant. `(a+)+` matches exactly the same set of strings as the simpler `a+`, the same language in the sense described in [Part 2](/blog/what-is-a-language/). On input that actually matches, both patterns behave the same way: fast, boring, unremarkable.

The trouble starts on input that does *not* match.

## The benchmark

Paste this directly into your browser's DevTools console or into a Node REPL. It needs no setup and no dependencies.

```javascript
const pattern = /^(a+)+b$/;

for (const n of [20, 25, 30, 35]) {
  const input = 'a'.repeat(n); // no trailing 'b', this will NOT match
  const t0 = performance.now();
  pattern.test(input);
  const t1 = performance.now();
  console.log(`n=${n}: ${(t1 - t0).toFixed(1)}ms`);
}
```

This tests four strings. The longest one is 35 characters long, shorter than this sentence. Here's roughly what you should expect to see:

| `n` | Input | Time (roughly) |
|---|---|---|
| 20 | 20 `a`s | a few milliseconds |
| 25 | 25 `a`s | on the order of 100ms |
| 30 | 30 `a`s | several seconds |
| 35 | 35 `a`s | several **minutes** |

The exact numbers will vary depending on your machine and engine version, so treat these as rough, order-of-magnitude estimates. What matters is the overall shape of the growth, and that shape is severe: each time you add 5 more characters to `n`, the runtime multiplies by roughly 30 times. Add five more characters after that and you're past an hour.

Notice what happens while `n=35` is running: nothing else can happen. `test()` runs synchronously. No `await`, no point where it yields control, no way to interrupt it. The thread is completely blocked.

Now compare this to the same pattern and the same strings run in a DFA-based engine:

```
$ grep -E '^(a+)+b$' <<< "$(printf 'a%.0s' {1..35})"
```

This returns instantly. It's also instant at 35 characters, at 3,500 characters, and at 35,000 characters. Rust's `regex` crate behaves the same way. The runtime stays linear in the length of the input, every time, no matter what string you give it. That's not because `grep` happens to be well-optimized. It's because it uses a fundamentally different algorithm with a different worst case.

## Why this happens

[Part 5](/blog/building-a-regex-engine/) built the NFA. This section explains what happens when you run that NFA the wrong way.

JavaScript's `RegExp`, like Python's `re` and like PCRE, is a **backtracking** engine. It walks the Thompson-construction NFA directly, using depth-first search: it picks a branch, follows it, and if that branch turns out to be a dead end, it undoes that choice and tries the next one.

Against the string `'aaaaaaaaaa'`, the pattern `(a+)+` behaves like two nested loops competing for the same characters. The inner `a+` could take all ten characters. Or nine, leaving one character for a second pass of the outer `+`. Or eight, splitting the rest as 8+2 or as 8+1+1. Every possible way of splitting the string into nonempty chunks counts as a separate branch in the search tree, and the engine has to try each one before it can rule it out.

The number of ways to split a string of length *n* into nonempty ordered pieces is 2^(n-1). This grows exponentially. And here's the especially costly part: because there's no `b` at the end of the string, **every single branch fails**. A successful match can stop as soon as the first success is found. A failed match forces the engine to exhaust the entire search tree before it can give up. So the worst case isn't the string that matches, it's the string that almost matches but doesn't.

Now compare this to the matching loop from [Part 5](/blog/building-a-regex-engine/):

```javascript
function matches(transitions, start, accepting, text) {
  let state = start;
  for (const ch of text) {
    state = transitions.get(`${state},${ch}`);
    if (state === undefined) return false; // dead end, short-circuit immediately
  }
  return accepting.has(state);
}
```

Try to find a branch here that could be retried. There isn't one. There's only a lookup, and that lookup has exactly one possible answer. Subset construction already combined every possible NFA branch into single, deterministic DFA states at the time the automaton was *built*. The entire "consider every way this could go" search was already done once, at compile time, and the result was baked into a lookup table. One lookup per character, for `n` characters total, and then it's done.

A backtracking engine pays that same search cost again for every character of input, every time it runs. A DFA pays that cost once, during construction, and never again.

## So why doesn't every engine just build a DFA?

Because backtracking gives you something valuable in return.

Backreferences (`\1`, meaning "match whatever that group matched, again") and lookaround (`(?=...)`, `(?<=...)`) can't be expressed as regular languages. No finite automaton can implement them, no matter how large you make it. That's not a limitation of current implementations, it's a proven mathematical fact. Depth-first search over the NFA, combined with the ability to undo choices, is what makes these features possible at all.

So general-purpose programming languages made a deliberate tradeoff. JavaScript, Python, Perl, and PHP chose to keep this extra expressiveness and accepted the exponential worst case that comes with it. Tools built for safety and high throughput at scale made the opposite choice, on purpose: `grep`, Rust's `regex` crate, and RE2, which Google built specifically because pathological regex patterns run against user-supplied input were taking down production systems. RE2 simply refuses to compile a pattern that contains a backreference. That restriction is the entire point of RE2.

Neither choice is wrong on its own. But you should know which one is running underneath your code.

## This is a real vulnerability class, and it can affect you

Catastrophic backtracking is a well-known enough problem to have its own name: **ReDoS**, short for regular expression denial of service. There are two common ways this affects a frontend engineer.

**On the server side.** A Node/Express route might validate a request body, a query parameter, or a `User-Agent` header using a regex that contains nested quantifiers, patterns like `(a+)+`, `(a*)*`, `(\s+)+$`, or `([a-zA-Z]+)*`. A single crafted 40-character string, sent in a single request, can pin the event loop at 100% CPU usage, and the process then stops responding to *every* user, not just the attacker. It doesn't take a flood of requests or a botnet, just one request. This exact scenario has taken down real production services in the past.

**On the client side.** The same kind of regex can appear inside an `onChange` or `onBlur` validator. The user types normally. Then the main thread enters the regex matching and never returns. No re-render, no scrolling, no click response, no loading spinner. The tab simply freezes, and the only way out is to close it. Your own users could trigger this completely by accident, for example by pasting a long string into an email field.

It's worth searching your codebase for any quantifier applied to a group that already contains one. Email and URL validators copied from Stack Overflow are the usual source of this problem. The typical fix is to remove the nesting, add a length bound, or move the validation off the main thread. Worth spending an afternoon on.

## Where we are

We've covered states and transitions. Languages defined as sets of strings. A physical machine that opens elevator doors. NFAs and DFAs, proven equal in computing power but very different in cost. A regex engine built from scratch. And now a 35-character string that can bring down a server.

All of this came from a single idea, appearing in six different forms. In the next post we set the different forms aside and look directly at the idea itself.

*Next: [Regex, Automata, and Computation Are the Same Thing](/blog/regex-automata-computation-same-thing/)*

---

*A note on how this is written: the ideas and the code are mine. I use AI to help edit the language, since English isn't my first language.*
