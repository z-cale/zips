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
: A logical role whose keypair $(\mathsf{ea\_sk}, \mathsf{ea\_pk})$
  governs El Gamal encryption [^elgamal] of vote shares for a given
  voting round.

EA key ceremony
: A per-round protocol that produces a fresh El Gamal keypair
  $(\mathsf{ea\_sk}, \mathsf{ea\_pk})$ and distributes key material to
  eligible validators.

Dealer
: The validator selected to generate the EA keypair and distribute key
  material to eligible validators.

Vote chain
: The blockchain that serves as the single source of truth for voting
  operations. See the Coinholder Voting Process
  ZIP [^draft-coinholder-voting] for infrastructure details.

Validator
: A vote chain consensus participant that maintains keypairs for
  consensus, account transactions, and Pallas-based key exchange.

Voting round
: A complete instance of a coinholder vote, scoped to a single Zcash
  mainnet snapshot and a fresh EA key.

Share
: A Shamir secret share $f(i)$ of $\mathsf{ea\_sk}$, held by a single
  validator after the ceremony.

Threshold ($t$)
: The minimum number of shares required to reconstruct
  $\mathsf{ea\_sk}$ or perform threshold decryption.

Verification key
: A per-validator public commitment $\mathsf{VK}_i = f(i) \cdot G$
  used to verify that a validator received a correct share.


# Abstract

This ZIP specifies the Election Authority (EA) key ceremony used in
Zcash shielded coinholder voting. Each voting round requires a fresh
El Gamal keypair: vote shares are encrypted under the EA public key,
homomorphically aggregated on-chain, and decrypted after the voting
window closes to produce a verifiable tally.

The ceremony is triggered automatically when a voting round enters the
PENDING state. The block proposer acts as the dealer: it generates the
EA keypair, splits the secret key into Shamir shares with threshold
$t = \lceil n/2 \rceil$, encrypts each share to the recipient
validator's Pallas public key via ECIES, and publishes the encrypted
shares and verification keys to the chain. Validators decrypt and
acknowledge their shares; the ceremony confirms when a majority of
eligible validators have acknowledged.

After the voting window closes, at least $t$ validators submit partial
decryptions that are combined via Lagrange interpolation to recover the
aggregate plaintext. A Chaum-Pedersen DLEQ proof published alongside
the tally allows any party to verify correct decryption against the
on-chain aggregate ciphertexts and the EA public key, without trusting
the Election Authority or any individual validator.


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

- The dealer generates $\mathsf{ea\_sk}$ and learns it during key
  generation before erasing it. This is the primary trust assumption:
  the dealer is a consensus-participating validator, automatically
  rotated via block proposer selection. A future DKG upgrade path would
  eliminate this (see [Rationale]).
- Individual validators hold only a Shamir share of
  $\mathsf{ea\_sk}$, not the full key. An adversary needs to compromise
  at least $t$ validators (where $t = \lceil n/2 \rceil$) to
  reconstruct the secret and decrypt individual vote shares.
  Voter identity remains protected because alternate nullifiers are
  unlinkable to on-chain spending.
- Each round uses a fresh $\mathsf{ea\_sk}$. Compromise of one round's
  key material does not affect privacy of past or future rounds.


# Requirements

- A fresh EA keypair is generated for each voting round with no manual
  intervention.
- The ceremony completes with partial validator availability (at least
  a majority of eligible validators).
- No single validator's non-cooperation blocks the ceremony indefinitely.
- The resulting keypair enables homomorphic aggregation and publicly
  verifiable decryption.
- No single validator (other than the dealer during key generation)
  learns the full $\mathsf{ea\_sk}$.


# Non-requirements

- Distributed key generation that eliminates the trusted dealer (future
  upgrade; see [Rationale]).
- The voting protocol itself: delegation, vote casting, share submission,
  and share reveal circuits (see [^draft-voting-protocol]).
- Vote chain infrastructure setup (see [^draft-coinholder-voting]).


# Specification

## El Gamal on Pallas

This section defines the El Gamal encryption scheme used throughout the
voting system. The Voting Protocol ZIP [^draft-voting-protocol] references
this section for vote share encryption.

### Setup

Let $G$ be the Pallas [^protocol-pallasandvesta] generator defined as
$\mathcal{G}^{\mathsf{Orchard}}$ in the Zcash protocol
specification [^protocol-orchardkeycomponents] (the Orchard spend
authorization base point). Let $\mathbb{F}_q$ denote the scalar field of
the Pallas curve. The EA keypair is:

- $\mathsf{ea\_sk} \in \mathbb{F}_q$: a random scalar.
- $\mathsf{ea\_pk} = \mathsf{ea\_sk} \cdot G$: the corresponding public
  key.

All El Gamal, ECIES, and DLEQ operations in this ZIP MUST use this
generator. Using an arbitrary point would break the homomorphic property
and compatibility with the voting circuit.

### Encryption

To encrypt a value $v$ (expressed in ballots, i.e., zatoshi
floor-divided by 12,500,000) with randomness
$r \leftarrow \mathbb{F}_q$:

$$
\mathsf{Enc}(v, r) = (r \cdot G, \enspace v \cdot G + r \cdot \mathsf{ea\_pk})
$$

The ciphertext is a pair of Pallas points $(C_1, C_2)$.

### Additive Homomorphism

Given ciphertexts $\mathsf{Enc}(a, r_1)$ and $\mathsf{Enc}(b, r_2)$,
component-wise point addition yields a valid encryption of the sum:

$$
\mathsf{Enc}(a, r_1) + \mathsf{Enc}(b, r_2) = \mathsf{Enc}(a + b, \enspace r_1 + r_2)
$$

This allows anyone to publicly aggregate encrypted vote shares without
decryption. The vote chain accumulates per-(proposal, decision) aggregates
by summing ciphertexts as share reveal transactions arrive.

### Decryption

Given an aggregate ciphertext $(C_{1,\text{agg}}, C_{2,\text{agg}})$ and
$\mathsf{ea\_sk}$:

$$
C_{2,\text{agg}} - \mathsf{ea\_sk} \cdot C_{1,\text{agg}} = \mathsf{total\_value} \cdot G
$$

To recover $\mathsf{total\_value}$ from
$\mathsf{total\_value} \cdot G$, the decryptor performs a bounded
discrete logarithm lookup using baby-step-giant-step. This is feasible
because $\mathsf{total\_value}$ is bounded by total ZEC supply
(approximately $2.1 \times 10^{15}$ zatoshi, or $1.68 \times 10^{8}$
ballots).

### Chaum-Pedersen DLEQ Proof

After decryption, the decryptor MUST publish a Chaum-Pedersen DLEQ
proof [^cp92] demonstrating that the same $\mathsf{ea\_sk}$ used to
generate $\mathsf{ea\_pk}$ was used to decrypt the aggregate. The proof
establishes:

$$
\log_G(\mathsf{ea\_pk}) = \log_{C_{1,\text{agg}}}(D)
$$

where $D = \mathsf{ea\_sk} \cdot C_{1,\text{agg}}$ and
$D = C_{2,\text{agg}} - \mathsf{total\_value} \cdot G$.

The proof protocol is:

1. Prover samples $k \leftarrow \mathbb{F}_q$ and computes
   $R_1 = k \cdot G$ and $R_2 = k \cdot C_{1,\text{agg}}$.
2. Fiat-Shamir challenge:
   $c = \mathsf{H2S}(\mathsf{BLAKE2b\text{-}256}(\texttt{"zally-dleq-v1"} \| \mathsf{compress}(G) \| \mathsf{compress}(\mathsf{ea\_pk}) \| \mathsf{compress}(C_{1,\text{agg}}) \| \mathsf{compress}(D) \| \mathsf{compress}(R_1) \| \mathsf{compress}(R_2)))$
   where $\mathsf{compress}$ is the 32-byte compressed Pallas point
   encoding and $\mathsf{H2S}$ converts the 32-byte digest to a Pallas
   scalar.
3. Response: $z = k + c \cdot \mathsf{ea\_sk}$.
4. The proof is serialized as $(c, z)$, each a 32-byte Pallas scalar
   (64 bytes total).

Verification:

1. Compute $R_1 = z \cdot G - c \cdot \mathsf{ea\_pk}$ and
   $R_2 = z \cdot C_{1,\text{agg}} - c \cdot D$.
2. Recompute $c'$ from $G$, $\mathsf{ea\_pk}$, $C_{1,\text{agg}}$, $D$,
   $R_1$, $R_2$ using the same hash as above.
3. Accept if $c = c'$.

Any party MAY independently verify the tally using only the on-chain data
($\mathsf{ea\_pk}$, aggregate ciphertexts, claimed totals, and the proof).
No trust in the EA or validators is required.

## ECIES on Pallas

This section defines the ECIES [^ecies] construction used to distribute
Shamir shares to validators during the ceremony.

For a recipient validator $V_i$ with Pallas public key
$\mathsf{pk}_i = \mathsf{sk}_i \cdot G$, and plaintext $m$
(the serialized share $f(i)$):

**Encryption** (by the dealer):

1. Generate ephemeral scalar $e_i \leftarrow \mathbb{F}_q$
   and compute ephemeral public key $E_i = e_i \cdot G$.
2. Compute ECDH shared secret $S_i = e_i \cdot \mathsf{pk}_i$.
3. Derive symmetric key
   $k_i = \mathsf{SHA256}(\mathsf{compress}(E_i) \| \mathsf{x}(S_i))$
   where $\mathsf{compress}$ is the 32-byte compressed Pallas point
   encoding and $\mathsf{x}(S_i)$ is the $x$-coordinate of $S_i$
   extracted by taking the compressed encoding and clearing bit 7 of
   byte 31 (the sign bit).
4. Encrypt: $\mathsf{ct}_i = \mathsf{ChaCha20\text{-}Poly1305}(k_i, \mathsf{nonce}{=}0, m)$.
5. Output $(E_i, \mathsf{ct}_i)$.

**Decryption** (by validator $V_i$):

1. Compute $S_i = \mathsf{sk}_i \cdot E_i$.
2. Derive $k_i = \mathsf{SHA256}(\mathsf{compress}(E_i) \| \mathsf{x}(S_i))$.
3. Decrypt $m = \mathsf{ChaCha20\text{-}Poly1305.decrypt}(k_i, \mathsf{nonce}{=}0, \mathsf{ct}_i)$.
4. Parse $m$ as share $f(i)$ and verify
   $f(i) \cdot G = \mathsf{VK}_i$.

A fresh ephemeral scalar $e_i$ MUST be generated for each validator to
prevent cross-validator key correlation.

## Pallas Key Registration

Before participating in any ceremony, a validator MUST register a
Pallas public key on-chain. Two registration paths are available:

- `MsgRegisterPallasKey`: for an existing bonded validator.
- `MsgCreateValidatorWithPallasKey`: atomically creates a staking
  validator and registers the Pallas key in a single transaction.

The submitted key MUST be a 32-byte compressed Pallas point that is on
the curve and is not the identity point. Each validator operator address
MAY register at most one Pallas key; duplicate registrations MUST be
rejected.

The registered key persists across rounds and is used for ECIES key
exchange during each ceremony the validator participates in.

## Ceremony Protocol

Each voting round triggers a fresh EA key ceremony after the round enters
the **PENDING** state.

### Eligibility

All validators with a registered Pallas public key at the time of round
creation are eligible for the ceremony.

### Dealer Selection

The next block proposer is automatically selected as the dealer.

### Key Generation and Distribution

Let $n$ be the number of eligible validators and
$t = \lceil n/2 \rceil$ (minimum 2) be the threshold.

The dealer:

1. Generates $\mathsf{ea\_sk} \leftarrow \mathbb{F}_q$ and
   computes $\mathsf{ea\_pk} = \mathsf{ea\_sk} \cdot G$.
2. Constructs a random polynomial
   $f(x) = \mathsf{ea\_sk} + a_1 x + \cdots + a_{t-1} x^{t-1}$
   of degree $t - 1$ with $f(0) = \mathsf{ea\_sk}$ and random
   coefficients
   $a_1, \ldots, a_{t-1} \leftarrow \mathbb{F}_q$.
3. Evaluates $f(i)$ for $i = 1, \ldots, n$ to produce $n$ Shamir
   shares [^shamir].
4. Computes verification keys $\mathsf{VK}_i = f(i) \cdot G$ for each
   share.
5. For each eligible validator $V_i$, encrypts the share $f(i)$ to
   $\mathsf{pk}_i$ using the ECIES construction defined in
   [ECIES on Pallas], producing $(E_i, \mathsf{ct}_i)$.
6. Publishes to the chain: $\mathsf{ea\_pk}$, the threshold $t$, all
   $(E_i, \mathsf{ct}_i)$ pairs, and all $\mathsf{VK}_i$.
7. Securely erases $\mathsf{ea\_sk}$, the polynomial coefficients, and
   all share values from memory.

### Validator Acknowledgment

Each eligible validator $V_i$:

1. Decrypts share $f(i)$ from $(E_i, \mathsf{ct}_i)$ per
   [ECIES on Pallas], which includes verification that
   $f(i) \cdot G = \mathsf{VK}_i$.
2. Stores $f(i)$ securely on disk.
3. Submits an ACK message to the chain containing the commitment
   $\mathsf{SHA256}(\texttt{"ack"} \| \mathsf{ea\_pk} \| \mathsf{validator\_address})$.

### Confirmation

The ceremony confirms under one of the following conditions:

- **Fast path**: all eligible validators ACK. The ceremony confirms
  immediately.
- **Timeout path**: after the ACK phase timeout (30 minutes), if at least
  a majority of eligible validators have ACK'd (i.e.,
  $\mathsf{acks} \times 2 \geq n$), the ceremony confirms. Non-ACK'd
  validators are stripped from the round and increment a
  consecutive-miss counter. After 3 consecutive misses, the validator
  MUST be jailed.
- **Failure**: if fewer than a majority of eligible validators ACK within
  the timeout, the ceremony resets and a new dealer is selected.

The majority threshold ensures that at least $t$ shares are available
for threshold decryption during the tally phase.

On successful confirmation, the round transitions from **PENDING** to
**ACTIVE**.

### Validator Set Changes

**Joining**: in the threshold model, new validators joining after the
ceremony cannot receive an independent share without a new dealing round.
An existing ACK'd validator MAY forward their own share to the new
validator via [ECIES on Pallas], allowing the new validator to
participate in threshold decryption using the forwarded share.

**Leaving**: a departing validator retains their share $f(i)$ and cannot
be forced to delete it. Under the threshold model, a single share does
not reveal $\mathsf{ea\_sk}$. Key rotation (generating a fresh
$\mathsf{ea\_sk'}$ for future rounds) further limits exposure.

## Tally

After the voting window closes and the round transitions to **TALLYING**:

1. For each (proposal, decision) pair, the aggregate ciphertext
   $(C_{1,\text{agg}}, C_{2,\text{agg}})$ is the component-wise sum of all
   submitted share ciphertexts. This aggregation is publicly verifiable —
   anyone can replay the addition from on-chain data.

2. At least $t$ validators submit partial decryptions to the chain.
   Each participating validator $V_i$ computes:

   $$D_i = f(i) \cdot C_{1,\text{agg}}$$

3. The partial decryptions are combined via Lagrange interpolation in
   the exponent. Given partial decryptions $\{(i, D_i)\}$ from a set
   $S$ of at least $t$ validators, the Lagrange coefficients are:

   $$\lambda_i = \prod_{j \in S,\, j \neq i} \frac{-j}{i - j}$$

   The combined result is:

   $$\mathsf{ea\_sk} \cdot C_{1,\text{agg}} = \sum_{i \in S} \lambda_i \cdot D_i$$

4. The aggregate plaintext is recovered:

   $$\mathsf{total\_value} \cdot G = C_{2,\text{agg}} - \mathsf{ea\_sk} \cdot C_{1,\text{agg}}$$

   $\mathsf{total\_value}$ is recovered via baby-step-giant-step.

5. Any set of $t$ validators MAY cooperate to reconstruct
   $\mathsf{ea\_sk}$ from their shares and produce the DLEQ proof per
   [Chaum-Pedersen DLEQ Proof]. The decryptor publishes
   $\mathsf{total\_value}$ for each (proposal, decision) pair along
   with the proof.

6. Any party MAY verify the proof against the on-chain aggregate
   ciphertexts and $\mathsf{ea\_pk}$.

Individual vote amounts are never revealed — only the aggregate total per
(proposal, decision).

## Key Retention

Each validator's share file for each round MUST be retained indefinitely
(32 bytes per share) to allow future retally or audit. Share files MUST
NOT be deleted.

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

**Threshold secret sharing**: splitting $\mathsf{ea\_sk}$ into Shamir
shares ensures that no single validator (other than the dealer at
generation time) holds the complete secret key. Compromise of fewer than
$t$ validators does not reveal the key. The threshold
$t = \lceil n/2 \rceil$ balances liveness (not requiring all validators
for decryption) with privacy (requiring a majority to collude for key
reconstruction).

**Trusted dealer with future DKG upgrade**: the dealer generates and
momentarily holds $\mathsf{ea\_sk}$ before erasing it. This is simpler
than a full distributed key generation (DKG) protocol and sufficient for
the initial deployment, given that the dealer is a validator already
trusted for consensus and is automatically rotated via block proposer
selection. A future DKG upgrade would eliminate this trust assumption
entirely by having validators jointly generate $\mathsf{ea\_pk}$ without
any single party ever holding $\mathsf{ea\_sk}$.

**ECIES on Pallas**: reuses the Pallas curve already present in the Orchard
protocol, avoiding additional curve dependencies. ChaCha20-Poly1305
provides authenticated encryption.

**Jailing, not slashing**: ceremony non-participation is penalized by
jailing (excluding from future ceremonies) rather than token slashing.
This is a liveness signal, not a safety violation.

**Post-quantum**: El Gamal encryption is breakable by a quantum adversary
with a sufficiently large quantum computer. A successful attack would
expose individual vote share amounts for the affected round. Post-quantum
migration is out of scope for this ZIP.


# Reference implementation

[^ref-impl] — a Go and Rust implementation built on Cosmos SDK with
Halo 2 zero-knowledge proof circuits.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^draft-coinholder-voting]: [Draft ZIP: Zcash Shielded Coinholder Voting](draft-valargroup-coinholder-voting-setup.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-voting-protocol.md)

[^elgamal]: [T. ElGamal, "A public key cryptosystem and a signature scheme based on discrete logarithms", IEEE Transactions on Information Theory, vol. 31, no. 4, pp. 469-472, 1985](https://doi.org/10.1109/TIT.1985.1057074)

[^cp92]: [D. Chaum and T. P. Pedersen, "Wallet Databases with Observers", in Advances in Cryptology — CRYPTO '92, pp. 89-105, 1993](https://doi.org/10.1007/3-540-48071-4_7)

[^ecies]: [V. Shoup, "A Proposal for an ISO Standard for Public Key Encryption", version 2.1, 2001](https://www.shoup.net/papers/iso-2_1.pdf)

[^shamir]: [A. Shamir, "How to share a secret", Communications of the ACM, vol. 22, no. 11, pp. 612-613, 1979](https://doi.org/10.1145/359168.359176)

[^protocol-pallasandvesta]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 5.4.9.6: Pallas and Vesta](protocol/protocol.pdf#pallasandvesta)

[^protocol-orchardkeycomponents]: [Zcash Protocol Specification, Version 2025.6.3 [NU6.1]. Section 4.2.3: Orchard Key Components](protocol/protocol.pdf#orchardkeycomponents)

[^ref-impl]: [z-cale/zally: Zcash Shielded Voting reference implementation](https://github.com/z-cale/zally)
