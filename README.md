# Circom-Security

## Table of Contents
   1. [Dangling Signal](#1-dangling-signal)
   2. [Under Constraint Circuit](#2-under-constraint-circuit)

## 1. Dangling Signal 
### Definition:

Unused or dangling signals are variables that are declared but never actually used in any computation or constraint. They can indicate mistakes, unnecessary complexity, or potential logical errors in the circuit.

### What to look for:

1- Signals declared but never assigned a value.

2- Signals assigned a value but never used in any other constraint or output.

3- Signals that may have been intended for a computation but were forgotten.

### Example:

```circom
// This circuit is created by me for explanation

signal input balance;
signal input amount;
signal output ok;

signal safeAmount;
safeAmount <== amount;

// Enforce range
component rc = Num2Bits(32);
rc.in <== safeAmount;


// Logic uses amount directly
component Lq = LessEq(32);
Lq.in[0] <== amount;
Lq.in[1] <== balance;

ok <== Lq.out;
```
In this circuit, the intention is to verify that Alice’s transfer `amount` is less than or equal to her actual `balance`, while also ensuring that the transfer amount is properly range-bounded.

To do this, the circuit introduces an intermediate signal `safeAmount`, assigns it the value of `amount`, and enforces a `Num2Bits(32)` range check on `safeAmount`.

However, when performing the comparison, the circuit mistakenly uses the original `amount` signal instead of the range-checked `safeAmount`.

As a result, the range check is not applied to the value that actually participates in the comparison.

### Why this is dangerous:

`safeAmount` is correctly range-checked

The comparison logic relies on `amount`, not the range-checked signal

Therefore, the verifier does not enforce that the compared value is within 32 bits

## 2. Under Constraint Circuit
### Definition:

An under-constrained circuit is when one or more signals (inputs, intermediates, or outputs) are not sufficiently constrained, allowing the prover to assign arbitrary values while still producing a valid proof.

### Common Causes:

1- Unconstrained inputs: Inputs that never appear in any constraint

2- Unconstrained outputs: Outputs declared but not tied to internal computation

3- Dangling signals: Signals assigned (<==) but never constrained, or constrained but never assigned

4- Partial constraints: Only some relationships are enforced (e.g., checking a*b=c but not constraining a or b)

5- Missing range checks: Assuming a value is boolean or within a range without enforcing it

6- Incorrect conditionals: Using if or ternary logic that affects assignments but not constraints


### Example:

```circom
signal input a;
signal input b;
signal input flag;
signal output out;

out <== flag * a + (1 - flag) * b;
```

This circuit implements a selector (multiplexer) that chooses between two values without using control flow.

Intended behavior:

If `flag = 1` → `out = a`

If `flag = 0` → `out = b`

The circuit assumes that `flag` is a boolean value. Without an explicit boolean constraint, the prover can choose any field element for `flag`.

### The fix:

Add `flag * (flag - 1) === 0;`

[Real world under constraint bug](https://zokyo-auditing-tutorials.gitbook.io/zokyo-tutorials/tutorial-16-zero-knowledge-zk/bugs-in-the-wild/maci-1.0-under-constrained-circuit)
