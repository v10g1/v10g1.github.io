---
title: "Unhashing the Hashes: Fiat-Shamir Transformation"
date: 2026-03-22 00:00:00 +0530
categories: [ZK, Mathematics]
tags: [zk]
math: true
image: /assets/blog3.png
---

In this post we will be deep diving into the core workings of the Fiat-Shamir transformation.

Personally, this is the part I struggled the most when reading the Thaler [book](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf).

## The Random Oracle Model

The Random Oracle Model is a setting used to get randomness into the system. The function maps some domain $D$ to the $k$-bit range $\{0,1\}^k$ — on any input $x \in D$, the oracle $R$ chooses its output $R(x)$ uniformly at random in the range.

**Assumptions:**
- Both the prover and verifier have query access to a random function $R$, which returns the same output for the same input within the current round.
- The Random Oracle assumption is not valid in the real world, as specifying a function $R$ requires $|D| \cdot k$ bits — essentially one must list the value $R(x)$ for every $x$.

This is very difficult to implement in practice, so we need a better solution.

## The Solution?

In a **public-coin protocol**, the structure goes as follows:

```
┌──────────────────────────┐
│        Prover (P)        │
│  (Computationally strong)│
└────────────┬─────────────┘
             │
             │  1. Commitment / First Message
             │     (depends on statement x)
             ▼
┌──────────────────────────┐
│       Verifier (V)       │
│   (Randomness is public) │
└────────────┬─────────────┘
             │
             │  2. Public Random Coins r
             │     (sampled and revealed)
             ▼
┌──────────────────────────┐
│        Prover (P)        │
└────────────┬─────────────┘
             │
             │  3. Response
             │     (depends on r and prior message)
             ▼
┌──────────────────────────┐
│       Verifier (V)       │
└────────────┬─────────────┘
             │
             │  4. Verify → Accept / Reject
             ▼
      ┌──────────────┐
      │   Decision   │
      └──────────────┘
```

**Key structural properties:**

- **Public coins:** The verifier samples randomness $r$ and sends it in the clear to the prover (no hidden randomness).
- **3-move interaction pattern:**

$$P \xrightarrow{\ a\ } V \xrightarrow{\ r\ } P \xrightarrow{\ z\ } V$$

This is exactly the **Arthur–Merlin (AM)** model — Arthur ($V$) is a randomized polynomial-time verifier, Merlin ($P$) is an all-powerful prover.

Two inefficiencies arise in practice:

1. Specifying the random oracle $R$ is infeasible as described above.
2. In systems like blockchains, the back-and-forth interaction cost is prohibitive.

**So what do we do instead?**

## Fiat-Shamir Transformation

The Fiat-Shamir transformation replaces each of the verifier's messages with a value derived from a hash function (standing in for the random oracle), where the query point is the full list of messages sent by the prover in rounds $1, 2, \ldots, v$.

In plain terms:

1. You (the prover) want to prove knowledge of some witness — say $c_1$.
2. You compute a commitment $a$ depending on your statement $x$.
3. Instead of waiting for the verifier to send randomness, you compute:

$$r = H(\text{domain\_separator} \,\|\, x \,\|\, a \,\|\, \text{transcript\_so\_far})$$

4. This hash output plays the role of the verifier's random challenge $r$.
5. The prover computes all challenges recursively and sends a single proof $\pi = (a, z)$ — no interaction required.

The verifier, on receiving $\pi$, recomputes the same hash and checks the proof. Both sides agree on $r$ without any back-and-forth.

## Non-Interactive Protocol Diagram

```
┌──────────────────────┐
│      Prover (P)      │
└─────────┬────────────┘
          │
          │  a ← Commit(x)
          │  r = H(domain_sep ‖ x ‖ a)
          │  z ← Response(x, a, r)
          │
          │  Proof π = (a, z)
          ▼
┌──────────────────────┐
│     Verifier (V)     │
└─────────┬────────────┘
          │
          │  r = H(domain_sep ‖ x ‖ a)
          │  Check: Verify(x, a, r, z) == 1
          ▼
       Accept / Reject
```

## Adaptive Soundness — A Critical Property

For the Fiat-Shamir transformation to be secure in settings where an adversary can **choose** the input $x$, it is essential that $x$ be included in the hash at every round. Soundness against an adversary that can choose $x$ is called **adaptive soundness**.

Without it, the door is open to statement malleability attacks.

## Weak Fiat-Shamir

**Weak Fiat-Shamir** is an insufficiently hardened version where the verifier's random challenge is replaced by a hash of only part of the transcript — typically just the prover's first message.

Given a 3-move protocol:

$$P \xrightarrow{\ a\ } V \xrightarrow{\ e\ } P \xrightarrow{\ z\ } V \quad \text{with } \mathsf{Verify}(x,a,e,z) = 1$$

Weak FS replaces the challenge with:

$$e = H(a)$$

The proof becomes $\pi = (a, z)$ where:

1. $a \leftarrow \mathsf{Commit}(x)$
2. $e \leftarrow H(a)$
3. $z \leftarrow \mathsf{Response}(x, a, e)$

> **Spot the bug.** Give yourself 60 seconds before reading on.

---

The hash is missing three critical bindings:

| Missing | Why it matters |
|---|---|
| Statement $x$ | Challenge is not tied to what is being proved |
| Full transcript | Prior messages can be swapped out |
| Domain separator | Proofs can be replayed across protocols |

### Statement Malleability

The challenge $e$ is not bound to the statement $x$.

If two statements $x_1$ and $x_2$ share the same commitment structure, then:

$$\text{same } a \implies \text{same } e \quad \text{(regardless of } x\text{)}$$

Concretely: suppose you hold a valid proof $(a, z)$ for statement $x_1$ such that $\mathsf{Verify}(x_1, a, H(a), z) = 1$. If the verification equation is structure-preserving, you may be able to reuse or tweak $z$ to produce a valid proof for a related statement $x_2$.

### Replay Attacks

Proofs are not instance-bound. A proof $\pi = (a, z)$ can be replayed across:

- Different protocol sessions
- Different public inputs
- Entirely different contexts

## Auditing Checklist

When auditing any Fiat-Shamir implementation:

1. Find all prover-controlled values in the commitment phase.
2. Check whether each one is included in the hash input.
3. If any is missing — that's your finding.
4. Check the verification equation for structural reuse.

## The Fix

Two words: **bind everything**.

Everything that the prover controls at commitment time must go into the hash:

$$e = H(\underbrace{x}_{\text{statement}} \,\|\, \underbrace{a}_{\text{commitment}} \,\|\, \underbrace{\text{transcript}}_{\text{prior msgs}} \,\|\, \underbrace{\text{ctx}}_{\text{domain sep}})$$

## Bits of Security

The security level of a non-interactive argument is the $\log_2$ of the work required to find a convincing proof of a false statement — this is called **bits of computational security**.

For example, 30 bits of security means $2^{30} \approx 1$ billion steps of work are required to break the scheme.

| Setting | Recommended security |
|---|---|
| Non-interactive arguments | $\geq 128$ bits computational |
| Interactive protocols | Lower levels acceptable |

The key asymmetry: in an **interactive** protocol at 60 bits of security, each attack attempt succeeds with probability at most $2^{-60}$. After $2^{50}$ attempts, a successful forgery is still unlikely. The prover does not learn whether the verifier's challenge will be "lucky" until after committing.

In a **non-interactive** argument, the prover can grind silently without any interaction. The canonical grinding attack on a Fiat-Shamir-transformed 3-message protocol involves the prover attempting to find a first message $\alpha$ such that $H(\alpha)$ lands in a favorable region — with no verifier interaction required per attempt.