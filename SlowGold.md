# slow-gold — LACTF 2025

**crypto / 500 pts / 0 solves during CTF**

> *The scenes of life are gold, so let's take it slow. After all, how often is it that you get a second chance?*

## tl;dr

Off-by-one in a for loop turns a perfectly good ZK proof into a confessional. We make the server leak its secrets one multiplication gate at a time, reconstruct a degree-10 polynomial, factor it, and win.

## The Setup

You connect to a server that runs a zero-knowledge permutation proof built on [emp-zk](https://github.com/emp-toolkit/emp-zk). Alice (server) has two secret 10-element arrays, `vec1` and `vec2`, which are permutations of each other. The protocol proves they're permutations using the classic Schwartz-Zippel trick: for a random challenge `X` chosen by Bob,

$$\prod_{i=0}^{9}(\texttt{vec1}[i]+X) = \prod_{i=0}^{9}(\texttt{vec2}[i]+X)$$

If the products match, the arrays are (overwhelmingly likely) permutations. All arithmetic is mod `p = 2^61 - 1`.

After the proof, you get to guess the contents of `vec1`. Get it right, get the flag. Get it wrong, get existential dread.

## The Bug

I spent an embarrassing amount of time staring at `emp-zk` internals before finding this gem in `ostriple.h`, inside `andgate_correctness_check`:

```cpp
for (uint32_t i = start + task_n - 1, k = 0; i < start + task_n; ++i, ++k)
```

Read that again. `i` starts at `start + task_n - 1` and the condition is `i < start + task_n`. That loop runs **exactly once**. It's supposed to iterate over all the multiplication gates to verify them. Instead it checks only the very last one and calls it a day.

This means 19 out of 20 multiplication gates in our circuit go completely unchecked. The ZK proof is about as zero-knowledge as a glass bathroom.

## What Leaks

When Bob participates in the andgate check, he already knows his own MAC keys (`ka`, `kb`, `kc`), the global MAC key `delta`, and the random challenge `chi`. Alice sends him values `U` and `V` for the checked gate. From these, Bob can compute:

```
F = V / chi + kc = a·kb + b·ka + delta·a·b
```

where `a` and `b` are the two inputs to that last multiplication gate. Because of how the circuit is structured, the last gate computes:

- `b = vec2[9] + X`
- `a = ∏(vec2[i] + X)` for `i = 0..8`

So `F` is a known linear combination of `a`, `b`, and `a·b` with coefficients that change every connection (since the MAC keys are freshly random each time). The *values* `a` and `b`, however, only depend on the secrets and our chosen `X`.

## Phase 1: Recover vec2[9]

Connect twice with the **same** `X`. We get two equations:

```
F₁ = a·kb₁ + b·ka₁ + δ₁·a·b
F₂ = a·kb₂ + b·ka₂ + δ₂·a·b
```

Same `a`, same `b`, different random keys. Solve both for `a` and set them equal — the `a` terms cancel and you're left with a **quadratic in `b`**. Quadratics over GF(p) are very solvable. We get (at most) two candidates for `b`, which gives us `vec2[9] = b - X`. Easy to tell which root is correct by repeating with a different `X`.

## Phase 2: Evaluate the polynomial

Now that we know `vec2[9]`, we can compute `b = vec2[9] + X` for any `X`. Connect 10 more times with distinct `X` values, compute `a = (F - b·ka) / (kb + delta·b)` for each, and recover `T = a·b = P(X)` where:

$$P(X) = \prod_{i=0}^{9}(\texttt{vec1}[i]+X)$$

We now have 11 evaluations (including the ones from phase 1) of a degree-10 polynomial.

## Phase 3: Recover vec1

Lagrange interpolate to get the coefficients of `P(X)`, then find all 10 roots. The roots are `-vec1[i] mod p`.

For root-finding I used Cantor-Zassenhaus, because the secret values are random 61-bit field elements — brute force isn't exactly an option.

## Phase 4: Profit

Submit the recovered values as guesses. The server checks if they're a permutation of `vec1`, and since they literally *are* `vec1`, we get the flag.

## Flag

```
lactf{1_h0p3_y0u_l1v3_th1s_0ne_t0_th3_fullest}
```

## Pain Points

- The remote server was extremely flaky. Connections dropped constantly, timeouts were generous (read: slow), and sometimes it just refused to talk. I ended up setting up a local copy with `socat` to debug the math before pointing it at remote.
- The `emp-zk` library is massive and poorly documented (sorry emp-toolkit maintainers, I still love you). Figuring out *which* gate was the last one and what values correspond to what took longer than the actual math.
- I had to instrument the C++ client to print MAC keys to stderr, rebuild it, and pipe everything through a Python harness. If you've ever debugged `__uint128_t` arithmetic at 3am, you know the vibes.
