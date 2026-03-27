---
title: "Hidden in Plain Sight Writeup"
date: 2026-03-27 00:00:00 +0530
categories: [ZK, Writeups]
tags: [zk, zkHack-Puzzles,writeups,solutions]
math: true
image: /assets/HiddenInPlainSightImage.png
---

In this blog we will be seeing the in depth analysis of the ZkHack 1 puzzle titled "Hidden in Plain Sight"
You can check puzzle [here](https://github.com/kobigurk/zkhack-hidden-in-plain-sight) 

## Initial Run 

When I executed the commands specified in the `Readme.md`

This is what we can see

```bash
zkhack-hidden-in-plain-sight git:(main) ✗ cargo run --release
   Compiling hidden-in-plain-sight v0.1.0 (/mnt/d/files/Zk-hack-puzzles/zkhack-hidden-in-plain-sight)
    Finished `release` profile [optimized] target(s) in 4.24s
     Running `target/release/verify-hidden-in-plain-sight`

    ______ _   __  _   _            _
    |___  /| | / / | | | |          | |
       / / | |/ /  | |_| | __ _  ___| | __
      / /  |    \  |  _  |/ _` |/ __| |/ /
    ./ /___| |\  \ | | | | (_| | (__|   <
    \_____/\_| \_/ \_| |_/\__,_|\___|_|\_\
    

One-thousand accounts participate in a shielded pool which hides the recipients and other data in each transaction between parties in the pool. The *recipient* is a 256-bit account address which is hidden by blinding a KZG-like polynomial commitment to the address. The sender of the transaction chooses two secret *blinding factors* known only to them by which the polynomial commitment is blinded. Included with the commitment are two openings used to verify the commitment. You intercept a transaction by observing the shielded pool. Armed with the blinded commitment and two openings from the intercepted transaction as well as the public data (the trusted setup for the KZG commitment scheme, a list of all one-thousand account addresses participating in the pool, and the two random challenges used to compute the openings) can you deanonymize the recipient address?

## Blinding Scheme
The 256-bit recipient address is split into a vector of 32 bytes, and each byte (as a BLS12-381 scalar) becomes a coefficient of a degree-31 polynomial P. There is an evaluation domain H = {1, ω, ω^2, ..., ω^(n-1)} with $ω^n = 1$ and a vanishing polynomial Z_H(x) = x^n -1 which evaluates to 0 on each element of the evaluation domain. The sender of the transaction chooses two secret blinding factors b_0 and b_1 and computes the blinded polynomial Q(x) = P(x) + (b_0 + b_1x) • Z_H(x).

## Polynomial Commitment
The KZG-like polynomial commitment scheme uses a public trusted setup S={g, g•s, ..., g•s^(n+1)}$ where g is the generator point of the BLS12-381 elliptic curve and s is a secret scalar. For the polynomial Q(x) = c_0 + c_1x+c_2x^2 + ... + c_(n+1)x^(n+1) the commitment com(Q) = c_0•g + c_1•g•s + c_2•g•s^2 + ... + c_(n+1)•g•s^(n+1).

## Openings
Openings of the polynomial are required in order to verify the polynomial commitment. These are simple evaluations of the polynomial at random challenges which are public. For instance, for challenges z_1 and z_2, the openings are Q(z_1) and Q(z_2). 


thread 'main' panicked at src/bin/verify-hidden-in-plain-sight.rs:59:5:
assertion `left == right` failed
  left: GroupAffine { x: Fp384(BigInteger384([0, 0, 0, 0, 0, 0])), y: Fp384(BigInteger384([8505329371266088957, 17002214543764226050, 6865905132761471162, 8632934651105793861, 6631298214892334189, 1582556514881692819])), infinity: true }
 right: GroupAffine { x: Fp384(BigInteger384([3352324664306673969, 3713628578161648899, 1878490628203132913, 3466885105729237597, 9132614200361243635, 1401601394934476803])), y: Fp384(BigInteger384([16570623658658115686, 10300613295629417811, 2383692348039288689, 12921730385411258416, 18039840282952916614, 602371133612781370])), infinity: false }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
➜  zkhack-hidden-in-plain-sight git:(main) ✗ 
```

One thing I can notice here for sure, the puzzle creator has madd clear that it is 
"KZG-like"

My initial thoughts on this 

- 1000 accounts -> Seems rather small

- We are given an SRS (Structured reference String).

- two random challenges.....Why are we given 2 random challenges?


The hint of the puzzle is "Sometimes you can say too much."

Which means they gave us more information than required? or they implemented something twice?

All these Questions will be ANSWERED ON THE SUNDAY LIVE WWE CHAMPIONS. 😂


## KZG Commitment
Imagine I have a secret polynomial $f(x)$, and I want to convince you of two things:
 
1. I'm **committed** to this polynomial — I can't change it later to cheat.
2. $f(z) = y$ for some point $z$ you choose — without revealing the entire polynomial.
 
This is the **polynomial commitment problem**. A naive solution would be to just hand you $f(x)$ directly. But that leaks everything. A hash of the coefficients would let you verify commitment, but you can't use it to check evaluations.
 
KZG (Kate-Zaverucha-Goldberg, 2010) solves both properties elegantly using elliptic curve pairings.


I would assume that you know
- [Elliptic curves](https://letmegooglethat.com/?q=elliptic+curves)
- [Bilinear Pairings](https://letmegooglethat.com/?q=Bilinear+Pairings)

### The core protocol
Let $f(x) = \sum_{i=0}^{d} c_i x^i$ be a degree-$d$ polynomial over a field $\mathbb{F}_p$.
 
### Step 1: Commit
 
The commitment is just:
 
$$C = [f(\tau)] = \sum_{i=0}^{d} c_i \cdot [\tau^i]$$
 
This is a single group element — no matter the degree of $f$. It binds you to $f$ without revealing any coefficients.
 
### Step 2: Open at a Point
 
Suppose you want to prove $f(z) = y$ for some public point $z$.
 
Consider the **quotient polynomial**:
 
$$q(x) = \frac{f(x) - f(z)}{x - z}$$
 
This $q(x)$ exists as a polynomial (with no remainder) **if and only if** $f(z) = y$. This is the factor theorem — $(x - z) \mid (f(x) - y)$ iff $f(z) = y$.
 
The **proof** is just a commitment to $q$:
 
$$\pi = [q(\tau)]$$
 
Again, a single group element.
 
### Step 3: Verify
 
The verifier checks using a pairing equation:
 
$$e(C - [y], [1]) = e(\pi, [\tau - z])$$
 
Let's unpack why this works. Substituting the definitions:
 
$$e([f(\tau)] - [y], [1]) = e([q(\tau)], [\tau - z])$$
 
$$e([f(\tau) - y], [1]) = e([q(\tau)(\tau - z)], [1])$$
 
By bilinearity, both sides equal $e(G_1, G_2)^{f(\tau) - y}$ and $e(G_1, G_2)^{q(\tau)(\tau-z)}$ respectively. Since $q(\tau)(\tau - z) = f(\tau) - y$ by construction, the check passes.


### Why KZG
| Property | KZG | Hash-based commitments |
|---|---|---|
| Commitment size | $O(1)$ — one group element | $O(d)$ |
| Proof size | $O(1)$ — one group element | $O(d)$ or $O(\log d)$ |
| Verification time | $O(1)$ — two pairings | $O(d)$ |
| Requires trusted setup | Yes | No |
| Post-quantum secure | No | Potentially |


The **constant-size proofs** are the killer feature. In systems like Plonk, Groth16, and many others, the entire SNARK proof bottlenecks to a handful of KZG opening proofs.
 

If you wanna learn which commitment scheme is the best read this AMAZING blog by [ZKsecurity](https://blog.zksecurity.xyz/posts/pcs-survey/)

## The Exploit

The function implementation in the Puzzle is as follow

```rust
fn generate_challenge() -> (Vec<G1Affine>, Vec<Vec<Fr>>, Fr, Fr, G1Affine, Fr, Fr) {
    use rand::Rng;
    let mut rng = rand::thread_rng();//Make the thread

    let number_of_accts = 1000usize;// 1000 accounts
    let accts = generate_accts(number_of_accts);//Creates 1000 accounts 
    let target_acct_index = rng.gen_range(0..number_of_accts);//Gets a random account based on this
    let target_acct = &accts[target_acct_index];// selects that account

    let domain: GeneralEvaluationDomain<Fr> =
        GeneralEvaluationDomain::new(number_of_accts + 2).unwrap();// Constructing the FFT domain, Why 2?
    let setup = generate_kzg_setup(domain.size());// We get the SRS.

    let target_acct_poly = DensePolynomial::from_coefficients_vec(domain.ifft(&target_acct));//[1,2,3] -> 1+2x+3x^2....
    let blinding_poly =
        DensePolynomial::from_coefficients_vec(vec![Fr::rand(&mut rng), Fr::rand(&mut rng)]);//Creates a random polynomial.
    let blinded_acct_poly = target_acct_poly + blinding_poly.mul_by_vanishing_poly(domain);// Creates the Blinded polynomial

    let commitment: G1Affine = kzg_commit(&blinded_acct_poly, &setup);// Calls the KZG commit function

    let challenge_1 = Fr::rand(&mut rng);// A random challenge
    let challenge_2 = Fr::rand(&mut rng);// One more random challenge

    let opening_1 = blinded_acct_poly.evaluate(&challenge_1);// First openning
    let opening_2 = blinded_acct_poly.evaluate(&challenge_2);// Second Opening

    (
        setup,
        accts,
        challenge_1,
        challenge_2,
        commitment,
        opening_1,
        opening_2,
    )
}
```

So what is the problem here?



### Attack Strategy
You have 1000 candidate addresses and need to identify which one was committed to. The blinding only introduces 2 unknowns (b_0, b_1), and you have exactly 2 openings. This is a brute-force over 1000 candidates with a linear system check.
We have 1000 accounts, each of which is 256 bit or 32 bytes long, Each byte is a Scalar coefficient, which means we can get a polynomial coefficients of degree 31 (256/8=32)

By that we can surely get $P_i(x)$

Step 2
We the Openings for 2 of the random values.
$$
Q(z_1) = P_i(z_1) + (b_0 + b_1·z_1) · Z_H(z_1)
$$
$$
Q(z_2) = P_i(z_2) + (b_0 + b_1·z_2) · Z_H(z_2)
$$
If we somehow arrange these 

like:

$$
α := (Q(z_1) - P_i(z_1)) / Z_H(z_1)  =  b_0 + b_1·z_1
$$
$$
β := (Q(z_2) - P_i(z_2)) / Z_H(z_2)  =  b_0 + b_1·z_2
$$
We have 2 variables and 2 equations, we can easily solve this.

Once we have B_0 and B_1, we can reconstruct the expected commitment:

All three components are computable from public data:

com(P_i) — from the trusted setup and coefficients of P_i
com(Z_H) — since Z_H(x) = x^n - 1, this is g·s^n - g from the setup
com(x·Z_H) — this is g·s^(n+1) - g·s

If this equals the intercepted com(Q), you found the recipie


## Why the Binding Failed?

The blinding $(b_0 + b_1·x)·Z_H(x)$ is meant to make &com(Q)$ unlinkable to any specific P. But:

- The two opening leaked the exact information needed to solve both blinding factors

- Once you fix a candidate P_i

And since Address space only had 1000 entries, we can brute force for all of them


If there were intended for verfiying the commitment - here they become the decryption oracle

And since Address space only had 1000 entries, we can brute force for all of them


It would look something like this in the end

