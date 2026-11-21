---
title: "Square Roots mod a Prime, Without the Coin Flip"
date: 2026-11-22T00:00:00+11:00
draft: false
math: true
summary: "A worked Tonelli–Shanks example shows why finding a non-square helps compute a square root modulo a prime, and where Dirichlet GRH bounds the deterministic search."
tags: ["Mathematics", "Number Theory", "Algorithms"]
---

Start with 5 and 41. Can you find a number whose square leaves a remainder of 5 after division by 41? One answer is 28: \(28^2=784=19\cdot41+5\). The puzzle is how to get there without already knowing a root.

Tonelli–Shanks gets there by first finding a number that **has no square root** modulo 41. It uses that number to build a sequence of corrections. Finding one is easy on average: half the nonzero residues qualify. But if we want to avoid guessing, we need to know how far we might have to count. Counting always works; the Generalized Riemann Hypothesis (GRH) for Dirichlet \(L\)-functions is what gives this search a worst-case bound polynomial in the bit length of the prime.

## Find a non-square first

Modulo an odd prime \(p\), two nonzero numbers can have the same square only if they are equal or negatives of one another. If \(x^2\equiv y^2\pmod p\), then \(p\) divides \((x-y)(x+y)\), so it must divide one of those factors. The \(p-1\) nonzero residues therefore pair off into \((p-1)/2\) distinct squares. Zero has a root of its own.

Euler’s criterion distinguishes the two halves. For a nonzero residue \(u\), compute \(u^{(p-1)/2}\bmod p\). The result is \(1\) if \(u\) is a square and \(-1\) otherwise. We use that test first on the input \(a=5\), then on candidates for the non-square \(z\):

| Number tested | Result modulo 41 | What it tells us |
| --- | ---: | --- |
| \(5^{20}\) | \(1\) | The input has a square root. |
| \(2^{20}\) | \(1\) | Skip 2: it is a square. |
| \(3^{20}\) | \(-1\) | Use 3 as the non-square. |

Some primes let us bypass this search. If \(p\equiv3\pmod4\) and \(a\) is a nonzero square, \(a^{(p+1)/4}\) is a root: squaring gives \(a\cdot a^{(p-1)/2}\equiv a\). There is also a shortcut for \(p\equiv5\pmod8\). Neither applies to 41, so we can see the general correction loop at work.

## Correct a candidate until its square matches

The corrections act on the power-of-two part of the multiplicative group. To isolate it, write \(p-1=q\,2^s\) with \(q\) odd. For 41, this is \(40=5\cdot2^3\). Form a candidate root \(x=a^{(q+1)/2}\) and an extra factor \(t=a^q\). They are linked by

\[
x^2\equiv a\,t\pmod p.
\]

The goal is to make \(t=1\). We can multiply \(x\) by a value \(b\) without breaking the link, provided we also multiply \(t\) by \(b^2\). The non-square tells us how to pick \(b\) so that the new \(t\) is closer to 1 in a precise sense: it takes fewer repeated squarings to get there.

For \(a=5\) modulo 41, we start with \(x=5^3\equiv2\) and \(t=5^5\equiv9\). Raise the non-square \(z=3\) to the same odd power to get \(c=z^q\equiv38\). Now square \(t\) and \(c\) repeatedly:

\[
\begin{aligned}
t:\quad &9 \longrightarrow 40 \longrightarrow 1,\\
c:\quad &38 \longrightarrow 9 \longrightarrow 40 \longrightarrow 1.
\end{aligned}
\]

Each arrow squares the value modulo 41, where 40 represents \(-1\). On the first round choose \(b=38\). The square \(b^2=9\) takes the same number of squarings to reach 1 as \(t\) does. Their product is \(9\cdot9\equiv40\), which takes only one. On the second round use \(b=9\), so \(b^2\equiv40\) and the new factor is \(40\cdot40\equiv1\).

| Correction | Candidate \(x\) | Extra factor \(t\) | \(x^2\bmod 41\) | \(5t\bmod 41\) |
| --- | ---: | ---: | ---: | ---: |
| Initial values | 2 | 9 | 4 | 4 |
| Multiply \(x\) by 38 | 35 | 40 | 36 | 36 |
| Multiply \(x\) by 9 | 28 | 1 | 5 | 5 |

The last row gives \(28^2\equiv5\pmod{41}\). The other root is \(41-28=13\).

The same argument works for any odd prime. At the start, Euler’s criterion gives \(t^{2^{s-1}}=1\) and \(c^{2^{s-1}}=-1\) modulo \(p\). So \(c\) takes exactly \(s\) squarings to reach 1 and \(t\) takes fewer.

At the start of a round with \(t\ne1\), let \(m\) be the number of squarings needed for \(c\) to reach 1 and \(i\) that number for \(t\). We have \(0\lt i\lt m\). Set \(b=c^{2^{m-i-1}}\) so \(b^2\) also needs exactly \(i\) squarings. One step before 1, both \(t\) and \(b^2\) reach \(-1\): it is the only square root of 1 other than 1 itself. Therefore

\[
(tb^2)^{2^{i-1}}\equiv(-1)(-1)=1\pmod p.
\]

The new \(t=tb^2\) takes at most \(i-1\) squarings, while the new \(c=b^2\) takes exactly \(i\). That is why the rule still applies on the next round, and why the loop must terminate. When it does, \(t=1\) and the invariant gives \(x^2\equiv a\).

## Bound the search for the first non-square

Everything so far is unconditional. The remaining question is how many candidates we must test before the correction loop can start. Independent uniform guesses take two trials on average. Counting from 2 finds the smallest positive non-square, \(n_p\), but the fact that half the residues are non-squares does not control how many squares can appear at the start.

Burgess’s unconditional bound is \(n_p=O_\varepsilon(p^{1/(4\sqrt e)+\varepsilon})\) for every fixed \(\varepsilon>0\). The exponent \(1/(4\sqrt e)\) is about 0.152. For \(k\)-bit primes, this gives an upper bound exponential in \(k\), not a proof that the search ever takes that long. [Banks and Makarov on the Burgess bound][burgess]

To see where GRH helps, give every nonzero square the value \(+1\), every non-square \(-1\), and every multiple of \(p\) zero. This is the *quadratic character* modulo \(p\). If the search has found only squares up to a given point, this character is \(+1\) at every prime up to that point. Ankeny used the zeros of Dirichlet \(L\)-functions to bound how long such a run can be. GRH places every zero in the critical strip on the line with real part \(1/2\). Restricting where the zeros can lie gives tighter control over how much the character can favour \(+1\) on small primes. Bach made the bound explicit. For prime moduli, Lamzouri, Li and Soundararajan gave a sharper bound under the same hypothesis:

\[
n_p \lt  (\ln p)^2 \qquad (p\ge5).
\]

Here \(\ln\) is the natural logarithm. The remaining odd prime, 3, has first non-square 2. For a 256-bit prime, the bound puts the first non-square below 31,500. The implementation does not need to compute the bound: it counts until it finds a witness. Each trial takes \(O(k)\) modular multiplications by repeated squaring; the search uses \(O(k^2)\) trials under the bound, and the correction loop takes \(O(s^2)\) modular multiplications with \(s\lt k\). Modular multiplication itself is polynomial in \(k\), so this gives a deterministic polynomial-time algorithm for residues \(0\le a\lt p\), even when both \(a\) and \(p\) vary. If \(a\) is a larger integer, first reduce it modulo \(p\). [Ankeny’s bound][ankeny] · [Bach’s explicit bound][bach] · [Lamzouri, Li and Soundararajan, Corollary 1.1][lls]

The distinction between classical RH and the Dirichlet version matters here. On 20 November 2026, Levent Alpöge announced a proof developed with Claude and released the Lean formalization. The Lean and Mathlib community statement certifies the classical theorem. Andrew Sutherland’s separate verification record includes the formal theorem for Dirichlet \(L\)-functions used here: independent checkers accepted the exported proof, and statement matching checked that the exported result proved the expected theorem. The runtime argument above uses that Dirichlet statement; classical RH alone does not supply it. [Alpöge’s announcement][announcement] · [Lean and Mathlib statement][lean] · [Sutherland’s verification record][verification]

## The Python implementation

The function takes an integer `a` and an odd prime `p`; it does not check primality. It returns one root if one exists and `None` otherwise. Python’s `pow(a, e, p)` computes \(a^e\bmod p\). The example follows the general loop, even for primes with shortcuts.

```python
def sqrt_mod(a, p):
    """Square root of a modulo an odd prime p.

    Returns x with x*x % p == a % p, or None if
    a is not a square. Uses no randomness.
    """
    a %= p
    if a == 0:
        return 0
    if pow(a, (p - 1) // 2, p) != 1:
        return None  # not a square (Euler)

    # 1. Find a non-square z, counting up from 2.
    #    Under GRH, z < ln(p)**2 for p >= 5.
    z = 2
    while pow(z, (p - 1) // 2, p) != p - 1:
        z += 1

    # 2. Write p - 1 = q * 2**s with q odd.
    q, s = p - 1, 0
    while q % 2 == 0:
        q //= 2
        s += 1

    # 3. Correct x while preserving x*x == a*t (mod p).
    c = pow(z, q, p)
    x = pow(a, (q + 1) // 2, p)
    t = pow(a, q, p)
    m = s
    while t != 1:
        # Count squarings until t reaches 1 modulo p.
        i, u = 0, t
        while u != 1:
            u = u * u % p
            i += 1
        b = pow(c, 1 << (m - i - 1), p)
        x = x * b % p
        t = t * b * b % p
        c = b * b % p
        m = i
    return x
```

```pycon
>>> sqrt_mod(5, 41)
28
>>> sqrt_mod(3, 41) is None
True
```

## Checking the implementation

For each of the 668 odd primes below 5,000, I squared every possible root to build the full set of squares. I then called `sqrt_mod` on every residue. Across all **1,548,134 inputs**, every returned root squared to the input. The function returned `None` exactly for inputs outside the set.

For larger primes, two different quantities matter. A large \(s\) leaves room for more correction rounds. A large \(z\) means more failed trials before the corrections can begin. For the first prime in the table, \(p-1\) contains 189 factors of two and the search stops at 5.

| Prime \(p\) | Bits | \(s\) | First non-square \(z\) | \(\lfloor(\ln p)^2\rfloor\) |
| --- | ---: | ---: | ---: | ---: |
| \(3\cdot2^{189}+1\) | 191 | 189 | 5 | 17,451 |
| NIST P-224 field prime | 224 | 96 | 11 | 24,107 |
| \(2^{64}-2^{32}+1\) | 64 | 32 | 7 | 1,967 |
| BLS12-381 scalar field prime | 255 | 32 | 5 | 31,206 |
| \(15\cdot2^{27}+1\) | 31 | 27 | 11 | 458 |
| \(2^{255}-19\) | 255 | 2 | 2 | 31,241 |
| secp256k1 field prime | 256 | 1 | 3 | 31,486 |
| \(2^{521}-1\) | 521 | 1 | 3 | 130,414 |

For each prime, I tested 2,000 uniformly sampled residues and 2,000 known squares made by squaring fresh samples. I used Python’s `random.Random(20261122)` and tested the primes in table order. All 32,000 calls passed. I checked roots by squaring them and checked each `None` with Euler’s criterion. On an AMD EPYC 9V74 running CPython 3.12, the mean time per call ranged from 0.04 to 2.04 ms across the eight primes. The slowest individual call took 8.2 ms. These are measurements for this workload and machine, not a worst-case time bound.

A separate scan measured just the search for \(z\). Across all 664,578 odd primes below ten million, the mean first non-square was 3.66. The largest was 53, first reached at \(p=9{,}257{,}329\), where the sharper GRH bound allows 257.

## What the guarantee covers

The bound applies even when both \(p\) and \(a\) vary. This matters for tasks such as decompressing a point on an elliptic curve. On a curve with equation \(y^2=x^3+Ax+B\) modulo \(p\), the stored \(x\)-coordinate determines the value whose square root we need, and one stored bit selects the intended \(y\). Each point can ask for a different square root. René Schoof’s 1985 algorithm already gave a deterministic polynomial bound in \(\log p\) for a fixed integer \(a\), with a constant that depends on \(a\). [NIST SP 800-186, appendix D.2.1][nist] · [Schoof’s paper][schoof]

The small-witness bound also has a use in Miller’s primality test, where a witness is a base that exposes an integer \(n\) as composite. Bach showed that it suffices to test bases up to \(2(\ln n)^2\) under the same Dirichlet hypothesis. [Bach][bach]

The prime modulus matters. If the modulus is a product of two distinct odd primes, a general square-root algorithm would also give a randomized algorithm for factoring that product. Finding a generator of the full multiplicative group or factoring an arbitrary polynomial over a finite field likewise requires more than the non-square used here. [Handbook of Applied Cryptography, chapter 3][handbook]

For a fixed prime, the non-square depends only on \(p\), so a caller can find it once and reuse it across calls. An arbitrary prime supplied with each call is the case where the worst-case bound matters. The Python implementation shows the arithmetic, but its search and loop are variable-time; it is not a constant-time implementation for sensitive inputs.

[announcement]: https://alpo.ge/remark.pdf
[lean]: https://leanprover-community.github.io/blog/posts/riemann-hypothesis/
[verification]: https://math.mit.edu/~drew/grh/
[nist]: https://doi.org/10.6028/NIST.SP.800-186
[burgess]: https://arxiv.org/abs/1411.2009
[ankeny]: https://doi.org/10.2307/1969420
[lls]: https://arxiv.org/abs/1309.3595
[bach]: https://doi.org/10.1090/S0025-5718-1990-1023756-8
[schoof]: https://doi.org/10.1090/S0025-5718-1985-0777280-6
[handbook]: https://cacr.uwaterloo.ca/hac/about/chap3.pdf
