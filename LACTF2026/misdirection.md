# misdirectiom

> To win the game, answer the riddle: what's inconveniently long and full of misdirection?

## tl;dr

Forge NTRUSign signatures using the zero polynomial (it just… works), then race condition the server into counting past its own limit by flooding it with padded requests.

## The Setup

We get a cute snake game where you grow a snake to length 14 to win the flag. Each "grow" requires a valid NTRUSign signature for the current count. The catch: the server hard-caps growth at 4.

```python

# limit to 4 count

if current_count < 4and client_count == current_count:

    ...

if verif:

        current_count += 1

```

So you can only go 0 → 1 → 2 → 3 → 4, but you need 14. Cool. Very fair.

The server runs gunicorn with `--threads 80` and uses a global `current_count` with no locking. Interesting.

## Bug 1: The Zero Polynomial Forgery

NTRUSign verification checks if `NTRUNorm(s, s*h - m, (0, q)) < N_BOUND` where:

-`s` is the signature polynomial

-`h` is the public key

-`m` is a hash of the message

-`N_BOUND = 545`, `q = 128`

What if we just... set `s = 0`? The zero polynomial?

- First component: `centered_norm(s) = 0` (it's all zeros lol)
- Second component: `centered_norm((-m) mod 128)` — the hash of hex chars has ASCII values in [48, 102], so after negation mod 128 the centered norm works out to roughly **~381**

Since 381 < 545, the verification happily accepts our signature of *nothing*. We can sign any message without the private key. The entire NTRUSign cryptosystem is just vibes here.

I verified this empirically — tested 150 different (count, r) combinations, max norm was 414. All passed.

## Bug 2: Race Condition

Great, we can forge signatures. But the `current_count < 4` check still blocks us. Enter: threading.

The server uses 80 gunicorn threads and a global `current_count` with zero synchronization. The `/grow` handler does:

```python

if current_count < 4and client_count == current_count:

# ... verify signature (SLOW) ...

if verif:

        current_count += 1# no lock btw

        ready_status["status"] = False

        NTRU.Signing(...)      # extremely slow

        ready_status["status"] = True

```

If we send a bunch of concurrent requests that all pass the `< 4` check before any of them reaches the increment, they'll ALL increment. Thread 1 bumps 3→4, thread 2 bumps 4→5, etc.

The race window is how long verification takes. Our forged signatures bypass the signature cache (they're not in it), forcing the server through `import_signature` + `NTRU.Verifying` — which involves a pure Python O(N²) `star_multiply` loop. With the GIL doing round-robin across 80 threads every 5ms, the first thread doesn't finish verification until all the others have entered it too.

## The Padding Trick

One problem: over the network, requests don't all arrive at the same time. If verification is too fast (~25ms), only 1-2 threads make it through the race window before `ready_status` gets set to False.

The fix is beautifully dumb. The `import_signature` parser iterates char-by-char:

```python

while sig[c] != '\n':

if sig[c] == "|":

        coeff += [int(nb)]

        nb = ""

else:

        nb += sig[c]

    c += 1

```

So we pad each coefficient with 150 leading zeros: `"000...0"` × 251 coefficients. `int("000...000")` is still 0 — the signature is still valid — but parsing a ~37KB string char-by-char takes way longer. This widens the race window from ~25ms to ~30ms+ per thread, and with 80 threads GIL-sharing, the first thread doesn't complete until several seconds of wall time have passed. Plenty of time for every thread to enter the critical section.

## Putting It Together

```

1. Grow legitimately: 0 → 1 → 2 → 3 (using server-provided signatures)

2. Pre-warm 80 TCP connections to the server

3. Fire 80 concurrent POST /grow requests with padded s=0 forgeries

4. Threads race: all pass count<4, all verify, all increment

5. current_count jumps from 3 to 3+N where N = number of winning threads

6. POST /flag → profit

```

The exploit is probabilistic — sometimes you get 14+ successes on the first try, sometimes you need a retry. But with 80 threads, you almost always get enough.
