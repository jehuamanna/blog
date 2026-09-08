---
title: Your React State Machine Is Secretly a DFA
description: Why a useState/useReducer component is, formally, a deterministic finite automaton. - Study Notes
pubDate: 2026-09-07T06:00:00Z
tags:
  - automata
  - react
  - state-machines
---

*Part 1 of a series on automata, computation, and why regex never lies to you.*

<div class="series-nav">
<p>In this series</p>
<ol>
<li><a href="/blog/react-state-machine-is-a-dfa/">Your React State Machine Is Secretly a DFA</a> (you are here)</li>
<li><a href="/blog/what-is-a-language/">What Even Is a "Language" in Computer Science?</a></li>
<li><a href="/blog/elevator-state-machine/">Modeling an Elevator With a State Machine</a></li>
<li><a href="/blog/nfa-vs-dfa/">Nondeterminism Doesn't Make Machines More Powerful (And Here's the Proof)</a></li>
<li><a href="/blog/building-a-regex-engine/">Building a Regex Engine From Scratch: Thompson's Construction to DFA</a></li>
<li><a href="/blog/catastrophic-backtracking/">Why (a+)+b Can Freeze JavaScript But Not grep</a></li>
<li><a href="/blog/regex-automata-computation-same-thing/">Regex, Automata, and Computation Are the Same Thing</a></li>
</ol>
</div>

You have probably written a component like this before. Maybe even this week.

```jsx
function UserCard({ id }) {
  const [status, setStatus] = useState("idle");
  const [user, setUser] = useState(null);

  useEffect(() => {
    setStatus("loading");
    fetchUser(id)
      .then((u) => { setUser(u); setStatus("success"); })
      .catch(() => setStatus("error"));
  }, [id]);

  if (status === "loading") return <Spinner />;
  if (status === "error") return <Retry onClick={() => setStatus("idle")} />;
  if (status === "success") return <Profile user={user} />;
  return null;
}
```

This component has four states: `idle`, `loading`, `success`, and `error`. Even without reading the code carefully, you already know some rules about it. `success` doesn't go back to `loading` on its own; something else has to trigger that. `error` only changes when the user clicks retry. And there's no way to reach `success` without first passing through `loading`.

You know these rules because you've already drawn this as a diagram, maybe on a whiteboard, maybe in a Figma comment, maybe only in your head. Four circles, arrows connecting them, one arrow pointing into `idle` from outside with no state before it.

That diagram has a formal name: a **state transition diagram**. And the thing it describes has a formal name too: a **finite state machine (FSM)**. This isn't just a way of thinking about UI code. It's a precise mathematical object with real theorems proven about it, and it's one of the first topics taught in university compiler courses. You've been building these machines by hand for years, using intuition, without knowing their real name.

## The definition (it's shorter than you think)

A deterministic finite automaton (DFA) is made of five parts:

1. **A finite set of states**, called *Q*. Here: `{ idle, loading, success, error }`.
2. **A finite alphabet of inputs**, called *Σ*. These are the events that can happen: `{ FETCH, RESOLVE, REJECT, RETRY }`.
3. **A transition function**, written *δ : Q × Σ → Q*. Give it a state and an input event, and it always gives back exactly one new state, never zero, never two. That's what "deterministic" means: certain, with no randomness and no choice involved.
4. **A start state**, called *q₀*. Here: `idle`.
5. **A set of accepting states**, written *F ⊆ Q*. Optional, and it might seem useless right now. Remember it anyway. This idea is the whole reason regex works, and we'll come back to it in [Part 2](/blog/what-is-a-language/).

That's the entire definition. Five parts, and you can already see four of them just by looking at your own component.

The interesting part is *δ*, the transition function, because you've already written it yourself. Not something similar to it. The exact same thing.

## Your reducer is δ

```jsx
const transition = (state, action) => {
  switch (state) {
    case "idle":
      return action.type === "FETCH" ? "loading" : state;
    case "loading":
      if (action.type === "RESOLVE") return "success";
      if (action.type === "REJECT") return "error";
      return state;
    case "error":
      return action.type === "RETRY" ? "loading" : state;
    case "success":
      return state;
    default:
      return state;
  }
};
```

Look at the shape of this function: `(state, action) => state`. Now compare it to *Q × Σ → Q*. Same shape. The signature of `useReducer` matches the signature of a transition function exactly, not just approximately. When you write a reducer, you're really writing δ, just expressed with `if` and `switch` statements instead of a table. A library like XState makes this more visible: instead of hiding the table inside control flow, it lets you write the table directly.

Now notice what's *missing* from the code. There's no case for `loading` receiving `RETRY`. No case for `success` receiving `REJECT`. These aren't mistakes, and they aren't gaps you forgot to fill in. They're the entire point. If a state doesn't handle a particular event, that event **cannot** happen in that state. The machine enforces this through its structure, so you don't need to hope a code reviewer notices the problem. It simply can't exist.

Compare that to the version most people write without thinking too much about it:

```jsx
const [isLoading, setIsLoading] = useState(false);
const [hasError, setHasError] = useState(false);
const [isDone, setIsDone] = useState(false);
```

Three boolean variables create eight possible combinations. Only four of these combinations actually make sense. The other four are bugs waiting for the right timing (a race condition) to trigger them. `isLoading && hasError && isDone` can technically exist in your code, but it has no real meaning. Eventually, someone ships this bug into production.

The state machine has exactly four states because there really are only four valid states. The illegal combinations aren't blocked by an extra check. They simply don't exist as an option in the first place.

## Why the name buys you something

**You can design the system before writing any code.** A transition diagram is a specification that a designer, a backend engineer, and a QA engineer can all read and discuss together. Almost nobody can have a useful discussion by just looking at a `useEffect` hook. Draw the state machine first, get everyone to agree on it, and writing the reducer becomes easy, since at that point you're really just copying the diagram into code.

**Illegal states become impossible to represent.** That's different from saying they're "unlikely" or "defended against with a check." A type with exactly four defined variants simply has no way to hold a fifth value.

**The machine can be checked completely.** A finite number of states times a finite number of events gives you a finite table, and a program can walk through that whole table automatically. Is any state impossible to reach from `idle`? Is there a state that accepts every event but never actually changes? Is there a state where `REJECT` does nothing, leaving the user staring at a spinner forever? These are questions with real, provable answers, not guesses. You could write a program to check all of this in a single afternoon, for the same reason a parser generator can tell you your grammar has an error before you ever run your code.

**And this pattern shows up everywhere.** Router transitions. A video player's states: `paused`, `playing`, `buffering`, `ended`. A multi-step form. Traffic lights. Drag-and-drop interactions. Once you learn to recognize this shape, you stop accidentally building weaker, less structured versions of it.

## The really surprising part

Everything so far could be treated as a useful *analogy* for thinking about UI code. Here's where it stops being an analogy.

The regex `/^\d{3}-\d{4}$/` **is** a finite state machine. Not resembles one. Is one. When the V8 engine compiles this pattern, it builds a set of states and a transition function. Matching a string means feeding in characters one at a time and checking which state you land in after each one. The tokenizer inside Babel, which decides whether a `<` character starts a JSX element, is also a finite state machine. So is the TCP handshake, with states like `SYN_SENT`, `SYN_RECEIVED`, and `ESTABLISHED`, written into an official networking specification by engineers who understood exactly what they were building. Even a physical elevator with three floors and one door is a finite state machine, just built out of metal and motors instead of code.

Your `status` variable, a regex, a lexer, a network protocol, and a physical elevator are all the *same mathematical object*. They use different alphabets, different sets of possible inputs, but they share the exact same five-part definition. Prove something true about one of them, and you've proven it true for all of them.

That leads to the main question the rest of this series is built around: if a regex and a reducer are the same kind of machine, what is a regex actually *computing*? It's deciding whether a string belongs to a **language**. Here, "language" doesn't mean English, or JavaScript, or anything you'd normally call a language in everyday speech. It means something smaller and more precise. Understanding that exact meaning is the key that unlocks the rest of this series: why some patterns can never be matched by regex no matter how clever you get, why a pattern like `(a+)+$` can crash a server, and why regex, unlike almost every other tool you use, truly can't lie to you.

*Next: [What Even Is a "Language" in Computer Science?](/blog/what-is-a-language/)*

---

*A note on how this is written: the ideas and the code are mine. I use AI to help edit the language, since English isn't my first language.*
