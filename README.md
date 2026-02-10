# Circom-Security

## Table of Contents
   1. [Dangling Signal](#1-dangling-signal)
   2. [Under Constraint Circuit](#2-under-constraint-circuit)
   3. [Arithmetic Underflow](#3-arithmetic-underflow)
   4. [Mismatching Bit Length](#4-mismatching-bit-length)
   5. [Assigned but Not Constrained](#5-assigned-but-not-constrained)
   6. [Trusted Setup Leak](#6-trusted-setup-leak)

## 1. Dangling Signal 
### Definition

Unused or dangling signals are variables that are declared but never actually used in any computation or constraint. They can indicate mistakes, unnecessary complexity, or potential logical errors in the circuit.

### What to look for

1. Signals declared but never assigned a value.

2. Signals assigned a value but never used in any other constraint or output.

3. Signals that may have been intended for a computation but were forgotten.

### Example

```circom
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

### Why this is dangerous

`safeAmount` is correctly range-checked

The comparison logic relies on `amount`, not the range-checked signal

Therefore, the verifier does not enforce that the compared value is within 32 bits

## 2. Under Constraint Circuit
### Definition

An under-constrained circuit is when one or more signals (inputs, intermediates, or outputs) are not sufficiently constrained, allowing the prover to assign arbitrary values while still producing a valid proof.

### Common Causes

1. Unconstrained inputs: Inputs that never appear in any constraint

2. Unconstrained outputs: Outputs declared but not tied to internal computation

3. Dangling signals: Signals assigned (<==) but never constrained, or constrained but never assigned

4. Partial constraints: Only some relationships are enforced (e.g., checking a*b=c but not constraining a or b)

5. Missing range checks: Assuming a value is boolean or within a range without enforcing it

6. Incorrect conditionals: Using if or ternary logic that affects assignments but not constraints


### Example

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

### The fix

Add `flag * (flag - 1) === 0;`

[Real world under constraint bug](https://zokyo-auditing-tutorials.gitbook.io/zokyo-tutorials/tutorial-16-zero-knowledge-zk/bugs-in-the-wild/maci-1.0-under-constrained-circuit)

## 3. Arithmetic Underflow
### Description

All arithmetic operations in Circom are evaluated over a finite field. As a result, addition, subtraction, and multiplication are performed modulo the field prime, not over bounded integers. If integer bounds are not explicitly enforced, arithmetic that is intended to model integer semantics may silently overflow or underflow while still satisfying all circuit constraints.

### Why this matters

In many circuits (balances, transfers, counters, indices), correctness relies on integer arithmetic, not field arithmetic. However, Circom does not distinguish between these two models unless explicitly instructed to do so via constraints.

A prover can therefore supply field elements that satisfy all arithmetic constraints modulo the field, yet violate the intended integer semantics.

The verifier has no way to detect this unless bounds are explicitly enforced.

### Example

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

### The fix
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

### Example
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

### The fix
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

## 5. Assigned but Not Constrained
### Description
In Circom, signals may be assigned values without being properly constrained. A common pitfall occurs when a signal is set using the assignment operator `<--`, but no constraint actually enforces the intended relationship in the circuit. Since Circom is declarative and operates over a finite field, assigning a value does not, by itself, guarantee correctness unless it is backed by explicit constraints.

### Root Cause
Developers may incorrectly assume that assigning a signal using `<==` always enforces the expected semantics. However, if the assigned signal is never used in a constraint that restricts its value, the prover is free to assign any value that satisfies the remaining constraints. This often results from unused intermediate signals, forgotten equality checks, or logic that references a different signal than the one that was constrained.

### Example
```circom
template Assign() {
    signal input bool;
    signal output out;
    
    signal internal;

    internal <-- bool != 0 ? 1 : 0;

    out <== internal;
}

component main = Assign();

/* INPUT = {
    "bool": "1"
} */
```
in this example, the line
`internal <-- bool != 0 ? 0 : 1;`
uses a conditional expression, but it only assigns a value to `internal` without enforcing any constraint.

As a result, `internal` is not constrained by the circuit logic. A dishonest prover can generate a valid witness and then manually modify the value of `internal` in the `witness.wtns` file to either `0` or `1`, regardless of the value of bool.

Since no constraint ties `internal` to `bool`, the modified witness will still satisfy all enforced constraints, allowing the prover to produce a forged proof.

### output
```
non-linear constraints: 0
linear constraints: 0
public inputs: 0
public outputs: 1
private inputs: 1
private outputs: 0
wires: 2
labels: 4
```
As shown above, no linear or non-linear constraints were generated.
Although the circuit compiles successfully and produces an output (`out = 1`), this output is not enforced by any constraint. Consequently, the circuit proves no relationship between the input and the output, leaving the witness fully controlled by the prover.

### The Fix
In our example, we were trying to determine whether a given value is zero. The most secure way to achieve this is to use the `IsZero` template from Circomlib.

For all other logic, it is essential to enforce constraints on the outputs of conditional expressions.

## 6. Trusted Setup Leak
### Description
Some zk-SNARK systems require a trusted setup phase that produces toxic waste which must be destroyed after setup. If this toxic waste is leaked, retained, or reproducible by an adversary, the soundness of the proof system is compromised. An attacker who obtains the trusted setup secret can generate arbitrary valid proofs for false statements, completely breaking the security guarantees of the circuit.

### Root Cause
The security of trusted-setup-based SNARKs relies on the assumption that no single party knows the full setup secret. Risks arise when:

1. The setup is generated by a single entity

2. Intermediate secrets are logged, backed up, or version-controlled

Unlike typical circuit bugs, this issue exists outside the circuit logic, but has total impact on correctness.

### Affected Systems
1. Groth16 (per circuit setup)

2. PLONK with KZG polynomial commitments / universal SRS (trusted setup; not IPA-based/transparent variants)

### Remediation
1. Treat setup secrets as toxic waste: never store, log, or back them up

2. Use multi-party computation (MPC) ceremonies with many independent contributors

#### References
[Trusted setup by RareSkills](https://rareskills.io/post/trusted-setup)

[Evaluating a Quadratic Arithmetic Program on a Trusted Setup](https://rareskills.io/post/r1cs-to-qap)
