---
title: Modeling an Elevator With a State Machine
description: Extending finite automata from yes/no acceptors into machines that act, using an elevator as the model. - Study Notes
pubDate: 2026-09-07T08:00:00Z
tags:
  - automata
  - state-machines
---

*Part 3 of a series on automata, computation, and why regex never lies to you.*

<div class="series-nav">
<p>Previously in this series</p>
<ol>
<li><a href="/blog/react-state-machine-is-a-dfa/">Your React State Machine Is Secretly a DFA</a></li>
<li><a href="/blog/what-is-a-language/">What Even Is a "Language" in Computer Science?</a></li>
</ol>
</div>

Every machine in this series so far has only had one job: check an input and give a verdict. You feed it a string, and it decides yes or no. `L(M)` is the set of strings it accepts, and a DFA's only output is that single verdict.

That works fine for a regex. It doesn't work for an elevator.

An elevator has states, you can even feel them as a passenger, but "accept" and "reject" mean nothing to a steel box hanging on a cable. What an elevator needs to do is *act*. Start the motor. Stop the motor. Open the doors. Close the doors, but not on someone's arm. The value here isn't in classifying a sequence of inputs. It's in what actually happens in the physical world while the machine runs.

So we need to extend the model. Not replace it, extend it.

## The machine, concretely

Four states make up the entire control layer of an elevator.

- `IDLE`: parked at a floor, motor off, doors shut.
- `MOVING_UP`
- `MOVING_DOWN`
- `DOORS_OPEN`

Five events can come in from the outside world:

- `call`: someone pressed a button in a hallway.
- `select`: someone pressed a floor button inside the cab.
- `arrived_at_target`: the position sensor reports that the car is level with the target floor.
- `door_timer_expired`: the doors have been open long enough.
- `obstruction_detected`: the door edge sensor detected something in the way. A bag. A hand.

Five actions go out to the hardware:

- `move_motor_up`, `move_motor_down`, `stop_motor`, `open_doors`, `close_doors`

Notice these actions aren't return values. Nothing collects them or checks them afterward. They're commands sent directly to physical relays, and once sent, they have real consequences that no later code can undo.

## Two ways to attach output: Moore and Mealy

The DFA in [Part 1](/blog/react-state-machine-is-a-dfa/) had no output at all. There are exactly two places you can add output to a machine like this, and both approaches have names dating back to the 1950s.

A **Moore machine** attaches output to *states*. The output depends only on which state you're currently in, nothing else. `DOORS_OPEN` means the door-hold solenoid is energized, continuously, for as long as the machine stays in that state. You don't need to separately command "keep the doors open," that's simply what being in that state *means*. The same applies to `MOVING_UP`: as long as you're in that state, the up-motor keeps running. In a Moore machine, being in a state is the output.

A **Mealy machine** attaches output to *transitions* instead. The output depends on both the current state *and* the event that just arrived. The moment `arrived_at_target` occurs while the machine is in `MOVING_UP`, it fires `stop_motor`. That isn't tied to a state, it's a single pulse triggered by a specific change. Model this as a Moore output instead and it would be wrong: you'd end up re-issuing `stop_motor` forever, even while the elevator is simply parked.

Real elevator controllers use both Moore and Mealy outputs together, and that's not a sign of sloppy design. Continuous physical conditions, like the motor being energized or the doors being held open, fit the Moore model. Discrete one-time commands, like stop, open, or close, fit the Mealy model. The formal model doesn't require you to pick only one style. It requires you to be precise about which style you're using in each case.

## The transition table

This table is the real artifact here. Not the diagram, not the code. The table itself. Everything else, including the diagram and the code, is just a different way of displaying the same table.

```
CURRENT STATE   EVENT                  NEXT STATE     ACTIONS
─────────────────────────────────────────────────────────────────────────
IDLE            call / select          MOVING_UP      move_motor_up
                (target above)
IDLE            call / select          MOVING_DOWN    move_motor_down
                (target below)
IDLE            call / select          DOORS_OPEN     open_doors
                (target == current)
MOVING_UP       arrived_at_target      DOORS_OPEN     stop_motor, open_doors
MOVING_DOWN     arrived_at_target      DOORS_OPEN     stop_motor, open_doors
DOORS_OPEN      obstruction_detected   DOORS_OPEN     open_doors  (re-fire,
                                                      reset door timer)
DOORS_OPEN      door_timer_expired     IDLE           close_doors
MOVING_UP       call / select          MOVING_UP      (none, queue it)
MOVING_DOWN     call / select          MOVING_DOWN    (none, queue it)
```

Here's the same thing written as executable JavaScript:

```javascript
const elevator = {
  IDLE: {
    dispatch: { next: "MOVING_UP", action: "move_motor_up" },   // guard: above
    open:     { next: "DOORS_OPEN", action: "open_doors" },     // guard: same
  },
  MOVING_UP: {
    arrived_at_target: { next: "DOORS_OPEN", action: ["stop_motor", "open_doors"] },
  },
  DOORS_OPEN: {
    obstruction_detected: { next: "DOORS_OPEN", action: "open_doors" },
    door_timer_expired:   { next: "IDLE", action: "close_doors" },
  },
};
```

Two rows in that table deserve a closer look.

`DOORS_OPEN + obstruction_detected -> DOORS_OPEN` is a self-loop that still triggers an action. It's a pure Mealy-style move: the state itself doesn't change, but the door reverses direction and the timer resets. This single row is the reason elevator doors don't injure people.

`MOVING_UP + call -> MOVING_UP, no action` is actually the more important row, because of what it deliberately doesn't do.

## Scheduling isn't the FSM's job

Suppose a `call` arrives while the car is climbing toward floor 7. Should it stop at floor 4 along the way? Reverse direction? Finish the current trip first and come back later?

The FSM has no opinion on this, and it must have no opinion. Deciding *which floor to serve next* is a separate problem called the elevator scheduling problem. It's a genuinely hard optimization problem, with real algorithms and heuristics (SCAN, LOOK, nearest-car assignment) and real tradeoffs between average wait time and worst-case wait time. Building designers actually pay outside consultants to solve this problem well.

That entire problem lives *outside* the state machine. The scheduler looks at pending calls, applies whatever heuristic it uses, and produces a single output: a target floor. It hands that decision to the FSM as a `dispatch` event. The FSM's job only begins after the decision has already been made, and its only responsibility is this: get to that floor without hurting anyone.

Separating these two problems makes both much easier to manage. The scheduler can be a complicated set of tuned heuristics; it can be swapped out, A/B tested, even replaced with a neural network by a junior engineer. Meanwhile the FSM stays simple, small, and provably correct: just nine rows you can check by hand.

You've probably drawn this same line before in your own work, or wished you had. Business logic and routing logic decide *what* should happen next. A UI state machine's guards decide *whether a given transition is allowed to happen at all*. Mix these two responsibilities together and nobody can tell whether a bug is caused by a bad decision or by an unsafe execution of an otherwise good one.

## Why the FSM shape earns its place here

The rule "never run the motor while the doors are open" is the core safety invariant. There are two different ways to enforce it.

The first way is to check for it explicitly: `if (doorsOpen && motorRunning) emergencyStop()`. Here the invariant is just a claim about how your code behaves at runtime, protected only by a conditional statement, one that someone might accidentally break while refactoring code at 2am.

The second way is to make the illegal configuration *impossible to represent in the first place*. Motor commands only appear on the transitions into `MOVING_UP` and `MOVING_DOWN`. The door-hold output only exists inside `DOORS_OPEN`. No single state has both motor-running and doors-open behavior, because no such state exists in `Q`, the set of states. You can't accidentally trigger a condition that has no representation anywhere in the system.

This is the same "illegal states unrepresentable" idea from [Part 1](/blog/react-state-machine-is-a-dfa/), just applied somewhere with much higher real-world stakes. In a React component, an impossible state combination might just cause a spinner that never stops. Here, the same kind of mistake could cause a real elevator to drop.

## One transition per event, until it isn't

Every machine we've looked at so far, the reducer, the regex, and the elevator, has had exactly one transition defined for each (state, event) pair. That's what the word *deterministic* means in "DFA," and it's also why the table above is a simple table rather than something more like a maze with multiple paths.

But nothing in principle forces this to be true. What if a machine could offer *three* different transitions for the same event and simply take all three at once? Or offer none at all? At first this sounds like it would break the rules. It sounds like it should produce a strictly more powerful kind of machine.

It turns out it doesn't, and the proof of that fact is one of the most useful results in this entire field.

*Next: [Nondeterminism Doesn't Make Machines More Powerful (And Here's the Proof)](/blog/nfa-vs-dfa/)*

---

*A note on how this is written: the ideas and the code are mine. I use AI to help edit the language, since English isn't my first language.*
