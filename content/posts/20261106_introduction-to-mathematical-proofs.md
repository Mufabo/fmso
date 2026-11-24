---
title: "An Introduction to Mathematical Proofs: Why Every 4-Digit Palindrome Is Divisible by 11"
date: 2026-11-06
draft: true
description: "A long-form, beginner-friendly introduction to writing mathematical proofs. We take one small, surprising fact (every four-digit palindrome is divisible by 11) and use it to learn what a proof is, how to turn a claim into precise mathematics, how to write a direct proof, how proofs go wrong, and how to generalise."
summary: "1221, 4554, 9009, 7337: every four-digit palindrome is a multiple of 11. Checking examples won't tell you why, and won't tell you it's always true. A proof will. This post uses that one fact to teach how mathematical proofs work, from definitions and quantifiers to a full written proof, common mistakes, other proof techniques, and a machine-checked version."
tags: ["mathematics", "proofs", "logic", "number-theory", "divisibility", "palindromes", "lean"]
categories: ["Mathematics"]
---

Pick any four-digit number that reads the same forwards and backwards: **1221**, **4554**, **7337**, **9009**. Now divide it by 11:

```text
1221 / 11 = 111
4554 / 11 = 414
7337 / 11 = 667
9009 / 11 = 819
```

No remainders. Try a few more and you'll keep getting the same result. It looks like **every four-digit palindrome is divisible by 11**.

But "it looks like" is not how mathematics works. How do you know the next one won't break the pattern? And even if you checked all of them, would you know *why* it's true?

This post is an introduction to mathematical proof, built entirely around that one small fact. By the end you'll have:

- a clear idea of what a theorem is, what a proof is, and what isn't a proof;
- a method for turning an informal claim into precise mathematics;
- a complete, carefully written proof of the palindrome claim;
- a feel for how proofs go wrong;
- a tour of the other standard proof techniques, each illustrated on the same theme;
- a generalisation to palindromes of any length, and a look at what a fully machine-checked proof looks like.

No background beyond school algebra is needed.

## What a proof is

A **proof** is an argument that shows a statement must be true, starting from things already accepted (definitions, axioms, previously proven results) and moving forward in steps that each follow logically from what came before.

Three things make a mathematical proof different from evidence in other fields:

1. **It is certain, not probable.** A scientist who sees the same result in a thousand experiments has strong evidence. A mathematician who sees a pattern in a thousand examples has a *conjecture*. Only a proof upgrades it to a theorem.
2. **It covers every case at once.** A statement like "every four-digit palindrome" talks about all of them. A proof has to work for each one, including the ones nobody has written down.
3. **Every step is checkable.** A reader should be able to look at each line and agree that it follows from the previous ones, without having to trust the author.

The history of mathematics has plenty of patterns that held for a very long time and then failed. A classic: the polynomial n² + n + 41 produces a prime number for n = 0, 1, 2, …, 39. Forty primes in a row. At n = 40 it gives 1681 = 41 × 41.[^1] No number of examples is a substitute for an argument.

### Theorems, lemmas and conjectures

A **theorem** is a mathematical statement that has been proven. That's all the word means: a precise claim plus a proof of it.

Being a theorem is a *status*, not a property of the sentence itself. "Every four-digit palindrome is divisible by 11" was true long before anyone wrote it down, but it only becomes a theorem once someone proves it. Until then it's a **conjecture**: a statement someone believes, usually because of evidence, but hasn't proven. A conjecture can go either way. It may be proven and become a theorem, or a counterexample may show it was false all along.

Mathematicians use a handful of related words, depending on the role a statement plays:

| Word | What it is | Example from this post |
| --- | --- | --- |
| **Definition** | Fixes the meaning of a word. It isn't proven; it's agreed on. | "d divides n" means n = d · k for some integer k. |
| **Axiom** | A starting assumption accepted without proof. Everything else is built on axioms. | Here we take the ordinary rules of algebra as given, such as a(b + c) = ab + ac. |
| **Conjecture** | Believed true, but not yet proven. | The palindrome claim, after checking a few examples but before Step 3. |
| **Theorem** | A proven statement, important enough to be a main result. | Every four-digit palindrome is divisible by 11. |
| **Lemma** | A proven statement whose main job is to help prove something else. | 10ʲ ≡ (−1)ʲ (mod 11), proved in the induction section. |
| **Corollary** | A statement that follows quickly from a theorem just proven. | 4321 isn't divisible by 11, so it isn't a palindrome. |

The difference between a theorem, a lemma and a corollary is about how a statement is *used*, not how hard it is. All three are proven, so all three are equally true.

### Hypothesis and conclusion

Most theorems have the shape **"if P, then Q"**. P is the **hypothesis**: what you're allowed to assume. Q is the **conclusion**: what you have to show. Our theorem fits this shape once you rephrase it slightly:

> If n is a four-digit palindrome, then n is divisible by 11.

Here the hypothesis is "n is a four-digit palindrome" and the conclusion is "n is divisible by 11".

A theorem promises the conclusion only when the hypothesis holds. It says nothing about other numbers. 1012 is divisible by 11 even though it isn't a palindrome, and that doesn't contradict anything: the theorem never claimed only palindromes are divisible by 11. Keeping the hypothesis and conclusion apart like this is what the rest of the post builds on: a proof starts from the hypothesis and has to reach the conclusion.

### "Formal" proofs: two meanings

You'll hear the word *formal* used in two ways, and it's worth separating them early.

- In everyday mathematical writing, a "formal proof" means a **rigorous** one: precise definitions, explicit logical steps, no hand-waving. That's what textbooks and papers contain, and it's the main subject of this post.
- In logic and computer science, a **fully formal proof** is one written in a strict symbolic language where every step is an application of a fixed rule, so that a computer program can check it mechanically. We'll see one of those at the end.

A rigorous written proof is really a compressed description of a fully formal one: it leaves out steps that any competent reader can fill in. Learning where that line sits is a big part of learning to write proofs.

## Step 1: Understand the claim

Before proving anything, you need to know *exactly* what you're claiming. Mathematical statements are built from defined words, and if you're vague about the definitions, your proof will be vague too.

Our claim is:

> Every four-digit palindrome is divisible by 11.

There are three ideas in there to pin down: *four-digit number*, *palindrome*, and *divisible by*.

### Definition: divisible

For integers d and n, we say **d divides n**, written **d ∣ n**, if there is an integer k such that

```text
n = d · k
```

So "11 divides 1221" means there's an integer k with 1221 = 11 · k (namely k = 111).

Notice what this definition does for us: it turns a statement about *division* (which has remainders, fractions, and other complications) into a statement about *the existence of an integer*. That's a typical move. To prove that 11 divides something, we'll need to **produce the integer k**, or at least show one exists.

### Definition: four-digit number

A **four-digit number** is an integer n with 1000 ≤ n ≤ 9999. Equivalently, writing it in decimal as digits d₃ d₂ d₁ d₀, it is

```text
n = 1000·d₃ + 100·d₂ + 10·d₁ + d₀
```

where each digit is an integer from 0 to 9, and the leading digit d₃ is **not** 0 (otherwise "0440" would really be the three-digit number 440).

This step, writing a number in terms of its digits and place values, is the key to the whole proof. Statements about digits are statements about *notation*; algebra works on *numbers*. The place-value formula is the bridge between them.

### Definition: palindrome

A number is a **palindrome** if its decimal digits read the same forwards and backwards. For a four-digit number d₃ d₂ d₁ d₀, that means

```text
d₃ = d₀   and   d₂ = d₁
```

So every four-digit palindrome has the shape **abba**: some digit a, then b, then b again, then a. Here a is between 1 and 9 (it's the leading digit) and b is between 0 and 9.

### Putting it together: the precise statement

Now we can rewrite the claim with no ambiguity at all:

> **Theorem.** For all integers a and b with 1 ≤ a ≤ 9 and 0 ≤ b ≤ 9, the number with decimal digits a, b, b, a is divisible by 11.

Or, using the place-value formula:

> **Theorem.** For all integers a and b with 1 ≤ a ≤ 9 and 0 ≤ b ≤ 9, we have 11 ∣ (1000a + 100b + 10b + a).

This rephrasing has done a lot of work already. We started with an English sentence about "palindromes"; we now have an algebraic statement about two integers a and b. The proof will be about the algebra.

### Quantifiers: "for all" and "there exists"

Look at the structure of that theorem. It begins with **"for all"**: it is a *universal* statement. Inside, "divisible by 11" unpacks to **"there exists"** an integer k. So the full shape is:

```text
for all a, b (in the allowed ranges),
    there exists an integer k such that
        1000a + 100b + 10b + a = 11k
```

These two words, *for all* (∀) and *there exists* (∃), are the most important words in proof writing, because they tell you what kind of argument you need:

| To prove... | You must... |
| --- | --- |
| "For all x, P(x)" | Show P(x) holds for an **arbitrary** x, using only what you know about x. |
| "There exists x with P(x)" | **Exhibit** a specific x (or otherwise show one must exist) and check P(x). |
| "For all x, P(x)" is **false** | Find **one** x where P(x) fails: a counterexample. |
| "There exists x with P(x)" is **false** | Show that **every** x fails P(x). |

The order matters too. "For all a, b there exists k" means k is allowed to depend on a and b. Different palindromes get different quotients, which is exactly what we saw: 1221 gives 111, 4554 gives 414.

## Step 2: Find the idea

Proofs are written forwards but usually *discovered* backwards. Before writing anything formal, play with the problem.

Expand the palindrome **abba** using place values:

```text
abba = 1000a + 100b + 10b + a
     = 1001a + 110b
```

Now look at the coefficients. Is there something about **1001** and **110**?

```text
1001 = 11 × 91
 110 = 11 × 10
```

Both are multiples of 11. That's the whole secret. A four-digit palindrome is 1001 copies of its outer digit plus 110 copies of its inner digit, and both 1001 and 110 happen to be multiples of 11. So the total must be a multiple of 11:

```text
abba = 1001a + 110b
     = 11·91a + 11·10b
     = 11·(91a + 10b)
```

And there's our k: **k = 91a + 10b**. Check it on an example: for 4554, a = 4 and b = 5, so k = 91·4 + 10·5 = 364 + 50 = 414, matching 4554 / 11 = 414.

This is the moment the problem goes from mysterious to obvious. Most good proofs have a moment like this: one idea that makes everything fall into place. The rest is writing it down carefully.

## Step 3: Write the proof

Here is the proof written the way you'd find it in a textbook.

> **Theorem.** Every four-digit palindrome is divisible by 11.
>
> **Proof.** Let n be a four-digit palindrome. Since n has four digits and reads the same forwards and backwards, its decimal representation has the form *abba*, where a and b are digits with 1 ≤ a ≤ 9 and 0 ≤ b ≤ 9. By the place-value representation of decimal numbers,
>
> n = 1000a + 100b + 10b + a = 1001a + 110b.
>
> Since 1001 = 11 · 91 and 110 = 11 · 10, we have
>
> n = 11 · 91a + 11 · 10b = 11 · (91a + 10b).
>
> Let k = 91a + 10b. Because a and b are integers, k is an integer, and n = 11k. By the definition of divisibility, 11 ∣ n. ∎

The ∎ symbol (or "Q.E.D.", *quod erat demonstrandum*, "which was to be shown") marks the end of the proof.

### Reading the proof line by line

It's short, but every sentence is doing a specific job. Let's annotate it.

**"Let n be a four-digit palindrome."**
This is how you prove a *for all* statement. We don't pick a particular palindrome like 1221; we name an **arbitrary** one, n, and assume nothing about it except what the hypothesis gives us. Whatever we conclude about n will then hold for every four-digit palindrome, because n could have been any of them.

**"…its decimal representation has the form abba, where 1 ≤ a ≤ 9 and 0 ≤ b ≤ 9."**
This *unpacks the definitions* of "four-digit" and "palindrome". We've turned the hypothesis into something we can calculate with. Note the ranges: they come straight from the definitions, and it's good practice to state them, even though this particular proof only needs a and b to be integers.

**"By the place-value representation…, n = 1000a + 100b + 10b + a = 1001a + 110b."**
This is the bridge from digits to arithmetic. The phrase "by the place-value representation" is a **justification**: it tells the reader which known fact licenses this step. Then a routine simplification.

**"Since 1001 = 11 · 91 and 110 = 11 · 10…"**
Two facts the reader can verify by multiplication. A proof doesn't need to justify arithmetic like this further.

**"n = 11 · (91a + 10b)."**
Factoring out the common 11, using the distributive law.

**"Let k = 91a + 10b. Because a and b are integers, k is an integer…"**
This is how you prove a *there exists* statement: **exhibit the witness**. The definition of divisibility asks for an *integer* k, so we must say why k is an integer. (Sums and products of integers are integers.) It's a small point, but skipping it is a genuine gap: if k were allowed to be a fraction, every number would be "divisible" by 11.

**"By the definition of divisibility, 11 ∣ n."**
We close the loop by pointing back to the definition we started from. The conclusion is exactly the claim.

### The anatomy of a direct proof

Zooming out, the proof has a shape you'll see over and over:

1. **Assume the hypothesis**, for an arbitrary object. ("Let n be a four-digit palindrome.")
2. **Unpack definitions** into something you can work with. ("n = abba = 1001a + 110b.")
3. **Manipulate** using known facts. ("1001 and 110 are multiples of 11, so factor.")
4. **Repack** into the form required by the conclusion's definition. ("n = 11k with k an integer.")
5. **State the conclusion.** ("So 11 ∣ n.")

This is a **direct proof**: start from what you know, walk forward to what you want. It's the most common kind of proof, and the first one to reach for.

## What *isn't* a proof

It's just as instructive to look at arguments that feel convincing but don't count, or that count but aren't satisfying.

### "I checked a lot of examples"

> 1001, 1111, 1221, 2332, 4554, 7337, 9009 are all divisible by 11, so all four-digit palindromes are.

Seven examples out of ninety. This is evidence, not proof. As the n² + n + 41 example showed, patterns can hold for a long time and then fail.

### "I checked *all* the examples"

There are exactly 9 × 10 = 90 four-digit palindromes (9 choices for a, 10 for b). A computer can check all of them in a fraction of a second:

```python
palindromes = [n for n in range(1000, 10000) if str(n) == str(n)[::-1]]
print(len(palindromes))                          # 90
print(all(n % 11 == 0 for n in palindromes))     # True
```

Is that a proof? **Yes, actually.** It's a **proof by exhaustion**: when a statement is about finitely many objects, checking every one of them is logically valid. Some famous theorems, like the Four Colour Theorem, were proved by reducing them to a large but finite set of cases and checking those by computer.[^2]

But it's a proof with two weaknesses:

- **It doesn't explain anything.** It tells you *that* the claim is true, not *why*. You'd learn nothing about six-digit palindromes, or about why 11 is special.
- **It doesn't scale.** The algebraic proof adapts to palindromes of any even length (we'll do this below). Exhaustion runs out of road as soon as the set is infinite.

Mathematicians value proofs that explain. The algebraic proof shows you the mechanism: 1001 and 110 are multiples of 11.

### "It's obvious"

> Palindromes are symmetric, and 11 is symmetric, so obviously palindromes are divisible by 11.

This sounds like an explanation but isn't one. "Symmetric" isn't defined, and nothing connects the symmetry of the digits "11" to divisibility. In fact, the reasoning predicts something false: **131** is a palindrome, but 131 = 11 × 11 + 10, so it's *not* divisible by 11. Any argument that "proves" a false statement is broken.

### Proving the converse by accident

A classic logical slip is proving, or assuming, the **converse** of a statement. Our theorem says:

> If n is a four-digit palindrome, then 11 ∣ n.

The converse is:

> If n is a four-digit number and 11 ∣ n, then n is a palindrome.

These are completely different claims, and the converse is false. One counterexample is enough to kill it: **1012 = 11 × 92**, which is divisible by 11 but is not a palindrome. Only 90 of the 819 four-digit multiples of 11 are palindromes.

A related mistake is "proving" the theorem by starting from the conclusion: assuming n = 11k and then showing something about palindromes. When you work backwards to *discover* a proof (as we did in Step 2), that's fine. But the written proof must run forwards, from hypothesis to conclusion.

### Forgetting the edge cases

Our definition required the leading digit a to be at least 1. Does the proof break if a = 0? Not here: the number would be 0bb0 = 110b, which is still a multiple of 11. But it would no longer be a four-digit number (0440 is just 440), so it's simply outside the theorem's scope. Being explicit about ranges is how you notice when an edge case *does* matter. In many proofs (division, square roots, logarithms) forgetting that something could be zero is exactly where the error hides.

## The same theorem, proved other ways

A single statement usually has many proofs. Seeing several side by side is one of the best ways to learn the main techniques.

### Proof using the divisibility rule for 11

You may have learned a shortcut at school: a number is divisible by 11 if and only if the **alternating sum** of its digits (from the right: +, −, +, −, …) is divisible by 11.[^3] For 4554: 4 − 5 + 5 − 4 = 0, and 0 is divisible by 11.

If you're allowed to use that rule as a known result, the proof is one line:

> **Proof.** The alternating digit sum of *abba* is a − b + b − a = 0, which is divisible by 11. By the divisibility rule for 11, so is *abba*. ∎

This shows something important about proofs: **what counts as a proof depends on what you're allowed to assume.** The proof is short because the hard work is hidden in the rule. To be complete, you'd need to know the rule has itself been proven. (It has; the reason is that 10 is one less than 11, so powers of 10 are alternately one more and one less than multiples of 11. We'll use this idea below.)

### Proof with modular arithmetic

**Modular arithmetic** is the arithmetic of remainders. We write

```text
x ≡ y (mod 11)
```

to mean "x and y leave the same remainder when divided by 11", or equivalently, 11 ∣ (x − y). Congruences can be added and multiplied just like equations.

The key fact is:

```text
10 ≡ −1 (mod 11)
```

because 10 − (−1) = 11. Raising both sides to powers:

```text
10² ≡ (−1)² = 1   (mod 11)
10³ ≡ (−1)³ = −1  (mod 11)
```

> **Proof.** Let n = abba = 1000a + 100b + 10b + a. Working modulo 11, and using 10 ≡ −1,
>
> n ≡ (−1)³a + (−1)²b + (−1)b + a = −a + b − b + a = 0 (mod 11).
>
> So 11 ∣ n. ∎

This proof is more abstract, but it shows you the real reason the theorem works: in a four-digit palindrome, each digit appears once in an "odd" position and once in an "even" position, and modulo 11 those positions carry opposite signs, so everything cancels. It also explains the divisibility rule above.

### Proof by contradiction

In a **proof by contradiction**, you assume the statement is false and show that this leads to something impossible.

> **Proof.** Suppose, for contradiction, that some four-digit palindrome n = abba is *not* divisible by 11. But n = 1001a + 110b = 11(91a + 10b), and 91a + 10b is an integer, so 11 ∣ n. This contradicts our assumption. Hence every four-digit palindrome is divisible by 11. ∎

This is valid, but notice that the contradiction part adds nothing: the "proof" inside is just the direct proof. That's a common beginner's habit, and a good rule of thumb is: **if your proof by contradiction never actually uses the assumption you're contradicting, write it as a direct proof instead.** Contradiction earns its keep when the assumption gives you something to work with, like the famous proof that √2 is irrational, or that there are infinitely many primes.

### Proof by contrapositive

Every statement "if P, then Q" is logically equivalent to its **contrapositive**: "if not Q, then not P". (Unlike the converse, which is not equivalent.) For us:

> If a four-digit number is **not** divisible by 11, then it is **not** a palindrome.

Proving this is the same as proving the original theorem. Here it doesn't make things easier, but for some statements the contrapositive is much more natural to attack. It also gives you a free, true corollary: since 4321 leaves a remainder when divided by 11 (4321 = 11 × 392 + 9), you know immediately, without looking at the digits, that it can't be a palindrome.

### Disproof by counterexample

Is the claim true for palindromes of every length? Let's test the natural generalisation:

> **Claim (false).** Every palindrome with at least two digits is divisible by 11.

To disprove a "for all" statement, you need just one counterexample. **131** is a three-digit palindrome, and 131 = 11 × 11 + 10, so it isn't divisible by 11. The claim is false. **101** works too.

Notice the asymmetry: proving a universal statement needs a general argument; disproving it needs a single, specific case. And a good counterexample doesn't just kill a claim, it points at the true one. Both of ours have an **odd** number of digits. That's a hint.

## Generalising: palindromes of even length

Whenever you prove something, it's worth asking what the proof *actually used*. Ours used one thing: in a four-digit palindrome, every digit appears in a pair of positions whose powers of 10 have opposite signs modulo 11. That doesn't depend on the number having exactly four digits. It depends on the number having an **even** number of digits.

> **Theorem.** Every palindrome with an even number of digits is divisible by 11.

Two-digit palindromes are 11, 22, …, 99, so this is clearly true there. Six-digit: 123321 = 11 × 11211. Eight-digit: 12344321 = 11 × 1122211. It holds.

> **Proof.** Let n be a palindrome with 2m digits, for some integer m ≥ 1. Write its digits as d₂ₘ₋₁ … d₁ d₀, so
>
> n = d₀ + 10·d₁ + 10²·d₂ + ⋯ + 10²ᵐ⁻¹·d₂ₘ₋₁.
>
> Because n is a palindrome, the digit in position i equals the digit in position 2m − 1 − i, for every i from 0 to 2m − 1. So the digits come in pairs (position i and position 2m − 1 − i), with each pair holding the same digit. Since 2m − 1 is odd, the two positions in a pair always have **opposite parity**: one even, one odd.
>
> Now work modulo 11. Since 10 ≡ −1, a digit d in position j contributes 10ʲ·d ≡ (−1)ʲ·d. In each pair, one position is even and one is odd, so the pair contributes d·(+1) + d·(−1) = 0. The pairs cover all 2m positions exactly once, so
>
> n ≡ 0 (mod 11),
>
> which means 11 ∣ n. ∎

One detail to check: could a position be paired with itself? That would need i = 2m − 1 − i, so 2i = 2m − 1, an even number equal to an odd number. Impossible. So every digit really does have a distinct partner. In an **odd**-length palindrome, the middle digit pairs with itself and nothing cancels it, which is exactly why 131 fails.

This is how mathematics tends to grow: prove a specific case, understand why the proof works, find the general statement that the same idea proves, and use counterexamples to find its boundary.

## A tour of the standard proof techniques

We've now seen most of the basic proof techniques. Here they are in one place, with when to reach for each.

| Technique | Shape | When it's useful | Our example |
| --- | --- | --- | --- |
| **Direct proof** | Assume P, deduce Q. | The default. Try it first. | 1001a + 110b = 11(91a + 10b) |
| **Exhaustion / cases** | Split into finitely many cases, prove each. | Finite sets; or when different cases behave differently (even/odd, positive/negative). | Check all 90 palindromes. |
| **Contrapositive** | Assume not Q, deduce not P. | When "not Q" is easier to work with than P. | Not divisible ⇒ not a palindrome. |
| **Contradiction** | Assume not (statement), derive something impossible. | When the negation gives you a concrete object to reason about. | (Works, but adds nothing here.) |
| **Counterexample** | Find one case where it fails. | Disproving "for all" statements. | 131 is not divisible by 11. |
| **Induction** | Prove for a base case, then show case k implies case k + 1. | Statements about all natural numbers, or anything built step by step. | See below. |

### A taste of induction

**Mathematical induction** is the standard tool for proving that something holds for every natural number. It works like dominoes: show the first one falls (the **base case**), and show that each one knocks over the next (the **inductive step**). Then they all fall.

The modular proof above relied on 10ʲ ≡ (−1)ʲ (mod 11) for every j ≥ 0. We took it for granted, but here's how you'd prove it by induction:

> **Lemma.** For every integer j ≥ 0, 10ʲ ≡ (−1)ʲ (mod 11).
>
> **Proof.** By induction on j.
>
> *Base case* (j = 0): 10⁰ = 1 = (−1)⁰, so the congruence holds.
>
> *Inductive step:* Suppose 10ʲ ≡ (−1)ʲ (mod 11) for some j ≥ 0. Then
>
> 10ʲ⁺¹ = 10 · 10ʲ ≡ (−1) · (−1)ʲ = (−1)ʲ⁺¹ (mod 11),
>
> using 10 ≡ −1 and the inductive hypothesis. So the congruence holds for j + 1.
>
> By induction, it holds for all j ≥ 0. ∎

This is a nice example of how proofs **stack**. Theorems are built on lemmas, lemmas on definitions and axioms. A well-written proof makes clear what it's resting on.

## Writing proofs well

Correctness is the minimum. A good proof is also readable. Some habits that help:

- **State what you're proving** before you start, precisely.
- **Name your objects.** "Let n be…", "Write n as abba where…". Then use those names consistently.
- **Say which kind of proof it is** when it's not a direct proof: "We argue by contradiction", "By induction on m". It tells the reader what to expect.
- **Justify each non-obvious step**, briefly: "by the definition of divisibility", "since 2m − 1 is odd". Skip justifications only for routine arithmetic and algebra.
- **Use words, not just symbols.** A proof is an argument written in sentences. Chains of equations with no words between them are hard to follow, and it's easy to hide a gap in them.
- **Watch your quantifiers.** "For an arbitrary n" and "for some particular n" are completely different. Mixing them up is the single most common serious error.
- **Check the edge cases**: zero, one, the smallest allowed value, empty sets, odd vs even.
- **Re-read it as a sceptic.** Imagine a reader who will only accept what you've actually written. Where would they object?
- **Finish clearly.** Restate the conclusion and mark the end.

And a habit for *finding* proofs rather than writing them: **work examples first**. Our proof came from expanding 4554 and noticing 1001 and 110. Examples can't prove a general claim, but they're the best source of ideas for how to.

## Fully formal proofs: letting a computer check

At the start we separated rigorous proofs from **fully formal** ones, written in a language precise enough for software to verify. Programs that do this are called **proof assistants**; well-known ones include Lean, Coq (now Rocq),[^4] Isabelle and Agda. Large parts of mathematics have been formalised in them, and mathematicians increasingly use them to check difficult new results.

Here's the core of our theorem in **Lean 4** with its mathematics library, Mathlib:

```lean
import Mathlib

-- A four-digit palindrome with outer digit a and inner digit b
-- is 1000a + 100b + 10b + a. It is divisible by 11.
theorem palindrome_four_digit_dvd_eleven (a b : ℕ) :
    11 ∣ 1000 * a + 100 * b + 10 * b + a :=
  ⟨91 * a + 10 * b, by ring⟩
```

Read it like the written proof:

- The statement says: for all natural numbers a and b, 11 divides 1000a + 100b + 10b + a.
- In Lean, `11 ∣ n` is defined as "there exists c with n = 11 * c", exactly our definition of divisibility.
- `⟨91 * a + 10 * b, …⟩` **exhibits the witness** k = 91a + 10b.
- `by ring` asks Lean to verify that 1000a + 100b + 10b + a really equals 11 · (91a + 10b), using the algebra of commutative rings.[^5]

If any step were wrong, Lean would refuse to accept the theorem. The checker doesn't care how convincing the argument sounds; it only cares whether every step follows from the rules.

Notice that the formal version doesn't need the digit ranges at all: it proves the algebraic fact for *every* pair of natural numbers, which is stronger. Translating "the number with digits abba" into Lean precisely, with `Nat.digits` and a definition of palindrome, is possible but takes more work. That gap between the informal statement and its formalisation is itself a lesson: most of the subtlety in a proof often lives in getting the **statement** right.

## Bottom line

- **A theorem is a statement that has been proven.** Until it's proven it's a conjecture. Lemmas and corollaries are proven too; the names only describe the role a result plays.
- **A proof is a step-by-step argument that a statement must be true**, from definitions and known results, in a way any reader can check. Examples are evidence; only a proof makes a theorem.
- **Start by making the claim precise.** Unpack every definition. For our theorem: "four-digit palindrome" became *abba* with 1 ≤ a ≤ 9, and "divisible by 11" became "equals 11k for an integer k".
- **Quantifiers tell you what to do.** "For all" means argue about an arbitrary object. "There exists" means produce a witness. One counterexample disproves a "for all".
- **Find the idea, then write it carefully.** *abba* = 1001a + 110b = 11(91a + 10b). That one observation is the whole proof; the rest is presentation.
- **The same theorem has many proofs.** Direct, exhaustive, by divisibility rule, modular, by contradiction. Each teaches a technique, and the best one explains *why*.
- **Ask what your proof actually used.** Ours used only the even number of digits, which gives the general theorem: every even-length palindrome is divisible by 11. The counterexample 131 shows why odd lengths fail.
- **Proofs can be checked by machine.** In a proof assistant like Lean, the four-digit case is three lines, and the computer verifies every step.

## Further reading

- Daniel J. Velleman, [*How to Prove It: A Structured Approach*](https://doi.org/10.1017/9781108539890) (Cambridge University Press). The standard introduction to proof-writing, with a strong focus on logic and quantifiers.
- Richard Hammack, [*Book of Proof*](https://richardhammack.github.io/BookOfProof/). A clear introductory text, freely available online from the author.
- George Pólya, [*How to Solve It*](https://doi.org/10.1515/9781400828678). The classic on how to *find* proofs: heuristics like working examples, specialising, and generalising.
- Kevin Houston, [*How to Think Like a Mathematician*](https://doi.org/10.1017/CBO9780511808258) (Cambridge University Press). A practical guide for students starting university mathematics.
- [*Mathematics in Lean*](https://leanprover-community.github.io/mathematics_in_lean/) and [*Theorem Proving in Lean 4*](https://lean-lang.org/theorem_proving_in_lean4/), from the Lean community. Free tutorials for writing fully formal proofs.

[^1]: Weisstein EW. [Prime-Generating Polynomial](https://mathworld.wolfram.com/Prime-GeneratingPolynomial.html), Wolfram MathWorld.
[^2]: Weisstein EW. [Four-Color Theorem](https://mathworld.wolfram.com/Four-ColorTheorem.html), Wolfram MathWorld.
[^3]: Weisstein EW. [Divisibility Tests](https://mathworld.wolfram.com/DivisibilityTests.html), Wolfram MathWorld.
[^4]: [The Rocq Prover](https://rocq-prover.org/), formerly known as the Coq Proof Assistant.
[^5]: [Mathlib.Tactic.Ring.Basic](https://leanprover-community.github.io/mathlib4_docs/Mathlib/Tactic/Ring/Basic.html), Mathlib documentation for the `ring` tactic.
