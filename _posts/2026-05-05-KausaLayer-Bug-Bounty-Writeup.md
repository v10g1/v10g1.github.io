---
title: "Kausalayer Bug Bounty Writeup"
date: 2026-05-05 00:00:00 +0530
categories: [ZK, Bug-Bounties]
tags: [zk,writeup,Bug-Bounty]
math: true
image: /assets/Catjo.png
---
# Introduction
Right after started learning about Zero knowledge proofs and cryptography. I realised that if I want to be the best in the this space I need to research on a real code base, a real production codebase which is being developed by the experience devs.

> Note -> After I submitted the reports, the fixes were done on the same day and were deployed. 


# About the Project
Resolution infrastructure for prediction markets on Solana. Uses zkTLS proof verification (Reclaim Protocol) and ZK ownership claims (Groth16) for trustless market resolution with bettor privacy.

**What it tries to solve?**
Prediction markets need a way to determine outcomes. Current solutions rely on human voting (UMA/optimistic oracles), which is slow and vulnerable to manipulation. KRN replaces human voters with cryptographic verification:

zkTLS Proofs — Bots fetch outcomes from data sources (Coinbase, ESPN, etc.) and generate zkTLS proofs that the data is authentic. Multiple sources required for consensus.
On-Chain Verification — Program verifies proof signatures from Reclaim Protocol attestors using secp256k1 recovery + keccak256.
Private Claims — Winners prove bet ownership with Groth16 ZK proofs (Poseidon commitments + nullifiers) and claim to a fresh address. No link between betting and claiming address.

What I focused during my hunt on this protocol:

Ownership (Groth16 BN254): Circuit with 1076 constraints. Proves: Poseidon commitment reconstruction, Merkle inclusion, nullifier correctness. Bettor claims to fresh address without revealing betting identity.
Along with that, I also focused on the the commitment On-chain

During this Hunt, I found 5 Vulnerbilities, which were submitted  to the Dev team

# Findings
During this hunt, I found 5 vulnerabilities, which were submitted to the dev team.

| S.No | Title                                                                 | Severity | Response      |
|------|----------------------------------------------------------------------|----------|---------------|
| 1    | Unrestricted commitment_root overwrites allows fake satisfiability   | Critical | Fixed         |
| 2    | Unconstrained direction input allows attacker to forge Merkle roots  | Critical | Fixed         |
| 3    | Market can be drained with Commitment Decoupling                     | Critical | Fixed         |
| 4    | redacted                                                            | High     | Acknowledged  |
| 5    | redacted                                                            | Medium | Fixed         |

# [Critical] Unrestricted commitment_root overwrites allows fake satisfiability
**File** -> [Redacted]
## Description
The entire privacy model of KRN depends on this flow:
1. Bettor places a bet, submitting a Posiedon hash: 
    Which is done in the ownership.circuit as
    ```rust
    Posiedon(market_id,outcome,amount,secret_nonce,pubkey)
    ```
2. These commitments are collected in a merkle tree, producing a commitment_root

3. When claiming, the bettor submits a Groth16 proof.
But the problem arises when the we see the actul implementation of the `place_bet.rs`
```rust
pub fn handle_place_bet(
    ctx: Context<PlaceBet>,
    _market_id: [u8; 32],
    commitment_hash: [u8; 32],
    commitment_root: [u8; 32],
    side: u8,
    amount: u64,
) -> Result<()> {
    require!(
        side == MarketAccount::OUTCOME_NO || side == MarketAccount::OUTCOME_YES,
        KrnError::InvalidBetSide
    );
    require!(amount > 0, KrnError::ZeroBetAmount);


    let market = &mut ctx.accounts.market;
    require!(market.state == MarketState::Open, KrnError::InvalidMarketState);


    // Transfer SOL from bettor to market pool
    system_program::transfer(
        CpiContext::new(
            ctx.accounts.system_program.to_account_info(),
            system_program::Transfer {
                from: ctx.accounts.bettor.to_account_info(),
                to: ctx.accounts.market_pool.to_account_info(),
            },
        ),
        amount,
    )?;


    // Update market pools
    market.total_pool = market.total_pool.checked_add(amount).unwrap();
    if side == MarketAccount::OUTCOME_YES {
        market.yes_pool = market.yes_pool.checked_add(amount).unwrap();
    } else {
        market.no_pool = market.no_pool.checked_add(amount).unwrap();
    }


    // Record commitment
    let commitment = &mut ctx.accounts.commitment;
    commitment.market_id = market.market_id;
    commitment.commitment_hash = commitment_hash;
    commitment.side = side;
    commitment.amount = amount;
    commitment.claimed = false;
    commitment.bump = ctx.bumps.commitment;


    // Update market commitment root (client computes Merkle root off-chain)
    market.commitment_root = commitment_root;


    msg!("Bet placed: side={}, amount={}", side, amount);
    Ok(())
}
```
We can clearly see that commitment_root is user supplied argument that is written directly to chain with zero verification. the program never checks that `commitment_root` is actually derived from the `commitment_hash` or any other on-chain state

## Attack Path
1. Pick an arbitrary Private input for the ZK circuit
market_id      = <Anythin>

resolved_outcome = YES

secret_nonce   = anything

amount= anything

original_pubkey = anything

2. Compute the corresponding commitment nd merkle root offline

3. generate a valid Groth16 proof for these inputs using snarkJS.

4. Call Place_bet with;
commitment_hash -> the computed commitment

commitment_root -> the computed root

amount = any nonzero value (even 1 Lamport)

The market will now store the attacker's commitment_root on-chain.

## Team's reponse
Thanks for the report @v10g1 — you're right. The commitment_root was directly overwritable by the last bettor, allowing the exact attack vector you described.

Fixed in commit [REDACTED].

What changed:

commitment_root parameter removed from place_bet instruction entirely

Implemented an on-chain incremental Merkle tree (depth 10, max 1024 bets per market)

Program now computes the root internally from every commitment_hash leaf inserted via place_bet

No client can supply or override the root — it's derived purely from on-chain state
Added MerkleTreeFull error when tree capacity is reached.
The attack is no longer possible because the attacker cannot control commitment_root — it's the cumulative hash of all commitments, computed by the program itself.
Deployed and verified on devnet

---


# [Critical] Unconstrained direction input allows the attacker to forge Merkle roots
File -> [REDACTED]
# Summary
There is a direction signal used in the ownership.circom to conditionally change the path along the merkle tree. However, the direction is never considered to be a boolean value.

## Description
The work of circom circuit is to enforce the SnarkJS generation of Groth16 proof for the ownership of a claimed Bet. The claims are stored in the merkle tree provided by the users. Each user bet would be represented as a single leaf in the tree.
For this, the circuit expects a direction input, which is used as a selector s in the Mux1 component to conditional order the hashes for the merkle proof. Howeverm `direction` is never constrained to be a boolean value. Mux1 calculates the output via the constraint 
```
out <== c[0] - s * (c[0] - c[1])
```
Because the direction is unconstrained, it can take any value in the finite field.
## Attack Path
An Attacker can pick arbitrary commitment_root, a random sibling, and computes c[0] and c[1]. They then algebraically solve for the required direction:
```
direction = (c[0] - target_root) * inverse(c[0] - c[1])
```
By passing this non binary value as a private witness, the circuit successfully validates against the target root, allowing any user to fake ownership of a winning bet without being in the merkle tree

## Team's Response
Fixed in commit [REDACTED]

Added direction * (1 - direction) === 0 constraint in ownership.circom. Non-binary values are now rejected by the circuit.

Circuit re-compiled, new verification key generated and deployed to devnet.

