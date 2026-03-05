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
: The blockchain that serves as the single source of truth for voting
  operations. See [System Overview] for the state it maintains.

Voting round
: A complete instance of a coinholder vote, scoped to a single Zcash
  mainnet snapshot and a fresh Election Authority key.

Vote round ID
: A unique identifier for a voting round. See [Round Creation] for the
  computation.

Poll runner
: The entity responsible for conducting a voting round.

Vote manager
: The on-chain role authorized to create voting rounds.

Bootstrap operator
: The entity that provisions the vote chain genesis and initial
  validator set.

Validator
: A vote chain consensus participant. See [Validator] under Roles for
  responsibilities and keypair details.

Election Authority (EA)
: A virtual signing key, jointly constructed by validators during a key
  ceremony so that no single party holds the private key. Used to encrypt
  vote shares and decrypt the final tally. See [^draft-ceremony] for the
  ceremony protocol.

Snapshot height
: The Zcash mainnet block height at which eligible Orchard note balances
  are captured. See [Snapshot Configuration] for constraints.

For definitions of cryptographic terms including *alternate nullifier*,
*nullifier non-membership tree*, *nullifier domain*, *pool snapshot*, and
*claim*, see the Orchard Proof-of-Balance ZIP [^draft-balance-proof]. For
EA key ceremony terms, see [^draft-ceremony]. For PIR-related terms, see
[^draft-pir].


# Abstract

This ZIP specifies how to operate the infrastructure for Zcash shielded
coinholder voting. It defines a purpose-built vote chain built on Cosmos
SDK, three operator roles (bootstrap operator, vote manager, validator),
and the lifecycle of a voting round from snapshot selection through tally
verification.

The vote chain stores vote commitments in a Poseidon Merkle tree, tracks
three nullifier sets to prevent double-voting, and accumulates encrypted
vote shares as homomorphic El Gamal ciphertexts. Each transaction
(delegation, vote, share reveal) is verified by a zero-knowledge proof
on-chain. Validators join through an automated onboarding script that
handles binary distribution, key generation, and on-chain registration.
A separate nullifier service provides private information retrieval of
exclusion proofs so voters can prove note non-spending without revealing
which notes they hold.

Tally correctness is independently verifiable by any party: a DLEQ proof
in the tally submission allows anyone with access to the chain state to
confirm correct decryption without trusting the Election Authority or
validators.

The EA key ceremony protocol is specified in [^draft-ceremony]. The
voter-facing protocol (delegation, vote casting, share reveal) is
specified in [^draft-voting-protocol].


# Motivation

Zcash coinholders need a way to make collective decisions — from fund
disbursement to protocol governance — while preserving the privacy
guarantees they expect from shielded transactions. Conducting such votes
requires dedicated infrastructure: a vote chain, validators, snapshot
configuration, and round management.

ZIP 1016 [^zip-1016] is the initial use case, requiring coinholder votes
to approve grants from the Coinholder-Controlled Fund. The infrastructure
specified here is general-purpose: any coinholder decision that can be
framed as a set of proposals with labeled options can be conducted through
this system.

This ZIP specifies the operational concerns — what to build, how to
configure it, and how to run a voting round — separately from the
cryptographic protocols and voter-facing interactions specified in
companion ZIPs.


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

Validators join the network via the automated `join.sh` [^join-sh]
script or by building from source. See [Onboarding Validators].

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

The bootstrap operator builds the vote chain binary (Go + Rust FFI for
Halo 2 and RedPallas verification), initializes a single-validator chain,
and starts the node. The reference implementation repository [^ref-impl]
contains step-by-step instructions (`SETUP_GENESIS.md`) and mise task
automation.

Initialization generates a Cosmos validator key, a Pallas keypair for the
EA ceremony, and a genesis block with the chain ID `zvote-1`. The node
exposes:

| Port  | Protocol | Exposure | Purpose              |
| ----- | -------- | -------- | -------------------- |
| 26656 | P2P      | Public   | Peer-to-peer gossip  |
| 26657 | RPC      | Local    | CometBFT RPC         |
| 1318  | REST     | Public   | Application REST API |

After the chain is producing blocks, the bootstrap operator registers the
node's public URL in the service discovery layer (see [Service Discovery])
so that joining validators and wallet clients can find the network.

### Onboarding Validators

New validators join through `join.sh` [^join-sh], a self-contained
script that requires no local clone of the repository. See
`SETUP_JOIN.md` in [^ref-impl] for full details. The script:

1. Downloads pre-built `zallyd` and `create-val-tx` binaries and
   verifies their SHA-256 checksums.
2. Discovers a live validator via the service discovery API.
3. Fetches genesis and syncs to the current height.
4. Generates consensus, account, and Pallas keypairs.
5. Self-registers with the service discovery API (appears as "pending"
   in the admin UI).
6. Waits for the bootstrap operator to approve and fund the validator
   via the admin UI.
7. On receiving funds, auto-registers on-chain with
   `MsgCreateValidatorWithPallasKey`.

The funding amount equals the validator's consensus voting power.
Developers with a local clone can alternatively run
`mise run validator:join`, which builds from source and then runs the
same `join.sh` flow.

### Nullifier Service (PIR Server)

The nullifier service provides nullifier exclusion proofs to voters via
PIR. See `SETUP_NULLIFIER_SERVICE.md` in [^ref-impl] for operational
setup.

The service pipeline:

1. **Ingest**: fetch Orchard nullifiers from Zcash mainnet via a
   lightwallet server (`lightwalletd`), or download a pre-built snapshot.
2. **Export**: build the nullifier non-membership tree (Indexed Merkle
   Tree as specified in [^draft-balance-proof]) and export the three-tier
   PIR database as specified in [^draft-pir].
3. **Serve**: expose a query endpoint on port 3000 for voters to
   privately retrieve exclusion proofs.

### Service Discovery

Validator and PIR server URLs are published to a Vercel Edge Config store,
managed through the admin UI. Wallet clients and the `join.sh` script
query this API to discover:

- Vote chain REST API endpoints.
- PIR server endpoints.

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
- `proposals`: 1 to 15 proposals, each with 2 to 8 labeled options (e.g.,
  "Support" / "Oppose"). The limit is 15 because the circuit's
  proposal authority bitmask reserves bit 0 as a sentinel.
- `vote_end_time`: deadline for all voting phases.
- `nullifier_imt_root`: root of the nullifier non-membership tree at
  snapshot.
- `nc_root`: Orchard note commitment tree root at snapshot.
- `verification_keys`: verification keys for the ZKP circuits (delegation,
  vote, share reveal).

The vote round ID is computed as a Poseidon hash over 8 Pallas base
field elements:

$$
\mathsf{vote\_round\_id} = \mathsf{Poseidon}(\mathsf{snapshot\_height},\; \mathsf{blockhash\_lo},\; \mathsf{blockhash\_hi},\; \mathsf{proposals\_hash\_lo},\; \mathsf{proposals\_hash\_hi},\; \mathsf{vote\_end\_time},\; \mathsf{nullifier\_imt\_root},\; \mathsf{nc\_root})
$$

where 32-byte inputs ($\mathsf{snapshot\_blockhash}$,
$\mathsf{proposals\_hash}$) are split into two 128-bit halves and
each half is embedded as a Pallas base field element ($\mathbb{F}_p$).
Poseidon is used because the vote round ID enters ZKP circuits as a
public input, requiring it to be a valid field element.

The round enters the **PENDING** state. The EA key ceremony
(see [^draft-ceremony]) runs automatically. On successful completion, the
round transitions to **ACTIVE** and the voting window opens.

### Round Lifecycle

1. **PENDING**: round created, awaiting EA key ceremony.
2. **ACTIVE**: ceremony complete, voting window open. Voters may delegate,
   vote, and submit shares (see [^draft-voting-protocol]).
3. **TALLYING**: `vote_end_time` has passed. Tally decryption runs
   automatically (see [^draft-ceremony]).
4. **FINALIZED**: tally published and verifiable.

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

**Cosmos SDK**: provides a mature BFT consensus engine (CometBFT),
validator lifecycle management (bonding, jailing for missed blocks or
missed ceremony acknowledgements, consensus power distribution), and a
transaction pipeline that can be extended with custom message types and
ante handlers for ZKP verification. The alternative — building a chain
from scratch — would duplicate well-tested consensus infrastructure.

**Funding equals voting power**: tying validator consensus power to
funding amount gives the bootstrap operator explicit control over power
distribution. An even distribution reduces the risk of a single validator
capturing consensus or disrupting the EA key ceremony.

**Automated validator onboarding**: `join.sh` eliminates manual
coordination between the bootstrap operator and joining validators. The
self-registration, admin-approval, and auto-bonding flow allows the
network to grow without requiring validators to build from source or
understand Cosmos SDK tooling.


# Reference implementation

[^ref-impl] — a Go and Rust implementation built on Cosmos SDK with
Halo 2 zero-knowledge proof circuits.


# References

[^BCP14]: [Information on BCP 14 — "RFC 2119: Key words for use in RFCs to Indicate Requirement Levels" and "RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words"](https://www.rfc-editor.org/info/bcp14)

[^zip-1016]: [ZIP 1016: Community and Coinholder Funding Model](zip-1016.md)

[^draft-balance-proof]: [Draft ZIP: Orchard Proof-of-Balance](draft-valargroup-orchard-balance-proof.md)

[^draft-voting-protocol]: [Draft ZIP: Zcash Shielded Voting Protocol](draft-valargroup-voting-protocol.md)

[^draft-ceremony]: [Draft ZIP: Election Authority Key Ceremony](draft-valargroup-ea-key-ceremony.md)

[^draft-pir]: [Draft ZIP: Private Information Retrieval for Nullifier Exclusion Proofs](draft-valargroup-gov-pir.md)

[^draft-onchain-voting]: [Draft ZIP: On-chain Accountable Voting](draft-ecc-onchain-accountable-voting.md)

[^join-sh]: [join.sh — validator join script](https://gist.github.com/greg0x/71bec808fbd02a7ef2a29b4386b8d842) (also distributed via [DigitalOcean Spaces](https://vote.fra1.digitaloceanspaces.com/join.sh) on each release)

[^ref-impl]: [z-cale/zally: Zcash Shielded Voting reference implementation](https://github.com/z-cale/zally)
