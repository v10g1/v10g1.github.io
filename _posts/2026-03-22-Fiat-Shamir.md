---
title: "Unhasing the Hashes: Fiat Shamir Transformation"
date: 2026-03-11 00:00:00 +0530
categories: [ZK, Mathematics]
tags: [zk]
maths: true
---


In this Blog we will be deep diving into the core workings of Fiat Shamir transformation.

Personally, this is the part I struggled the most when I was reading the Thaler [book](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf)
## The Random Oracle Model
The Random oracle model is setting, which is used to get the randomness into the system, by the means of the efficient systems, the function maps somme domain $D$ to the k-bit range ${0,1}^k$ we mean the following: on any input x $\in D$, R chooses its output $R(x)$ uniformly at a random point in the range.

Assumption
- The prover and verifer have the query acess to a random function R, which returns the same ouput for the current round.
- The Random oracle assumption is not valid in the real world, as specifyin a function R requires $|D|K$ bits essentially one must list the value $R(x)$ . 

Which is very difficult to implement in a real world case

We need to find the solution for the same.
## The Solution ?
Normally, in a Public coin protocol, then structure goes like follow
```bash
          ┌──────────────────────────┐
          │        Prover (P)        │
          │  (Computationally strong)│
          └────────────┬─────────────┘
                       │
                       │  1. Commitment / First Message
                       │  (depends on statement x)
                       ▼
          ┌──────────────────────────┐
          │       Verifier (V)       │
          │   (Randomness is public) │
          └────────────┬─────────────┘
                       │
                       │  2. Public Random Coins (r)
                       │  (sampled and revealed)
                       ▼
          ┌──────────────────────────┐
          │        Prover (P)        │
          └────────────┬─────────────┘
                       │
                       │  3. Response
                       │  (depends on r and prior msg)
                       ▼
          ┌──────────────────────────┐
          │       Verifier (V)       │
          └────────────┬─────────────┘
                       │
                       │  4. Verify:
                       │     Accept / Reject
                       ▼
                ┌──────────────┐
                │   Decision   │
                └──────────────┘
```
Key structural properties
Public coins:
The verifier samples randomness r and sends it in the clear to the prover (no hidden randomness).

Interaction pattern (3-move example):

P → V : a
V → P : r   (public randomness)
P → V : z


This is exactly the Arthur–Merlin (AM) model:
Arthur (V) = randomized polynomial-time verifier
Merlin (P) = all-powerful prover

In this system, I can think of two in-efficencies (Not vulerabilities)

1. R problem we talked about earlier

2. In blockchain system, doing back and foreth with this the Random R, would be too tedious of task

Note -> Even if the system is not a blockchain system, the time spend in this would be too much 

**So what can we do instead?**

Here comes the fiat shamir heuristics

The fiat shamir transformation replaces each of the verifier's messages from the interaction protocol I with a value derived from the random oracle, where the query point is the list of messages sent by the prover in the round 1,2,....v. 

Let me try to explain in a Lame Language.

1. Lets say you are the verfier and I am the Prover.

2. I, as a prover want to prove that I know something lets say $c_1$

3. Now I will calculate the expression which will be needed to prove for the first round.

Now pause this Prover side for now, and lets try to understand the same in the Verfier side

1. For the verifier, You will recieve a commitment from my side which i cannot change for the current round

2. You will check the, having two options in hand, accept or reject, if reject, protocol will be stopped with negative outcome.

3. If the first round is accepted, then you will try to calculate the Hash, of the message that the prover sent in the last round (will take about this part later again).

r = H(domain_separator || x || a || transcript_so_far)

4. and that Hash works as the random R, 


Usually the prover computes all the hashes recursively with the follow-up answer, before even sending the first round. 

please don't go yet!! I know this this is hard to process, by the end of this blog post I know you will understand atleast 90% of this.

## How this is represented?

Below is the diagram which my trusted friend ChatGPT made it for me 
```bash
        ┌──────────────────────┐
        │      Prover (P)      │
        └─────────┬────────────┘
                  │
                  │ a ← Commit(x)
                  │ r = H(a, x)
                  │ z ← Response(...)
                  ▼
             Proof π = (a, z)

        ┌──────────────────────┐
        │     Verifier (V)     │
        └─────────┬────────────┘
                  │
                  │ r = H(a, x)
                  │ Verify(...)
                  ▼
               Accept / Reject
```

## A Common vulnerability

for the fiat-shamir transformation ot be secure in settings where an adversary can choose the input x to the IP or argument, it is essential that x be appended to the list that is hashed in each around. This property of soundness against protects adversary than can choose x is called adaptice soundness. 

But Vrisan, How would anyone be able attack this?

## Weakness in  Fiat-shamir ?

Weak Fait shamir is the basic and insufficiently hardened version of the fiat shamir Transformation where the verifier's random challenge is replaced by a hash of only part of the transcript- typically just the prover first message.

It's weak because it omits critical bindings (to the statement, context, and full transcript), which opens the door to malleability and soundess issues in realistic settings

How? let me explain.

Lets imagine a 3-move public coin protocol

P → V : a        (commitment)

V → P : e        (random challenge)

P → V : z        (response)

with the verification 

$Verify(x,a,e,z)==1$

## Weak fiat-Shamir Transformation
Lets start with the challenge, find the bug in the following protocol.
In Weak FS, We replace the verifier's randomness with:

$e=H(a)$

> Note -> assume that the Hash function is completely fine and has not problems
So the protocol become non-interactive:

1. a ← Commit(x)

2. e ← H(a)

3. z ← Response(x, a, e)


Output proof: π = (a, z)


So what is wrong in here?

Now give yourself 1 minute and try to figure out the problem here


3.....2.....1


So There are some missing binding in the hashing function

Which are 
1. Statement X is not used
2. Full transcript is not used
3. Domain seperation is not used


Now lets analyze each of the mistake thoroughly

**Statement Malleability**
The challenge is not bound to the statement x.
If two statements $x_1$ and $x_2$ share structure:

Same a -> Same e

Independent of what statement is being proven

Attack Intuition
Suppose you have a valid proof for the statement $x_1$

$(a_1,z)$ such that Verify(x,a,H(a))==1

Now consider a related Statement $x_2$

If the verification equation is structure-preserving, then same a give you same e

You may be able to reuse or tweak to make it valid for $x_2$

**Replay Attacks**

Proofs are not instance bound

You generate the proof $\pi =(a,z)$

Now the attacker can resuse it in different contexts
- Different protocol 
- Different session
- different public input internpretation

## When a Fiat shamir implementation is Weak?
When you are auditing any system which implements the Fiat Shamir, you wanna check these first
1. Find the prover controlled values.
2. Are they in transcript?
3. no? bingo

4. Check the verification

find a way to manipulate it

Simple.



## What would be a fix?

2 words. bind EVERYTHING

Everything? 

Yes everything that can be changed by the Prover during the commitement.

It would look something like 

e = H(x || a || transcript)

## Bits of Security 
The security level of a non-interactive argument is measured by the amount of work that
must be done to find a convincing “proof” of a false statement. Similar to other cryptographic primitives
like digital signatures and collision-resistant hash functions, the logarithm of this amount of work is referred
to as the number of bits of computational security. For example, 30 bits of security implies that 230 ≈ 1
billion “steps of work” are required to attack the argument system
This is inherently an approximate
measure of real-world security because the notion of one step of work can vary, and practical considerations
like memory requirements or opportunities for parallelism are not considered.

**Appropriate security levels for interactive vs. non-interactive arguments.**

non interactive arguments are generally recommended to be deployed with at least 100 or 128 bits of computation security. In contract, it may be appropirate in some contexts to set statistical or interactive security levels lower

The key difference is that, with statistical or interactive security, the cheating prover has to actually
interact with the verifier in order to “attempt” an attack that will succeed with only tiny probability.

This is because the cheating prover in an interactive protocol is hoping to get a lucky verifier challenge, and the prover does not know whether or not the verifier challenges will be lucky untill after sending one or more messages to the verifier and recieving challenges in response.

For example, suppose that an interactive protocol is run at 60 bits of interactive security. Ths means that each attempted attack succes with the probability of at most $2^{-60}$ So after, say 2^50 attemptes, its still unlikely that the fake proof was done.


In contrast to non-interactive arguments, a cheating prover can silently attack a protocol without any interaction witht he verifier. For example if apply the fiat shamir transformation to a 3 message interactive protocol as considered in figure 5.2 below, the canonical "grinding attack" on the resulting non-interactive argument involves the prover attempting to guess a first message $\alpha$ that field a l
