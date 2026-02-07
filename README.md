# Circom-Security

## Table of Contents
   1. [Dangling Signal](#1-dangling-signal)
   2. [Under Constraint Circuit](#2-under-constraint-circuit)
   3. [Arithmetic Underflow](#3-arithmetic-underflow)
   4. [Mismatching Bit Length](#4-mismatching-bit-length)

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
In this circuit, the intention is to verify that Alice’s transfer `amount` is less than or equal to her actual `balance`, while also ensuring that the transfer amount is properly range checked.

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

## 3. Arithmetic Underflow
### Description

All arithmetic operations in Circom are evaluated over a finite field. As a result, addition, subtraction, and multiplication are performed modulo the field prime, not over bounded integers. If integer bounds are not explicitly enforced, arithmetic that is intended to model integer semantics may silently overflow or underflow while still satisfying all circuit constraints.

### Why this matters

In many circuits (balances, transfers, counters, indices), correctness relies on integer arithmetic, not field arithmetic. However, Circom does not distinguish between these two models unless explicitly instructed to do so via constraints.

A prover can therefore supply field elements that satisfy all arithmetic constraints modulo the field, yet violate the intended integer semantics.

The verifier has no way to detect this unless bounds are explicitly enforced.

### Example:

```circom

signal input balance;
signal input amount;
signal output balanceAfter;

balanceAfter <== balance - amount;
```
#### Intended semantics:
`balance ≥ amount`

`balanceAfter ≥ 0`
#### Enforced semantics:

`balanceAfter ≡ balance − amount (mod p)`

```
If `amount > balance`, then under field arithmetic:

balanceAfter ≡ balance − amount (mod p)

This results in a large field element close to `p`, rather than a negative number.
```

All constraints are satisfied, yet the circuit proves an invalid state transition.

### How to fix:
Without proper range checks, `amount` and `balance` may not represent valid non-negative integers.  
Using `Num2Bits(nBits)` ensures that `amount` is constrained to a valid non-negative integer range.

```circom
signal input balance;
signal input amount;
signal output balanceAfter;

// Range checks
component rcBalance = Num2Bits(nBits);
rcBalance.in <== balance;

component rcAmount = Num2Bits(nBits);
rcAmount.in <== amount;

// Ensure subtraction is safe
component le = LessEq(nBits);
le.in[0] <== amount;
le.in[1] <== balance;

// Compute balance safely
balanceAfter <== balance - amount;
```

## 4. Mismatching Bit Length
### Description

Circom circuits frequently rely on bit length assumptions to enforce integer semantics, especially when using components such as `Num2Bits()`, `LessThan()`, or `LessEqThan()`. A common pitfall arises when different parts of the circuit assume different bit widths for the same value. If a signal is range checked with fewer bits than are assumed by downstream arithmetic or comparisons, higher order bits may go unconstrained, leading to unintended behavior.

### Root Cause
Range checks in Circom only constrain a value to the range `[0, 2^n)` for the specific `n` used. If a signal is later used in a component that assumes a larger bit width, the circuit implicitly allows values outside the originally intended range. Because Circom operates over a finite field, these excess bits do not cause a failure and may silently affect arithmetic or comparison results.

### Example:
```circom
signal input amount;
signal input balance;
signal output out;

// amount is range-checked to 32 bits
component amtRC = Num2Bits(32);
amtRC.in <== amount;

// Comparison assumes 64-bit values
component le = LessEq(64);
le.in[0] <== amount;
le.in[1] <== balance;

out <== le.out;
```

In this example, `amount` is constrained to `[0, 2^32)`, but the comparison logic assumes 64-bit operands. The higher 32 bits of `amount` are implicitly assumed to be zero, yet this assumption is never enforced for `balance`. If `balance` is not range-checked consistently, the comparison may succeed or fail in unexpected ways depending on unconstrained higher-order bits.

### How to fix:
Ensure that all signals participating in the same arithmetic or comparison are range-checked using the same bit length.

``` circom
signal input amount;
signal input balance;
signal output ok;

// amount is range-checked to 64 bits
component amtRC = Num2Bits(64);
amtRC.in <== amount;

component balRC = Num2Bits(64);
balRC.in <== balance;

// Comparison assumes 64-bit values
component le = LessEq(64);
le.in[0] <== amount;
le.in[1] <== balance;

ok <== le.out;
```
