---
title: "Zero Sum Game Writeup"
date: 2026-03-18 00:00:00 +0530
categories: [ZK, Writeups]
tags: [zk, zkHack-Puzzles,writeups,solutions]
math: true
---
> WARNING -> Contains spoilers for the challenge, Please don't read it if you want to attempt it in the future

[ZkHack](https://zkhack.dev/) has a bunch of puzzles for the people trying to get into Zero knowledge auditing or the space in general, So I asked my friend Chatgpt which one would be good for the first one and He said this one

## Sum check protocol

On the core this protocol uses Sum check protocol, which is the base for this challenge first lets understand What really is this "Sum check protcol"

For the reference am reading the book by Justin Thaler, You can also read it from [here](https://people.cs.georgetown.edu/jthaler/ProofsArgsAndZK.pdf)

Let's consider a general system where there are two actors $P$ or the Prover and $V$ or the verifier, we would not wanna assume that prover is a trusted entity, it may or may not be one.

In cryptographical proof systems there is a term something known as "commiting" and this is no different than your Girlfriend not commiting to you (OUCH!). 

Commiting an equation qould simply mean that the entity cannot change it anymore for the current round. Here a prover will commit something which he wants it to be verified.

At the start of the sum check protocol, the prover sends a value $C_1$ claimed to equal the true answer. The sume check protocol proceeds in V rounds, one for each varaibles in the sent eqaution g. At the start of the first round, the prover send apolynimial $g_1(X_1)$ claimed to equal the polynomial $S_1(X_1)$

which can be defined as the follow
$$
s_1(X_1) := \sum_{(x_2, \ldots, x_v) \in \{0,1\}^{v-1}} g(X_1, x_2, \ldots, x_v)
$$

we need to keep the track of the follow 

$$H = s_1(0) + s_1(1)$$

The verifier will need the check the claimed $C_1$ it should be eqaul, if not the protocol revert.

By revert I simply mean that the proof is reject by the verifier.

What's next?

Now we have the polynomial and we have checked for the current claimed value for the boolean hypercube. The idea of the sum-check protocol is that $V$ is probabilistically check this equality of polynomials hold by picking a random field element $r_1 \in \mathbb{F}$ and confirming that

$$g_1(r_1)=s_1(r_1)$$

Now if the equality is held then this would meanthen this hold for all $r_1 \in \mathbb{F}$

Well technically, there is a small probability that it will not sound enough, according to schwartz zippel lemma we have a probablity that it will be sound for atleast 1- deg(g)/$|\mathbb{F}|$ 

Round two:

The protocol thus proceeds in this recursive manner, with one round per recursive call. This means that in round j, variable $X_j$ gets bound to a random field element $r_j$ chosen by the verifier. This proceeds until round v, in which the prover is forced to send a polynomial again like

$$s_v := g(r_1,....,r_{v-1},X_v)$$

When the verifier goes to check that $g_v(r_v)=s_v(r_v)$ , there is no need for further recursion: since the verifier is given oracle access to g, V can evaluate $s_v(r_v)=g(r_1,...r_v)$ with the single oracle query


Now at this point you would ask that how would the equation would look at a generalized form?

$$
g_j(X_j) = \sum_{(x_{j+1}, \ldots, x_v) \in \{0,1\}^{v-j}} 
g(r_1, \ldots, r_{j-1}, X_j, x_{j+1}, \ldots, x_v)
$$

If this felt a little too much, please consider reading the thaler book.


Now lets try to cover, the Univariate Sum check Protocol

The univariate sum check protocol is an interactive proof system that allows a prover to convince a verifier that the sum of a univariate polynomial over a specific domain equals some claimed value — without the verifier having to evaluate every point themselves.

but v10g, how does this work

Let $f:\mathbb{F}->\mathbb{F}$ be a univariate polynomial of degree d over the field $\mathbb{F}$. Let $H= {h_1,h_1,.....,h_{n-1}}$ subset of $\mathbb{F}$ with $|H|=n$

The prover wants to convince the verifier that 

$\sum_{(a \in H)}f(a) = s$

for some claimed sum s. The verifier cannot afford to evaluate F at all n points- that's the entire cost they are trying to avoid

Unlike the multilinear sum-check, the univariate version works by encoding the sum constraints directly into a polynomial identity, The core idea:

 Step 1- Commit

The prover commits to a polynomial $f$(typically via a polynomial commitment scheme like KZG or FRI). The verifier receives a commitment lets say com(f)

step 2- The algebraic encoding of the sum 

the key identity is:

$$
\sum_{a \in H} f(a) = n \cdot [x^0]\!\left( \frac{f(x)}{Z_H(x)} \right) \cdot Z_H'(x)
$$
and this the vanishing polynomial fro $Z_H(x) = \prod_{a \in H} (x - a)$

but v10g, what the heck is a vanishing polynomial now bro?

A vanishing polynomial over a set 𝐻 is exactly what the name suggests: a polynomial that evaluates to zero on every element of 𝐻

You are not adding $Z_H(x)$ because of its value, we are adding it because of its zero divisibility behavious.

-> If we explicitly check all the points, its gonna cost us more than US's defence budget.

Instead, we enforce one statement
$f(x)=Z_H(x) \dot q(x)$

It check all the constraints automatically.

It is a gatekeeper polynomial.

*Lets try dry running the protocol*
Let  
$$
H = \{\omega^0, \omega^1, \ldots, \omega^{n-1}\}
$$  
be the set of $n$-th roots of unity in $\mathbb{F}$.

The vanishing polynomial over $H$ is:
$$
Z_H(x) = x^n - 1
$$

---

 Prover's Claim

$$
\sum_{i=0}^{n-1} f(\omega^i) = s
$$

---

 Step 1 — Accumulator Polynomial

Define $g : H \to \mathbb{F}$ such that:

$$
g(\omega^0) = f(\omega^0)
$$

$$
g(\omega^i) = g(\omega^{i-1}) + f(\omega^i), \quad \text{for } i = 1, \ldots, n-1
$$

Thus:

$$
g(\omega^{n-1}) = s
$$

The prover interpolates $g(x)$ as a polynomial of degree $< n$.

---

 Step 2 — Encode the Recurrence

$$
g(\omega x) - g(x) - f(x) = 0 \quad \forall x \in H
$$

Therefore, there exists a quotient polynomial $q_1(x)$ such that:

$$
g(\omega x) - g(x) - f(x) = q_1(x)\, Z_H(x)
$$

---

Step 3 — Boundary Conditions

 Initial condition

$$
g(\omega^0) - f(\omega^0) = 0
$$

Encoded as:

$$
(g(x) - f(x)) \cdot L_0(x) = 0 \pmod{Z_H(x)}
$$

Terminal condition

$$
g(\omega^{n-1}) - s = 0
$$

Encoded as:

$$
(g(x) - s) \cdot L_{n-1}(x) = 0 \pmod{Z_H(x)}
$$

---

Lagrange Basis Polynomials

$$
L_i(x) = \prod_{j \ne i} \frac{x - \omega^j}{\omega^i - \omega^j}
$$

Equivalent form:

$$
L_i(x) = \frac{Z_H(x)}{(x - \omega^i)\, Z_H'(\omega^i)}
$$

---

Quotient Forms

There exist polynomials $q_2(x), q_3(x)$ such that:

$$
(g(x) - f(x)) \cdot L_0(x) = q_2(x)\, Z_H(x)
$$

$$
(g(x) - s) \cdot L_{n-1}(x) = q_3(x)\, Z_H(x)
$$

---

Step 4 — Random Linear Combination

The verifier samples a random challenge $\alpha$.

The prover defines:

$$
T(x) = q_1(x) + \alpha\, q_2(x) + \alpha^2\, q_3(x)
$$

The verifier checks:

$$
\begin{aligned}
&\left[g(\omega x) - g(x) - f(x)\right] \\
&\quad + \alpha \left[(g(x) - f(x)) \cdot L_0(x)\right] \\
&\quad + \alpha^2 \left[(g(x) - s) \cdot L_{n-1}(x)\right] \\
&= T(x) \cdot Z_H(x)
\end{aligned}
$$

---

Step 5 — Evaluation at a Random Point

The verifier samples:

$$
z \xleftarrow{\$} \mathbb{F} \setminus H
$$

The prover opens:

$$
f(z), \quad g(z), \quad g(\omega z), \quad T(z)
$$

The verifier checks:

$$
\begin{aligned}
&\underbrace{g(\omega z) - g(z) - f(z)}_{\text{recurrence}} \\
&\quad + \alpha \underbrace{(g(z) - f(z)) \cdot L_0(z)}_{\text{init}} \\
&\quad + \alpha^2 \underbrace{(g(z) - s) \cdot L_{n-1}(z)}_{\text{terminal}} \\
&\stackrel{?}{=} T(z) \cdot Z_H(z)
\end{aligned}
$$

---

Verifier Computation

The verifier can compute locally:

$$
L_0(z), \quad L_{n-1}(z), \quad Z_H(z) = z^n - 1
$$

in $O(\log n)$ time.


## The Vulnerability
In the challenge, we need to make a proof which will convice the verifier somehow that the $\sum_{(a \in H)}f(a)=0$, but how can we even do that?

This is made possible due to the fact that Bob has made the mechanism which made the sum of the Polynomial F secret, which is used to make the protocol anonymous, it allows the malicious prover to send the a secret polynomial s, and also allowing the opening of the same. This sum can be used to exploit the protocol

In the honest protocol

the goal is to prove $\sum_{(a \in H)}f(a)=\gamma$ without the verifier ever evalutating all the N points, the core trick is that we use a vanishing polynomial
$Z_H(x)=x^n-1$

or more like $$f(x)=h(x)Z_H(r)+r(x)$$

since $Z_H()$ vanishes on H we dont actually need to check for all the points, it automatically constraints the sum check to check, for any x not in H the sum will not be eqaul to zero.

Since $Z_H$ vanishes on  $H$, the first term contributed nothing ot the sum

then key fact is 

$$
\sum_{h \in H} h^k =0
$$

this holds become h is a multiplicative subgroup- its elements are nth roots of unity $\omega$ which are symmeterically distributed and calcel each other out!!!

The only exception is K=0, where the sum is just n, we can easily exclude it

The verifier then checks this equation at a random point z via Schwartz zipple. The actual protocol adds a masking polynomial, making the check:

$$
f(x) + s(x) = h(x)\cdot Z_H(x) + x \cdot g(x) + \frac{\gamma}{N}
$$

However, the problem arises when the verifier tries to check the eqaubtion above at a random point

But there is no check on $s(x)$ whatsoever, the prover can set it to anything he wants

## The Exploit

Say the real sum is gamma_real != 0 but your want to claim gamma_fake =0

One approach that we can check is that we set the $s(x)$ to a randomized value to hide the constant term. Pick radnom coef $t_0,t_1....t_{n-2}$ and define the function $s(x)$ as the sum of the $t_1x^i$ over i= 0 to n-2

and final eqaution SHOULD cancel out the terms, leaving the original eqaution, the check will pass


Thank you for reading this Puzzle report, Follow for more

Cheers! 🍻