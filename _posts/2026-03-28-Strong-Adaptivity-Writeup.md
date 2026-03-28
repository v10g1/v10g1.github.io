---
title: "Strong Adaptivity Write-up"
date: 2026-03-28 00:00:00 +0530
categories: [ZK, Writeups]
tags: [zk, zkHack-Puzzles,writeups,solutions]
math: true
image: /assets/StrongAdaptivity.jpg
---
This is the brief discussion on the solution of Strong Adaptivity by ZkHack.dev


This is a weak fiat shamir bug which I have covered in my blog post right [here](https://v10g1.github.io/posts/Fiat-Shamir/)

So I won't be explaining the bug again However let's discuss it.

# The Exploit
The Fiat Shamir Challenge was implemented as the follwing 


```rust
let challenge = b2s_hash_to_field(&(*ck, commitment));
```

Let look at `CommitKey` and `ProofCommitment`

```
pub struct CommitKey {
    pub message_generator: GAffine,
    pub hiding_generator: GAffine,
}
pub struct ProofCommitment {
    pub comm_rho: GAffine,
    pub comm_tau: GAffine,
}
```

It does not Include the $C_1$ and $C_2$ Which is actually one of the important part of implementing the Fiat Shamir

The challenge doesn't bind to the statement being proved. This is a real soundness issue, but it alone doesn't let you produce openings to different messages — it enables proof malleability/reuse across instances.


Yup that's it.

Thank you for reading this 👋👋





