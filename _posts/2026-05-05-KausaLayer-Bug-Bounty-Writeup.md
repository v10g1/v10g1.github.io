---
title: "Kausalayer Bug Bounty Writeup"
date: 2026-05-05 00:00:00 +0530
categories: [ZK, Bug-Bounties]
tags: [zk, writeup, Bug-Bounty]
math: true

---

# Introduction

Right after I started learning about zero-knowledge proofs and cryptography, I realized that if I want to be the best in this space, I need to research a real codebase — a real production codebase being developed by experienced devs.

> **Note:** After I submitted the reports, the fixes were done on the same day and were deployed.

# About the Project

Resolution infrastructure for prediction markets on Solana. Uses zkTLS proof verification (Reclaim Protocol) and ZK ownership claims (Groth16) for trustless market resolution with bettor privacy.

**What it tries to solve?**

Prediction markets need a way to determine outcomes. Current solutions rely on human voting (UMA/optimistic oracles), which is slow and vulnerable to manipulation. KRN replaces human voters with cryptographic verification:

- **zkTLS Proofs** — Bots fetch outcomes from data sources (Coinbase, ESPN, etc.) and generate zkTLS proofs that the data is authentic. Multiple sources are required for consensus.
- **On-Chain Verification** — The program verifies proof signatures from Reclaim Protocol attestors using secp256k1 recovery + keccak256.
- **Private Claims** — Winners prove bet ownership with Groth16 ZK proofs (Poseidon commitments + nullifiers) and claim to a fresh address. No link between the betting and claiming address.

What I focused on during my hunt on this protocol:

- **Ownership (Groth16 BN254):** Circuit with 1076 constraints. Proves: Poseidon commitment reconstruction, Merkle inclusion, nullifier correctness. Bettor claims to a fresh address without revealing their betting identity.
- Along with that, I also focused on the commitment on-chain.

During this hunt, I found 5 vulnerabilities, which were submitted to the dev team.

# Findings

| S.No | Title                                                               | Severity | Response     |
|------|---------------------------------------------------------------------|----------|--------------|
| 1    | Unrestricted commitment_root overwrites allows fake satisfiability  | Critical | Fixed        |
| 2    | Unconstrained direction input allows attacker to forge Merkle roots | Critical | Fixed        |
| 3    | Market can be drained with Commitment Decoupling                    | Critical | Fixed        |
| 4    | redacted                                                            | High     | Acknowledged |
| 5    | redacted                                                            | Medium   | Fixed        |

---

# [Critical] Unrestricted commitment_root overwrites allows fake satisfiability

**File** → [Redacted]

## Description

The entire privacy model of KRN depends on this flow:

1. Bettor places a bet, submitting a Poseidon hash, which is done in `ownership.circom` as:
```rust
    Poseidon(market_id, outcome, amount, secret_nonce, pubkey)
```
2. These commitments are collected in a Merkle tree, producing a `commitment_root`.
3. When claiming, the bettor submits a Groth16 proof.

But the problem arises when we see the actual implementation of `place_bet.rs`:

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

We can clearly see that `commitment_root` is a user-supplied argument that is written directly to chain with zero verification. The program never checks that `commitment_root` is actually derived from the `commitment_hash` or any other on-chain state.

## Attack Path

1. Pick an arbitrary private input for the ZK circuit:
   - `market_id` = \<anything\>
   - `resolved_outcome` = YES
   - `secret_nonce` = anything
   - `amount` = anything
   - `original_pubkey` = anything

2. Compute the corresponding commitment and Merkle root offline.

3. Generate a valid Groth16 proof for these inputs using SnarkJS.

4. Call `place_bet` with:
   - `commitment_hash` → the computed commitment
   - `commitment_root` → the computed root
   - `amount` = any nonzero value (even 1 Lamport)

The market will now store the attacker's `commitment_root` on-chain.

## Team's Response

> Thanks for the report @v10g1 — you're right. The `commitment_root` was directly overwritable by the last bettor, allowing the exact attack vector you described.
>
> Fixed in commit [REDACTED].
>
> **What changed:**
> - `commitment_root` parameter removed from `place_bet` instruction entirely.
> - Implemented an on-chain incremental Merkle tree (depth 10, max 1024 bets per market).
> - Program now computes the root internally from every `commitment_hash` leaf inserted via `place_bet`.
> - No client can supply or override the root — it's derived purely from on-chain state.
> - Added `MerkleTreeFull` error when tree capacity is reached.
>
> The attack is no longer possible because the attacker cannot control `commitment_root` — it's the cumulative hash of all commitments, computed by the program itself.
> Deployed and verified on devnet.

---

# [Critical] Unconstrained direction input allows the attacker to forge Merkle roots

**File** → [REDACTED]

## Summary

There is a direction signal used in `ownership.circom` to conditionally traverse the path along the Merkle tree. However, the direction is never constrained to be a boolean value.

## Description

The work of the circom circuit is to enforce the SnarkJS generation of a Groth16 proof for the ownership of a claimed bet. The claims are stored in a Merkle tree. Each user bet is represented as a single leaf in the tree.

For this, the circuit expects a `direction` input, which is used as a selector `s` in the `Mux1` component to conditionally order the hashes for the Merkle proof. However, `direction` is never constrained to be a boolean value. `Mux1` calculates the output via the constraint:
```
out <== c[0] - s* (c[0]-c[1])
```

Because the direction is unconstrained, it can take any value in the finite field allowed by Circom

## Attack Path
An Attacker picks an arbitrary `commitment_root` , a random sibling, and computes `c[0]` and `c[1]`. They then can algebrarically solve for the required direction.

By passing this non-binary value as a private witness, the circuit successfully validates against the target root, allowing any user to fake ownership of a winning bet without being in the merkle tree.

## Team Response:
> Fixed in commit [Redacted].  
> Added direction * (1 - direction) === 0 constraint in ownership.circom. Non-binary values are now rejected by the circuit.  
> Circuit re-compiled, new verification key generated and deployed to devnet.

# [Critical] Market can be drain with Commitment Decoupling
**File** -> [REDACTED]

## Description
The payout is calculated based on the amount stored in the commitment account provided by the caller. However, the ownershop_proof only proves the bet amount is associated with some valid leaf in the tree. The ZK proof does not prove the bet amount 

```rust
pub fn handle_claim_winning(
    ctx: Context<ClaimWinning>,
    _market_id: [u8; 32],
    _nullifier: [u8; 32],
    ownership_proof: OwnershipProofData,
) -> Result<()> {
    let market = &ctx.accounts.market;
    require!(
        market.state == MarketState::Resolved,
        KrnError::MarketNotResolved
    );


    // Validate proof public inputs match on-chain state
    validate_ownership_public_inputs(
        &ownership_proof,
        &market.market_id,
        market.outcome,
        &_nullifier,
        &market.commitment_root,
    )?;


    // Verify Groth16 ownership proof on-chain
    verify_ownership_proof(&ownership_proof)?;


    // Record nullifier to prevent double-claiming
    let nullifier_account = &mut ctx.accounts.nullifier_account;
    nullifier_account.market_id = market.market_id;
    nullifier_account.nullifier = _nullifier;
    nullifier_account.claimed_at = Clock::get()?.unix_timestamp;
    nullifier_account.bump = ctx.bumps.nullifier_account;


    // Calculate payout based on commitment amount and pool ratios
    let commitment = &ctx.accounts.commitment;
    require!(
        commitment.side == market.outcome,
        KrnError::OutcomeMismatch
    );


    let winning_pool = if market.outcome == MarketAccount::OUTCOME_YES {
        market.yes_pool
    } else {
        market.no_pool
    };


    // Payout = (bet_amount / winning_pool) * total_pool
    let payout = (commitment.amount as u128)
        .checked_mul(market.total_pool as u128)
        .unwrap()
        .checked_div(winning_pool as u128)
        .unwrap() as u64;


    // Transfer from pool to recipient
    let market_id = market.market_id;
    let bump = ctx.bumps.market_pool;
    let signer_seeds: &[&[&[u8]]] = &[&[b"pool", market_id.as_ref(), &[bump]]];


    let transfer_ix = anchor_lang::solana_program::system_instruction::transfer(
        &ctx.accounts.market_pool.key(),
        &ctx.accounts.recipient.key(),
        payout,
    );
    anchor_lang::solana_program::program::invoke_signed(
        &transfer_ix,
        &[
            ctx.accounts.market_pool.to_account_info(),
            ctx.accounts.recipient.to_account_info(),
            ctx.accounts.system_program.to_account_info(),
        ],
        signer_seeds,
    )?;


    msg!("Winning claimed: payout={}", payout);
    Ok(())
}
```
Here we can clearly see there is no verification that the funds that the user is trying to claim is associated with the caller or not.

## Attack Path
> Market already has more funds than user's amount.
1. Attack Places a massive bet, say 10k Sol on a likely outcome
2. The attacker uses Wallets B,C,D to place many tiny minimum bets on the same outcome, getting many valid leaves
3. Once the market resolves, the attacker generates ZK proofs for their tiny bets.
4. The attacker calls multiple `claim_winning` multiple times using wallet A as the claimer (passing Wallet A's commitment account) but submittting the ZK proofs from their tiny bets.
5. The contract pays put the massive rewards multiple times, draining the entire market pool

## Team's response
>Fixed in commit fa5b4a2.  
>amount is now a public input in the ownership circuit. On-chain verification validates that the amount in the proof matches commitment.amount in the account. Decoupling is no longer possible — the proof must contain the exact amount that was committed.  

>Circuit re-compiled, new verification key generated and deployed to devnet.  

---

Rest of the two findings (One High severity and One Medium ) are being excluded from this writeup, One of them was a design choice and other was Fixed within 2 hours of submission.


# Final Thoughts
This was my first ever contribution to open source projects, and I had a really good time overall. The team was very supportive. I had my doubts when I first messaged them, but It turned out great. 

Throughout the whole submission process, The Protocol was very helpful, they cleared out my doubts and even pointed out the things I should focus on. After the submission I was rewarded appropriately
Signing Off ✌️