    ZIP: Unassigned
    Title: Zcash Shielded Coinholder Voting
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

Vote chain
: A purpose-built Cosmos SDK blockchain that serves as the single source
  of truth for all voting operations. The vote chain stores vote
  commitments, nullifier sets, and encrypted share accumulators, and
  verifies zero-knowledge proofs for each transaction type.

Voting round
: A complete instance of a coinholder vote, from round creation through
  tally. Each round is scoped to a single Zcash mainnet snapshot and a
  fresh Election Authority key.

Vote round ID
: A unique identifier for a voting round, derived from the round's
  snapshot and proposal parameters. See [Round Creation] for the
  computation.

Poll runner
: The entity responsible for conducting a voting round: selecting the
  snapshot height, coordinating validators, publishing the round, and
  overseeing the tally.

Vote manager
: The on-chain role that creates voting rounds. Only the vote manager
  can publish new rounds via `MsgCreateVotingSession`.

Bootstrap operator
: The entity that generates the vote chain genesis, funds validators
  from the token reserve, and controls initial consensus power
  distribution.

Validator
: A vote chain consensus participant that runs the chain, participates
  in the EA key ceremony, and contributes to the automatic tally. Each
  validator maintains three keypairs: consensus, account, and Pallas.

Snapshot height
: The Zcash mainnet block height at which eligible Orchard note balances
  are captured. See [Snapshot Configuration] for constraints.

For definitions of cryptographic terms including *alternate nullifier*,
*nullifier non-membership tree*, *nullifier domain*, *pool snapshot*, and
*claim*, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
EA key ceremony terms, see [^draft-ceremony]. For PIR-related terms, see
[^draft-pir].


# Abstract

This ZIP specifies the node operator and poll runner concerns for
conducting a Zcash shielded coinholder vote: vote chain infrastructure
setup, validator onboarding, voting round configuration, and tally
verification. For the EA key ceremony protocol, see [^draft-ceremony].
For the voter-facing voting protocol (delegation, vote casting, share
reveal), see [^draft-voting-protocol].


# Motivation

ZIP 1016 [^zip-1016] establishes a Coinholder-Controlled Fund funded by 12%
of block rewards, requiring coinholder votes to approve grant proposals.
Conducting such votes requires infrastructure operated by node operators
and poll runners. This ZIP specifies the operational concerns for that
infrastructure — what to build, how to configure it, and how to run a
voting round — separately from the cryptographic protocols and voter-facing
interactions specified in companion ZIPs.


# Privacy Implications

- Communication with the PIR server reveals that a client is participating
  in the voting process, though the PIR protocol hides which specific
  nullifier is being queried.
- The vote chain is a public ledger. All transactions (delegation, vote,
  share reveal) are visible, but their contents are encrypted or
  zero-knowledge proven. Node operators see encrypted data, not plaintext.
- Validator power distribution affects the trust model for the EA key
  ceremony. See [^draft-ceremony] for EA-specific privacy implications.


# Requirements

- A new poll runner can set up infrastructure and conduct a voting round
  by following this specification and the referenced companion ZIPs.
- The vote chain operates as a public, verifiable ledger — anyone can run
  a monitoring node to audit.
- The system operates with partial validator availability.


# Non-requirements

- Governance policy decisions such as proposal eligibility, quorum
  requirements, and fund disbursement rules (see ZIP 1016 [^zip-1016]).
- The cryptographic proof-of-balance protocol (see [^draft-balance-proof]).
- Voter-facing protocol: delegation, vote casting, share splitting, and
  share reveal circuits (see [^draft-voting-protocol]).
- EA key ceremony protocol and tally decryption (see [^draft-ceremony]).
- PIR protocol details (see [^draft-pir]).
- On-chain accountable voting (see [^draft-onchain-voting]).


# Specification

## System Overview

The coinholder voting system operates on a purpose-built Cosmos SDK vote
chain. Zcash mainnet snapshots provide the set of eligible Orchard note
balances.

The vote chain stores:

- A **Vote Commitment Tree** (VCT): a Poseidon Merkle tree of vote
  commitments.
- Three **nullifier sets**: governance nullifiers (alternate nullifiers
  from note claims), VAN nullifiers (from delegation consumption), and
  share nullifiers (from share reveals).
- An **encrypted share accumulator** per (proposal, decision): the
  homomorphic sum of El Gamal ciphertexts for each vote option.

The vote chain verifies a zero-knowledge proof for each transaction type:
delegation, vote, and share reveal. The proof circuits are specified in
[^draft-voting-protocol].

## Roles

### Bootstrap Operator

The bootstrap operator generates the vote chain genesis block, funds
validators from the token reserve, and controls initial consensus power
distribution. Funding amount determines each validator's consensus voting
power. An even distribution across validators reduces the risk of consensus
capture.

### Vote Manager

The vote manager creates voting rounds by submitting
`MsgCreateVotingSession` transactions. Only the vote manager role can
publish new rounds.

Assignment rules:

- **Bootstrap phase**: any bonded validator MAY claim the vote manager
  role.
- **Subsequent rounds**: the current vote manager retains the role, or any
  bonded validator MAY reassign it.

### Validator

Validators participate in consensus, the EA key ceremony
(see [^draft-ceremony]), and automatic tally computation. Each validator
maintains three keypairs:

- **Consensus keypair**: used for CometBFT consensus.
- **Account keypair**: used for submitting chain transactions.
- **Pallas keypair**: used for ECIES key exchange during the EA ceremony.

Validators join the network via the automated `join.sh` script or by
building from source. See [Onboarding Validators].

### Role Summary

| Action                   | Bootstrap Op. | Vote Manager | Validator |
| ------------------------ | :-----------: | :----------: | :-------: |
| Generate genesis         |       X       |              |           |
| Fund validators          |       X       |              |           |
| Create voting round      |               |      X       |           |
| Participate in consensus |               |              |     X     |
| EA key ceremony          |               |              |     X     |
| Compute tally            |               |              |     X     |
| Verify tally             |       X       |      X       |     X     |

## Vote Chain Infrastructure

### Genesis Validator Setup

Prerequisites:

- Linux or macOS
- Go 1.24 or later
- Rust 1.83 or later
- C toolchain (GCC or Clang)

Build the vote chain binary:

    git clone https://github.com/z-cale/zally
    cd zally/sdk
    make install-ffi

Initialize the chain:

    zallyd init <moniker> --chain-id zvote-1

This generates a Pallas keypair for the EA ceremony and configures the
REST API on port 1318.

Network ports:

| Port  | Protocol | Exposure | Purpose              |
| ----- | -------- | -------- | -------------------- |
| 26656 | P2P      | Public   | Peer-to-peer gossip  |
| 26657 | RPC      | Local    | CometBFT RPC         |
| 1318  | REST     | Public   | Application REST API |

Start the chain and verify that block production begins:

    zallyd start
    curl http://localhost:26657/status

### Onboarding Validators

**Automated flow** (`join.sh`):

1. Download prebuilt binaries.
2. Discover the network via service discovery (see [Service Discovery]).
3. Fetch genesis and sync to current height.
4. Generate consensus, account, and Pallas keypairs.
5. Request funding from the bootstrap operator via the admin UI.
6. Auto-register with `MsgCreateValidatorWithPallasKey`.

**Source-based flow**:

    mise run validator:join

**Funding**: the bootstrap operator sends tokens to the validator's account
address. The funding amount equals the validator's consensus voting power.

### Nullifier Service (PIR Server)

The nullifier service provides nullifier exclusion proofs to voters
via PIR.

**Bootstrap**:

1. Ingest Orchard nullifiers from Zcash mainnet via `lightwalletd`.
2. Build the Indexed Merkle Tree (nullifier non-membership tree) as
   specified in [^draft-balance-proof].
3. Prepare the three-tier PIR database as specified in [^draft-pir].

**Serve**: expose a query endpoint for voters to privately retrieve
exclusion proofs. See [^draft-pir] for the YPIR+SP protocol and query
mechanics.

### Service Discovery

An Edge Config registry (hosted on Vercel) stores:

- Validator P2P addresses and REST API URLs.
- PIR server URLs.

Wallet clients and the `join.sh` script discover the network through this
registry.

## Conducting a Voting Round

### Snapshot Configuration

The poll runner selects a Zcash mainnet snapshot height subject to these
constraints:

- The height MUST be at or after NU5 activation (Orchard is required).
- The height MUST be a multiple of 10.

Selecting the snapshot triggers:

1. The PIR server rebuilds its nullifier non-membership tree at that
   height.
2. The note commitment tree root ($\mathsf{nc\_root}$) and nullifier IMT
   root ($\mathsf{nullifier\_imt\_root}$) are captured at the snapshot
   height.

### Round Creation

The vote manager publishes a new round via `MsgCreateVotingSession` with
the following parameters:

- `snapshot_height`: the selected Zcash mainnet block height.
- `snapshot_blockhash`: the block hash at `snapshot_height`.
- `proposals`: 1 to 16 proposals, each with 2 to 8 labeled options (e.g.,
  "Support" / "Oppose").
- `vote_end_time`: deadline for all voting phases.
- `nullifier_imt_root`: root of the nullifier non-membership tree at
  snapshot.
- `nc_root`: Orchard note commitment tree root at snapshot.
- `verification_keys`: verification keys for the ZKP circuits (delegation,
  vote, share reveal).

The vote round ID is computed as:

$$
\mathsf{vote\_round\_id} = \mathsf{Blake2b}(\mathsf{snapshot\_height} \| \mathsf{snapshot\_blockhash} \| \mathsf{proposals\_hash} \| \mathsf{vote\_end\_time} \| \mathsf{nullifier\_imt\_root} \| \mathsf{nc\_root})
$$

where Blake2b [^blake2] is used with a 256-bit output.

The round enters the **PENDING** state. The EA key ceremony
(see [^draft-ceremony]) runs automatically. On successful completion, the
round transitions to **ACTIVE** and the voting window opens.

### Round Lifecycle

1. **PENDING**: round created, awaiting EA key ceremony.
2. **ACTIVE**: ceremony complete, voting window open. Voters may delegate,
   vote, and submit shares (see [^draft-voting-protocol]).
3. **TALLYING**: `vote_end_time` has passed. Tally decryption runs
   automatically (see [^draft-ceremony]).
4. **COMPLETE**: tally published and verifiable.

### Timing Parameters

| Parameter                      | Value        | Notes                         |
| ------------------------------ | ------------ | ----------------------------- |
| EA ceremony timing             |              | See [^draft-ceremony]         |
| Voting window                  | Configurable | Set by `vote_end_time`        |
| Tally computation              | Automatic    | Triggered after window closes |

## Verification and Auditing

Anyone MAY run a **monitoring node** — a full chain replica that does not
participate in consensus — to independently verify all aspects of a voting
round:

- Verify every zero-knowledge proof submitted in delegation, vote, and
  share reveal transactions.
- Track all three nullifier sets (governance, VAN, share) for
  double-spending.
- Recompute the aggregate El Gamal ciphertexts per (proposal, decision)
  from individual share reveals.
- Verify the DLEQ proof in `MsgSubmitTally` to confirm correct decryption
  (see [^draft-ceremony]).

No trust in the Election Authority or validators is required for tally
verification: the DLEQ proof is independently checkable by any party with
access to the chain state.


# Rationale

**Separate vote chain (not Zcash mainnet)**: the vote chain is purpose-built
for governance with ZKP-optimized state transitions (Poseidon hashing, custom
transaction types). Zcash mainnet's transaction throughput and scripting model
are not designed for interactive multi-phase voting protocols.


# Reference implementation

[z-cale/zally](https://github.com/z-cale/zally) — a Go and Rust
implementation built on Cosmos SDK with Halo 2 zero-knowledge proof
circuits.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-voting-protocol.md)

[^draft-ceremony]: [Draft ZIP: Election Authority Key Ceremony](draft-valargroup-ea-key-ceremony.md)

[^draft-pir]: [Draft ZIP: Private Information Retrieval for Nullifier Exclusion Proofs](draft-valargroup-gov-pir.md)

[^draft-onchain-voting]: [Draft ZIP: On-chain Accountable Voting](draft-ecc-onchain-accountable-voting.md)

[^blake2]: [J.-P. Aumasson, S. Neves, Z. Wilcox-O'Hearn, and C. Winnerlein, "BLAKE2: simpler, smaller, fast as MD5", in Applied Cryptography and Network Security, pp. 119-135, 2013](https://doi.org/10.1007/978-3-642-38980-1_8)
