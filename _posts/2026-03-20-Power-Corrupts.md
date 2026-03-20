---
title: "Power Corrupts Writeup"
date: 2026-03-20 00:00:00 +0530
categories: [ZK, Writeups]
tags: [zk, zkHack-Puzzles,writeups,solutions]
math: true
---

NOTE -> this is a draft, not a complete blog


## Initial run

```
cargo run --release
```
the result was as follow
```bash
   Finished `release` profile [optimized] target(s) in 13.58s
     Running `target/release/blstest`

    ______ _   __  _   _            _
    |___  /| | / / | | | |          | |
       / / | |/ /  | |_| | __ _  ___| | __
      / /  |    \  |  _  |/ _` |/ __| |/ /
    ./ /___| |\  \ | | | | (_| | (__|   <
    \_____/\_| \_/ \_| |_/\__,_|\___|_|\_\
    

Bob has invented a new pairing-friendly elliptic curve, which he wanted to use with Groth16.
For that purpose, Bob has performed a trusted setup, which resulted in an SRS containting
a secret $\tau$ raised to high powers multiplied by a specific generator in both source groups. 
The exact parameters of the curve and part of the output of the setup are described in the 
document linked below.

Alice wants to recover $\tau$ and she noticed a few interesting details about the curve and
the setup. Specifically, she noticed that the sum $d$ of the highest power $d_1$ of $\tau$ in 
$\mathbb{G}_1$ portion of the SRS, meaning the SRS contains an element of the form 
$\tau^{d_1} G_1$ where $G_1$ is a generator of $\mathbb{G}_1$, and the highest power $d_2$ 
of $\tau$ in $\mathbb{G}_2$  divides $q-1$, where $q$ is the order of the groups. 

Additionally, she managed to perform a social engineering attack on Bob and extract the 
following information: if you express $\tau$ as $\tau = 2^{k_0 + k_1((q-1/d))} \mod r$, 
where $r$ is the order of the scalar field, $k_0$ is 51 bits and its fifteen most 
significant bits are 10111101110 (15854 in decimal). That is A < k0 < B where 
A = 1089478584172543 and B = 1089547303649280.

Alice then remembered the Cheon attack...

NOTE: for exponentiating $F_r$ elements, use the `pow_sp` and `pow_sp2` functions in
`utils.rs`.

The parameters of the curve and the setup are available at 
https://gist.github.com/kobigurk/352036cee6cb8e44ddf0e231ee9c3f9b


thread 'main' panicked at src/main.rs:55:5:
assertion `left == right` failed
  left: true
 right: false
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
but what the heck is a cheon attack?
## What is a cheon attack
In the puzzle we can clearly see that the they stated something like "cheon" attack, so let's first study what does that mean?

The Cheon attack is a specialized algorithm in cyrptography that weakens the hardness of Discrete logarithm problem under certain structural conditions

Normally, given:
a cyclic group $G$ of order $n$

a generator g, 

and $h = g^x$

recovering x is hard (this is the DLP)

The cheon attack exploits the cases where:

- The group order n has a special divisibility structure, and 
- you are given auxiliary information, typically something like:
$$
g^{x^d}
$$

for some divisor $d|(n-1)$ or $d|(n+1)$

**How does this work?**

if such a divisor d exists and is "nice", meaning not too large, cheon showed that you can solve the DLP faster than generic attacks

instead of:
$$O(\sqrt{n})$$
we can get it in 

$$O(\sqrt{n/d}+\sqrt{d})$$

so if: 
- d almost equal to root n

then complexity is $$O(n^{1/4})$$
which exponentially faster than standard attacks


You can read about it more [Here](https://ethresear.ch/t/cheons-attack-and-its-effect-on-the-security-of-big-trusted-setups/6692)
On the core it works by using of those algos which are used to reduce the discret log problems like the baby step giant step algo or the pollard rho algo 
when using the Pollard's rho algorithm we can eigther use a large memory or a constant memory version. for the part memory version, ie which requires saving around 1.25 times the elements of G, the expected number of evalutions. 

The Marlin Authors also noticed that if you are given $g,g^{\alpha}$ and $g^{\alpha^d}$ it is also possible to use the pairing to tranfer the problem.
## Elliptic curves

An elliptic curve is a figure which follows the eqaution 
$$
y^2= x^3+ax+b
$$
and also follows the non singularity.

But v10g, why did you tell us about this here?

in the challenge we are given a elliptic curve which are pairing friendly.

How does a curve become pairing friendly?

- Small Embedding degree ($\rho$-value)
The embedding degree k is the smallest integer such that $r|p^1-1$, where r is the prime order of the curve's subgroup and p is the field characteristic. for a random curve, k is astronmically large (exponential in log r), making the pairing computationally infeasible. Pairing friendly curves are engineered to have a small k- typically between 2 and 53
smaller K means that the Piring lands in a small extension field F (p^k) which amkes the miller loop tractable

- Large prime Order subgorund
Security requires that r be a large prime- at least 256 bits for 128 bit security. this prevents DLP attacks on G1 and G2

- rho value close to 1

- CM discriminant tractable

- twist availabiltiy

Lets read the statement again 

Alice wants to recover $\tau$ and she noticed a few interesting details about the curve and
the setup. Specifically, she noticed that the sum $d$ of the highest power $d_1$ of $\tau$ in 
$\mathbb{G}_1$ portion of the SRS, meaning the SRS contains an element of the form 
$\tau^{d_1} G_1$ where $G_1$ is a generator of $\mathbb{G}_1$, and the highest power $d_2$ 
of $\tau$ in $\mathbb{G}_2$  divides $q-1$, where $q$ is the order of the groups.

There are parameters given in the github gists

->
$p$ (size of the finite field over which the curve is defined) = 5739212180072054691886037078074598258234828766015465682035977006377650233269727206694786492328072119776769214299497

$q$ (size of the subgroup of the elliptic curve) = 1114157594638178892192613

$k$ (embedding degree) = 12

$d_1$ (highest power of $\tau$ in the $\mathbb{G}_1$ portion of the SRS) = 11726539

$d_2$ (highest power of $\tau$ in the $\mathbb{G}_2$ portion of the SRS) = 690320833

Curve equation = $E : y^2 = x^3 + 4$. This is a pairing friendly curve meaning that $E(F_p)$ contains a subgroup of size $q$ and $E(F_{p^k})$ contains a different subgroup of size $q$

$P \in \mathbb{G}_1$ (generator in $\mathbb{G}_1$) = (4366845981406663127346140105392043296067620632748305894915559567990751463871846461571751242076416842353760718219463, 2322936086289479068066490612801140015408721761579169972917817367391045591350600034547617432914931369293155123935728)

$\tau P \in \mathbb{G}_1$ =  (4098523714512767373909357151251054769972251070043521443432671846694698445310918888674563384159054413429407635337545, 5104199250567875649831070757916425527562742910121942289068977475282886267374972984853923971702383949149250750103817)

$\tau^{d_1} P \in \mathbb{G}_1$ =  (5666793736468521094298738103921693621508309431638529348089160865885867964805742107934338916926827890667749984768215, 3326251098281288448602352180414320622119338868949804322483594574847275370379159993188118130786632123868776051955196)

$Q \in \mathbb{G}_2$  (generator in $\mathbb{G}_2$) =  (2847190178490156899798643792842723617787968359868175140038826869144776012793105029391523604954249120667126821536281 + 1513797577242500304678874752065526230408447782356629533374984043360635354098197307045487457331199798062484459984831 ∗ u, 2398127858646538650279262747029238501121661957103909673770298065006753715123740323569605568913154172079135187452386 +
5444946257649901533268220138726124417824817651440748374257708320447300055543369665159277001725118567443194417165086 ∗ u)

$\tau^{d_2} Q \in \mathbb{G}_2$ =  (3536383419772898871062064633012296862124372086039789534814905834326827967479599778599887194095392165550880125330266 + 2315075704417849395497347082310199859284883937672695000597201154920791698799875018503579607990866594389648640170976 ∗ u, 58522461936731088989461032245338237080030815519467180488197672431529427745827070450637290003234818635632376291077 + 200313434320582884299950030908390796161004965251373896142196467499133624968316891420231529223691679020778320981956 ∗ u)


> mannnnn, the fuck is this?

We need to think how can we get the Tau from the setup parametere

## Bilinear pairings
A simple property in pairings:
$$
e(\tau P, Q) = e(P, Q)^{\tau}
$$

$$
e(\tau^{d_1} P, Q) = e(P, Q)^{\tau^{d_1}}
$$
Don't worry, I also don't know much about bilinear pairings, just remember this property

## The Attack
We will use th Cheon's attack in this challenge, we need a divisor such that 
$d|q-1$ or $d|q-l$ we found the this relation with the d1+d2 -> d, we can use this for the Cheon's attack.

How?

Let's see below:
Let's assume 2 is a generator of Fq
then every element can be written as $$2^k mod q $$

So there exists some $k \in \mathbb{Z_{q-1}}$

lets assume 

$\tau=2^k mod q $

since $d|(q-1)$, we can write:

$q-1=d.m$

So any K in Z can be written uniquely as:

$$k=k_0 +k_1.m$$

Now if we put that in 

$$\tau=2^{k_0+k_1.m}$$

which further can be written as 

$$\tau= 2^{k_0 + k1*\frac{q-1}{d}}$$

Now if you can recall the fermat's little theorem which states that

$$a^{p-1}=1(mod p)$$

where p is a prime

By that 
$$\tau^{q-1}= 1modq$$

We can further write it as 

$\tau^d=2^{dk_0}$

We can write 
$$
\left(g^{\tau^d}\right)^{(2^{-d})^{A' + i \cdot 2^{18}}} = g^{(2^d)^j}
$$


The rest will implemented in the test will attached below:

Notes: The POC done below is a POC from the official writeup, it involves the same  steps as i said.
```rust
use crate::utils::{bigInt_to_u128, pow_sp, pow_sp2};
use ark_bls12_cheon::{Bls12Cheon, Fq12, Fr, G1Projective as G1, G2Projective as G2};
use ark_ec::pairing::Pairing;
use ark_ff::{Field, PrimeField};
use std::collections::HashMap;
use std::ops::Mul;
use std::time::Instant;

pub fn attack(P: G1, tau_P: G1, tau_d1_P: G1, Q: G2, tau_d2_Q: G2) -> i128 {
    let d1 = 11726539;
    let d2 = 690320833;
    let d: u64 = d1 + d2;
    let q = 1114157594638178892192613u128;
    let q1_d = (q - 1) / u128::from(d);

    // Translate the problem into the target group
    // (".0" needs to be used here to extract the Fq12 from the PairingOutput type)
    let g: Fq12 = Bls12Cheon::pairing(P, Q).0;
    let g_tau: Fq12 = Bls12Cheon::pairing(tau_P, Q).0; // = g^tau
    let g_tau_d: Fq12 = Bls12Cheon::pairing(tau_d1_P, tau_d2_Q).0; // = g^(tau^d)

    // Find k0 such that tau^d = (2^d)^k0 mod q
    let two_pow_d = pow_sp(Fr::from(2u64), d.into(), 31);
    let g_2_d = g_tau_d.pow(two_pow_d.into_bigint());
    println!("2^d = {}", two_pow_d);

    // We know the high bits of k0 and the subject provides A and B such as A < k0 < B
    let k0_base: u128 = 0x3dee000000000;
    assert_eq!(1089478584172543, k0_base - 1); // A
    assert_eq!(1089547303649280, k0_base + (1 << 36)); // B

    // Shanks's Baby-Step Giant-Step algorithm: with 36 bits to find, split in two
    // To find i, j such that g^((2^d)^k0) = g^(tau^d) and k0 = k0_base + i * 2^20 + j,
    // Compute g^((2^d)^j) and for each i, search (g^(tau^d))^(2^(-d*k0_A))^((2^(-d*2^20))^i)
    let mut k0 = 0u128;
    let bsgs_split = 1u64 << 20;
    println!("Baby step for k0...");
    let start = Instant::now();
    let baby_steps = pow_sp2(g, two_pow_d, bsgs_split);
    println!(
        "Baby step for k0 done in {} seconds",
        start.elapsed().as_secs()
    );

    let mut giant = g_tau_d.pow(pow_sp(two_pow_d.inverse().unwrap(), k0_base, 51).into_bigint());
    let giant_inc = pow_sp(two_pow_d.inverse().unwrap(), bsgs_split.into(), 64).into_bigint();
    println!("Giant step for k0...");
    let start = Instant::now();
    for i in 0..q1_d / u128::from(bsgs_split) {
        if let Some(j) = baby_steps.get(&giant) {
            let duration = start.elapsed().as_secs();
            k0 = k0_base + i * u128::from(bsgs_split) + u128::from(*j);
            println!(
                "Found i = {}, j = {}, k0 = {} in {} seconds",
                i, j, k0, duration
            );
            break;
        }
        giant = giant.pow(giant_inc);
    }
    assert_eq!(k0, 1089539821761426);

    // Ensure that tau^d == 2^(d*k0) modulo q by checking the pairing
    assert_eq!(g_tau_d, g.pow(pow_sp(two_pow_d, k0, 51).into_bigint()));

    // Compute 2^((q - 1) / d)
    let eta = pow_sp(Fr::from(2u64), q1_d, 51);

    // Shanks's Baby-Step Giant-Step algorithm: with 30 bits to find
    // Find i, j such that (g^tau)^(2^(-k0)) = g^(eta^k1) with k1 = i * 2^17 + j
    // Compute g^(eta^j) and for each i, search (g^tau)^(2^(-k0))((eta^(-2^17))^i)
    let mut k1 = 0u128;
    let bsgs_split = 1u64 << 16;
    println!("Baby step for k1...");
    let start = Instant::now();
    let baby_steps = pow_sp2(g, eta, bsgs_split);
    println!(
        "Baby step for k1 done in {} seconds",
        start.elapsed().as_secs()
    );

    let mut giant = g_tau.pow(pow_sp(Fr::from(2u64).inverse().unwrap(), k0, 51).into_bigint());
    let giant_inc = pow_sp(eta.inverse().unwrap(), bsgs_split.into(), 64).into_bigint();
    println!("Giant step for k1...");
    let start = Instant::now();
    for i in 0..d / bsgs_split {
        if let Some(j) = baby_steps.get(&giant) {
            let duration = start.elapsed().as_secs();
            k1 = u128::from(i * bsgs_split + *j);
            println!(
                "Found i = {}, j = {}, k1 = {} in {} seconds",
                i, j, k1, duration
            );
            break;
        }
        giant = giant.pow(giant_inc);
    }
    assert_eq!(k1, 690599720);

    let tau = pow_sp(Fr::from(2u64), k0 + k1 * q1_d, 80);
    println!("Found tau = {}", tau);
    assert_eq!(tau, Fr::from(284865198031253921498207u128));
    assert_eq!(P.mul(tau), tau_P);

    return bigInt_to_u128(tau.into_bigint()) as i128;
}
```
By this way we can have the value of Tau.
Good Day have a Nice ONEEEEE!!!!!