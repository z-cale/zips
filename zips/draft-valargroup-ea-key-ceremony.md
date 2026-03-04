    ZIP: Unassigned
    Title: Election Authority Key Ceremony
    Owners: Dev Ojha <dojha@berkeley.edu>
            Roman Akhtariev <ackhtariev@gmail.com>
            Adam Tucker <adamleetucker@outlook.com>
            Greg Nagy <greg@dhamma.works>
    Status: Draft
    Category: Process
    Created: 2026-03-04
    License: MIT
    Pull-Request: <https://github.com/zcash/zips/pull/???>


# Terminology

The key words "MUST", "REQUIRED", "MUST NOT", "SHOULD", and "MAY" in this
document are to be interpreted as described in BCP 14 [^BCP14] when, and
only when, they appear in all capitals.

The terms below are to be interpreted as follows:

Election Authority (EA)
: A logical role whose secret key $\mathsf{ea\_sk}$ is used for El Gamal
  encryption [^elgamal] of vote shares. In the current design, all ACK'd
  validators hold $\mathsf{ea\_sk}$ for the round.

EA key ceremony
: A per-round protocol that produces a fresh El Gamal keypair
  $(\mathsf{ea\_sk}, \mathsf{ea\_pk})$ and distributes it to eligible
  validators via ECIES [^ecies].

Dealer
: The validator selected to generate the EA keypair and distribute it to
  all eligible validators. The next block proposer is automatically
  selected as dealer.

Vote chain
: A purpose-built Cosmos SDK blockchain that serves as the single source
  of truth for voting operations. See the Coinholder Voting Process
  ZIP [^draft-coinholder-voting] for infrastructure details.

Validator
: A vote chain consensus participant. Each validator maintains three
  keypairs: consensus (CometBFT), account (chain transactions), and
  Pallas (ECIES key exchange during the ceremony).

Voting round
: A complete instance of a coinholder vote. Each round is scoped to a
  single Zcash mainnet snapshot and a fresh EA key.


# Abstract

This ZIP specifies the Election Authority (EA) key ceremony used in
Zcash shielded coinholder voting. Each voting round requires a fresh
El Gamal keypair for encrypting vote shares. The ceremony automates
key generation, distribution via ECIES, and validator acknowledgment,
producing a per-round keypair that enables homomorphic tally aggregation
and publicly verifiable decryption via Chaum-Pedersen DLEQ proofs.


# Motivation

The coinholder voting system encrypts individual vote shares under an
Election Authority public key so that no party learns individual vote
amounts during the voting window. After the window closes, the EA secret
key is used to decrypt the homomorphically aggregated ciphertext and
produce a verifiable tally.

Generating and distributing this key securely, automatically, and per-round
is critical: a long-lived key would accumulate risk, manual distribution
would not scale, and a single point of failure would compromise liveness.
This ZIP specifies the automated ceremony that addresses these concerns.


# Privacy Implications

- All validators that ACK a round's ceremony hold that round's
  $\mathsf{ea\_sk}$. If any one of them is compromised, an adversary can
  decrypt individual encrypted vote shares for that round, breaking
  vote-amount privacy. Voter identity remains protected because alternate
  nullifiers are unlinkable to on-chain spending.
- Each round uses a fresh $\mathsf{ea\_sk}$. Compromise of one round's
  key does not affect privacy of past or future rounds.
- The dealer learns $\mathsf{ea\_sk}$ by construction. In the current
  trusted-dealer model, the dealer has the same capabilities as any
  other ACK'd validator for that round.


# Requirements

- A fresh EA keypair is generated for each voting round with no manual
  intervention.
- The ceremony completes with partial validator availability (at least
  one-third of eligible validators).
- No single validator's non-cooperation blocks the ceremony indefinitely.
- The resulting keypair enables homomorphic aggregation and publicly
  verifiable decryption.


# Non-requirements

- Threshold secret sharing or distributed key generation (future upgrade;
  see [Security Considerations]).
- The voting protocol itself: delegation, vote casting, share submission,
  and share reveal circuits (see [^draft-voting-protocol]).
- Vote chain infrastructure setup (see [^draft-coinholder-voting]).


# Specification

## Ceremony Protocol

Each voting round triggers a fresh EA key ceremony after
`MsgCreateVotingSession` is accepted and the round enters the **PENDING**
state.

### Eligibility

All validators with a registered Pallas public key at the time of round
creation are eligible for the ceremony.

### Dealer Selection

The next block proposer is automatically selected as the dealer.

### Key Generation and Distribution

The dealer:

1. Generates a random scalar $\mathsf{ea\_sk} \in \mathbb{F}_{q_{\mathbb{P}}}$.
2. Computes $\mathsf{ea\_pk} = \mathsf{ea\_sk} \cdot G$ where $G$ is the
   Pallas generator.
3. For each eligible validator $V_i$ with Pallas public key $\mathsf{pk}_i$:
   a. Performs ephemeral ECDH on the Pallas curve: generates ephemeral
      scalar $e_i$, computes shared secret from $e_i \cdot \mathsf{pk}_i$.
   b. Derives a symmetric key from the shared secret.
   c. Encrypts $\mathsf{ea\_sk}$ using ChaCha20-Poly1305 with the derived
      key.
4. Publishes $\mathsf{ea\_pk}$ and the per-validator ECIES ciphertexts to
   the chain.

### Validator Acknowledgment

Each eligible validator:

1. Decrypts $\mathsf{ea\_sk}$ from its ECIES ciphertext using its Pallas
   secret key.
2. Verifies that $\mathsf{ea\_sk} \cdot G = \mathsf{ea\_pk}$.
3. Submits an ACK message via `PrepareProposal`.

### Confirmation

The ceremony confirms under one of the following conditions:

- **Fast path**: all eligible validators ACK. The ceremony confirms
  immediately.
- **Timeout path**: after the ACK phase timeout (30 minutes), if at least
  one-third of eligible validators have ACK'd, the ceremony confirms.
  Non-ACK'd validators are stripped from the round and increment a
  consecutive-miss counter. After 3 consecutive misses, the validator
  MUST be jailed.
- **Failure**: if fewer than one-third of eligible validators ACK within
  the timeout, the ceremony resets and a new dealer is selected.

On successful confirmation, the round transitions from **PENDING** to
**ACTIVE**.

## Tally Decryption

After the voting window closes and the round transitions to **TALLYING**:

1. The block proposer loads $\mathsf{ea\_sk}$ and decrypts the aggregate
   ciphertext for each (proposal, decision) pair:

   $$
   \mathsf{total\_value} \cdot G = C_{2,\text{agg}} - \mathsf{ea\_sk} \cdot C_{1,\text{agg}}
   $$

2. The proposer recovers $\mathsf{total\_value}$ from
   $\mathsf{total\_value} \cdot G$ using baby-step-giant-step discrete
   logarithm (feasible because $\mathsf{total\_value}$ is bounded by total
   ZEC supply).

3. The proposer submits `MsgSubmitTally` with the recovered values and a
   Chaum-Pedersen DLEQ proof [^cp92] demonstrating correct decryption.

4. Results are queryable via the REST API:

       GET /zally/v1/tally-results/{round_id}

   The response contains, for each (proposal, decision) pair, the
   `total_value` in zatoshi.

### DLEQ Verification

Any party with access to the chain state MAY independently verify the
tally by checking the DLEQ proof against $\mathsf{ea\_pk}$, the aggregate
ciphertexts, and the claimed totals. No trust in the EA or validators is
required.

## Key Retention

The $\mathsf{ea\_sk}$ file for each round MUST be retained indefinitely
(32 bytes per key) to allow future retally or audit. Keys MUST NOT be
deleted.

## Timing Parameters

| Parameter                      | Value        | Notes                         |
| ------------------------------ | ------------ | ----------------------------- |
| Ceremony deal timeout          | ~30 blocks   | TBD; time for dealer message  |
| ACK phase timeout              | 30 minutes   | Fixed                         |
| Consecutive ceremony miss jail | 3 misses     | Validator jailed, not slashed |
| Slashing fractions             | 0            | Jailing only, no token burns  |


# Rationale

**Per-round keys**: scoping $\mathsf{ea\_sk}$ to a single round limits the
impact of key compromise to that round only. Validator rotation between
rounds is handled naturally — departing validators cannot decrypt future
rounds. This avoids the complexity of re-initialization or long-lived key
management.

**Trusted dealer**: the current model is simple and sufficient for the
initial deployment. The dealer is a validator already trusted for consensus
and is automatically rotated via block proposer selection.

**ECIES on Pallas**: reuses the Pallas curve already present in the Orchard
protocol, avoiding additional curve dependencies. ChaCha20-Poly1305
provides authenticated encryption.

**Jailing, not slashing**: ceremony non-participation is penalized by
jailing (excluding from future ceremonies) rather than token slashing.
This is a liveness signal, not a safety violation.


# Security Considerations

**Trust model**: all ACK'd validators hold $\mathsf{ea\_sk}$ for the
round. Compromise of any single validator breaks vote-amount privacy for
that round. Voter identity remains protected because alternate nullifiers
are unlinkable to on-chain spending.

**Key isolation**: each round uses a different $\mathsf{ea\_sk}$. A
validator compromised after one round cannot decrypt votes in subsequent
rounds.

**Trusted dealer limitation**: the dealer generates and knows
$\mathsf{ea\_sk}$. In the current model, this is equivalent to any other
ACK'd validator's knowledge. A future upgrade path is:

1. **Threshold secret sharing** (t-of-n Shamir shares with Feldman
   commitments): the dealer distributes shares such that no single
   validator holds the full key. Decryption requires cooperation of at
   least $t$ validators.
2. **Distributed key generation** (DKG): eliminates the trusted dealer
   entirely. Validators jointly generate $\mathsf{ea\_pk}$ without any
   single party ever holding the complete $\mathsf{ea\_sk}$.

**Post-quantum**: El Gamal encryption is breakable by a quantum adversary
with a sufficiently large quantum computer. A successful attack would
expose individual share amounts for the affected round. Post-quantum
migration is out of scope for this ZIP.


# Reference implementation

[z-cale/zally](https://github.com/z-cale/zally) — a Go and Rust
implementation built on Cosmos SDK with Halo 2 zero-knowledge proof
circuits.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-coinholder-voting]: [Draft ZIP: Zcash Shielded Coinholder Voting](draft-valargroup-coinholder-voting.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-voting-protocol.md)

[^elgamal]: [T. ElGamal, "A public key cryptosystem and a signature scheme based on discrete logarithms", IEEE Transactions on Information Theory, vol. 31, no. 4, pp. 469-472, 1985](https://doi.org/10.1109/TIT.1985.1057074)

[^cp92]: [D. Chaum and T. P. Pedersen, "Wallet Databases with Observers", in Advances in Cryptology — CRYPTO '92, pp. 89-105, 1993](https://doi.org/10.1007/3-540-48071-4_7)

[^ecies]: [V. Shoup, "A Proposal for an ISO Standard for Public Key Encryption", version 2.1, 2001](https://www.shoup.net/papers/iso-2_1.pdf)
