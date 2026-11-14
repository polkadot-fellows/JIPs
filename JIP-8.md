# JIP-8: Curve host calls for proof verification

This JIP proposes five host calls that give services native-speed elliptic-curve arithmetic for checking
zero-knowledge proofs and BLS-style commitments: multi-scalar multiplication and the pairing
check, on BN254 and on BLS12-381, and multi-scalar multiplication on Grumpkin, BN254's cycle
partner. Each takes its input in the encoding of the matching Ethereum precompile (Grumpkin's
by analogy with BN254's), with the same semantics, and is priced from end-to-end benchmarks
under an invariant stated in the Gray Paper's own terms. An Aztec-sized UltraHonk verification falls from
181M gas to 52M.

**Reference implementation:** [Aztec on JAM](https://abutlabs.github.io/aztec-on-jam/) —
Aztec's released software, settling on a six-validator JAM testnet whose client implements
these host calls.

## Motivation

### Privacy at launch

The Fellowship's required reading, *ĐApps: What Web 3.0 Looks Like* (G. Wood, 2014), states
the standard plainly: "Information that we assume to be private, we keep secret and never
reveal."

On a public ledger, zero-knowledge proofs are how that promise is kept: the user proves a
transaction is valid without revealing it, and validators check the proof without learning
anything. Today JAM cannot afford that check for the systems people actually use. Verifying
one Aztec transaction's client proof costs more than two work packages' entire budget, so
JAM's validators could check private transactions only indirectly, through a proof made by a
third party's prover. With this JIP, JAM's own validators verify every private transaction
themselves (*Reference implementation*), and every pairing-based check gets 3 to 16 times
cheaper. Adopted before JAM launches, it needs no activation fork: JAM can start life able to
run the strongest privacy systems deployed today.

### The cost of proofs today

A service that checks a proof spends most of its gas on curve arithmetic, which today is
metered PVM code: the PVM has no cryptographic host calls (GP 0.8.0, Appendix B, calls 0–27).
Measured on a six-validator PolkaJam v0.8.0 testnet, refine gas from PolkaJam's telemetry, each
row with its script in [aztec-on-jam](https://github.com/abutlabs/aztec-on-jam):

| Check | As PVM code today | With this JIP, projected | With this JIP, measured on a client | How the projection is obtained |
|---|---|---|---|---|
| UltraHonk, Aztec's epoch-proof shape (plain, Keccak transcript, 2^24, every gate type) | 181,129,375 | about 52.1M (3.5×) | 52,377,458 (3.5×) | Measured split: MSM (62 points) 71.1M, pairing 48.7M, other work 36.0M. Curve work re-priced at this JIP's constants |
| UltraHonk at 2^16 | 170,681,509 | about 47.9M (3.6×) | 48,129,253 (3.5×) | The same |
| One BN254 2-pair pairing check, end to end | 64,337,004 | 8,258,199 (7.8×) | 8,258,199, the price | The same function run as PVM code, and as the call |
| One BN254 128-point G1 MSM, end to end | 110,727,638 | 12,852,234 (8.6×) | 12,852,234, the price | The same |
| EIP-4844 KZG point evaluation | 199,453,527 | about 8.9M plus a SHA-256 (about 22×) | 12,111,139 (16.5×) | Composed from calls 103 and 104 (*Suggestions & examples*): the calls 8,879,921, the rest decompressing C and π and the SHA-256 |

The measured column is refine gas on lasair, a client that implements the calls (*Reference
implementation*). Gas is the protocol's: without the calls it equals PolkaJam v0.8.0's to the
unit on the same payloads.

A full refine package (5e9 gas) checks 95 Aztec-sized UltraHonk proofs instead of 27, and a tiny
one (1e9) 19 instead of 5 (measured at 2^16: 104 fit a full package and the 105th runs out).
Settling an Aztec epoch (its proof, its blob opening and 33 attestation signatures) falls from
about 813M gas to about 498M. The 433M of secp256k1 signature recovery that remains is outside
this JIP (*Rationale*).

One saving needs no JIP. The reference verifier's default curve hooks encode and decode every
point. Hooks that convert field by field verify the same 2^24 proof for 156.0M gas instead of
181.1M. Services should take that first. It leaves 77% of the verification in curve arithmetic,
which only a host call reaches.

## Rationale

- **Primitives, not verifiers.** Barretenberg changed its proof format at every major release
  (bb 3 → 5 → 6: challenge derivation, new subrelations). The curves did not change. A
  `verify_ultrahonk` call would need a protocol amendment per release, while an MSM and a
  pairing check serve every pairing-based proof system: UltraHonk, PLONK, Groth16, KZG
  commitments, and BLS signatures.
- **Whole operations, not field arithmetic.** Field-arithmetic host calls would not pay. Poseidon2,
  which is pure field arithmetic, already runs within f = 4.7 of native in the recompiled PVM,
  and a host-call boundary per multiplication costs more than that recovers. MSMs and pairings
  are where the PVM is furthest from native, because native libraries use algorithms (Pippenger
  windows, endomorphisms, optimised Miller loops) that a service would otherwise carry as code.
- **These curves.** BN254 is what deployed proof systems target: Noir and Barretenberg,
  Groth16 on Ethereum, Aztec. BLS12-381 is already in the GP (`beefy.tex`) and is the curve of
  Ethereum's KZG commitments and BLS signatures.
- **Grumpkin's MSM.** Grumpkin is BN254's cycle partner: its base field is BN254's scalar
  field. Aztec's client proofs (CHONK) end in an inner-product argument over it. A batch of
  them shares one 2^15-point Grumpkin MSM, which costs 9.0 to 9.8B gas as PVM code. That is more
  than a package, so no package can check even one batch without this call (measured with
  Barretenberg's own verifier compiled for the PVM). arkworks' MSM alone, as PVM code, costs
  11.06B at that size, and the call costs 1.93B. One client proof's whole check is 3.46B gas
  with the call and 11.36B without (*Reference implementation*). Grumpkin has no pairing, so
  the MSM is its only call.
- **Not in this JIP.** These each need their own measured case and their own proposal:
  - secp256k1 recovery: the largest remaining cost of settling Aztec, but Ethereum-specific;
  - Poseidon2 over BN254: a refine-side note tree already appends 11,350 notes a package
    without it;
  - a dedicated KZG point evaluation: it composes from 103 and 104 for less than a dedicated
    call would cost.
- **Separate indices,** one per operation, as JIP-1 uses. A selector argument adds a failure
  mode and saves nothing.

## Alternatives considered

1. **Better PVM code.** Field-wise curve hooks save ~14% of an UltraHonk check, and services
   should adopt them. Algorithmic gains are mostly already in the arkworks code services run.
   What remains is the PVM's own overhead on 254- and 381-bit arithmetic.
2. **An instruction or gas-table change.** A widening multiply-accumulate, add-with-carry, or a
   re-costed `mul_upper_u_u` (`pvm.tex`) would help every field, including the ones this JIP
   leaves out, and would cheapen the 36M of an UltraHonk check that no call touches. It is
   complementary to host calls, not a replacement. Measured by instruction class under the
   GP's cost model, it recovers little on the curves:
   - multiply instructions are 8 to 12% of the kernels' gas, and loads and stores of limbs are
     38 to 46%;
   - a widening multiply would save 3 to 6%, and an add-with-carry 7 to 10%: together about 11
     to 16%, against the calls' 7 to 25 times.
3. **A verifier call.** Rejected above: proof formats change faster than the protocol should.
4. **Larger refine budgets.** A bigger budget leaves each check's share of a package unchanged.
   The cost is relative to all other work on a core.

## Objections, and answers

The strongest reasons to reject this JIP, as we understand them, each with our answer.

1. **It writes particular cryptography into consensus, permanently.** JAM's bet is that the
   PVM is the one thing every validator must agree on; everything else is service code. Five
   calls make five libraries' behaviour a consensus rule for every client.
   *Answer:* the calls are primitives, not verifiers (*Rationale*), and the measured PVM route
   recovers 11 to 16% against the calls' 7 to 25 times (*Alternatives considered*, 1 and 2).
   Nothing in the protocol depends on a service using them: services probe and fall back to
   PVM code (*Activation and detection*), so one blob runs on clients with and without this
   JIP.

2. **Byte-for-byte agreement is hard, and a disagreement forks the chain.** Curve libraries
   differ at the edges — points off the curve, infinity, non-canonical encodings — and
   Ethereum's precompiles have produced consensus bugs there.
   *Answer:* the edges are specified (*Common rules*, *Determinism and consensus*) and tested:
   103 JAM-native vectors, produced by two independent implementations a curve and, for 105,
   by Barretenberg as a third; Ethereum's own EIP-196, EIP-197 and EIP-2537 vectors; and a
   second library over 10,000 random and adversarial inputs a call in the reference
   implementation. The cases where a client that reduces a coordinate would differ from one
   that rejects it each have a vector. We ask for a second client team to implement before acceptance.

3. **Gas prices are fragile.** Too low, a call becomes a way to overload validators; too high,
   nobody uses it. Ours come from one reference machine and today's libraries.
   *Answer:* each price is the reference machine's p99 time times a stated margin, a rule
   anyone can rerun (*Gas pricing*); the measured prices against a second machine are in
   *Open questions* 4. Like the Gray Paper's other gas constants, they can be amended. A price
   set too high breaks nothing: services fall back to PVM code.

4. **The curves may age badly.** BN254's security is estimated near 100 bits after the
   exTNFS attacks; proof systems are moving to small fields without pairings; no
   pairing-based scheme survives a quantum computer. Call 105 serves mainly one proof system.
   *Answer:* BLS12-381 and pairings are already in the Gray Paper (`beefy.tex`), so this adds
   no new class of assumption. BN254 is what deployed systems verify today: Noir and
   Barretenberg, Groth16, Aztec. A later JIP can deprecate a call the way this one adds it, at
   an activation timeslot. And the proposal is separable: 101–104 can be adopted without 105,
   at an activation slot of their own (the reference implementation already activates 105
   separately).

5. **It is not strictly needed.** Aztec's own design works on JAM without it: Aztec's prover
   folds every transaction into one epoch proof, and JAM checks that in 340.8M gas.
   *Answer:* true, and the reference implementation runs that way. What this JIP adds is the
   check JAM exists to offer: validators re-executing every transaction's verification
   themselves. Without call 105 that check is impossible, not slower: one Aztec client proof
   costs more than two packages' whole budget as PVM code, and fits comfortably in one with
   the call, measured and finalized on a six-validator net (*Rationale*, Grumpkin; *Reference
   implementation*).

6. **The evidence is narrow.** One client implements the calls, built by the proposal's
   authors; the measurements come from one team; it has had no independent review yet.
   *Answer:* every number has its script in the reference implementation, and the gas is the
   protocol's, so any client can reproduce it. The same service blobs run unchanged on
   PolkaJam without the calls. This objection is answered only by others: the PR asks client
   teams to review, and one to implement.

## Host-call specification

### Common rules

The calls are available in refine (Ψ_R) and accumulate (Ψ_A), and are added to their
host-call dispatch functions. They are not available in is-authorized (Ψ_I), which stays as the
GP defines it (`pvm_invocations.tex`, `eq:isauthorizedmutator`). In Ψ_I each returns `WHAT` at
Cgasunknown, as any unknown call does.

Every call takes φ7 = input offset and φ8 = input length, and evaluates in this order:

1. **Gas.** The call's gas is computed from φ8 alone, in unbounded integers, using the schedules
   in *Gas pricing*. If it exceeds the gas remaining, the invocation runs out of gas (∞). If φ8
   is not a multiple of the call's item size, the gas is that of ⌊φ8 / item⌋ items, at least the
   1-item price, and step 3 returns `HUH`. An MSM with φ8 = 0 is also priced as one item.
2. **Memory.** If μ_{φ7…φ7+φ8} is not readable, or the output range is not writable, the
   invocation panics (☇), as other host calls do on inaccessible memory. Since φ8 ≥ 2^32 can
   never be readable, this bounds φ8.
3. **Validation.** If the input is malformed (below), φ'7 = `HUH`, and nothing is written.
4. **Result.** Otherwise φ'7 = `OK`, and the result is written.

**Semantics are those of the named EIP** (for 105, which has none, those stated below, EIP-196's
by analogy), with this mapping: wherever the EIP's precompile fails,
the call returns `HUH`. One exception: an empty `bls12_381_pairing_check` input holds
(k = 0, φ'8 = 1), as EIP-197's does for BN254, where EIP-2537 refuses it. Where the precompile
returns output, the call returns `OK` and writes the
same bytes, or, for a pairing check, sets φ'8 to the precompile's 0 or 1. Scalars are any 32-byte
big-endian integer, as in the EIPs. Malformed means:
- a length that is not a whole number of items, or no items where an MSM needs at least one;
- a field element not below the field modulus, or, for BLS12-381, one whose 64-byte encoding
  has nonzero top 16 bytes;
- a point that is not on the curve and is not the all-zero encoding of infinity;
- a point outside the prime-order subgroup where the EIP requires the check. That is G2 for
  BN254; for BLS12-381, G1 and G2 in both calls (EIP-2537). BN254's G1 and Grumpkin have cofactor 1.

### 101. `bn254_g1_msm`

**Gas usage**: G_BN_MSM(n), n = ⌊φ8 / 96⌋. **Input registers**: φ7, φ8, and φ9 = output offset.

- input: n ≥ 1 items of (G1 point, 64 bytes: x ‖ y, each 32-byte big-endian, (0, 0) for
  infinity, as EIP-196; scalar, 32 bytes).
- `OK`: μ'_{φ9…φ9+64} = Σ scalar_i · point_i, in the same encoding.

EIP-196 defines ADD and MUL, not an MSM. With n = 1 this call is EIP-196's MUL. EIP-196's ADD is
an MSM of two points with scalars 1.

### 102. `bn254_pairing_check`

**Gas usage**: G_BN_PAIR(k), k = ⌊φ8 / 192⌋, and k = 1 for 0 < φ8 < 192 (*Common rules*). **Input registers**: φ7, φ8.

- input: k ≥ 0 items of (G1, 64 bytes; G2, 128 bytes: x_im ‖ x_re ‖ y_im ‖ y_re, the imaginary
  part first, as EIP-197).
- `OK` with φ'8 = 1 if ∏ e(G1_i, G2_i) = 1, else φ'8 = 0. With k = 0, φ'8 = 1 (EIP-197).

### 103. `bls12_381_g1_msm`

**Gas usage**: G_BLS_MSM(n), n = ⌊φ8 / 160⌋. **Input registers**: φ7, φ8, φ9 = output offset.

- input: n ≥ 1 items of (G1 point, 128 bytes, EIP-2537: x ‖ y, each 64-byte big-endian with
  the top 16 bytes zero, all zero for infinity; scalar, 32 bytes).
- `OK`: μ'_{φ9…φ9+128} = Σ scalar_i · point_i, in the same encoding.

### 104. `bls12_381_pairing_check`

**Gas usage**: G_BLS_PAIR(k), k = ⌊φ8 / 384⌋, and k = 1 for 0 < φ8 < 384 (*Common rules*). **Input registers**: φ7, φ8.

- input: k ≥ 0 items of (G1, 128 bytes; G2, 256 bytes: x_c0 ‖ x_c1 ‖ y_c0 ‖ y_c1, as EIP-2537).
- `OK` with φ'8 = 1 if ∏ e(G1_i, G2_i) = 1, else φ'8 = 0. With k = 0, φ'8 = 1.

### 105. `grumpkin_g1_msm`

**Gas usage**: G_GK_MSM(n), n = ⌊φ8 / 96⌋. **Input registers**: φ7, φ8, φ9 = output offset.

- The curve: Grumpkin, y² = x³ − 17 over F_r, where r is BN254's group order (its scalar
  field). Its group has prime order p, BN254's base-field modulus, so the cofactor is 1.
  −17 is not a square mod r, so no point has x = 0 and (0, 0) is free to encode infinity.
- input: n ≥ 1 items of (point, 64 bytes: x ‖ y, each 32-byte big-endian below r, (0, 0) for
  infinity; scalar, 32 bytes, any big-endian integer, reduced mod p).
- `OK`: μ'_{φ9…φ9+64} = Σ scalar_i · point_i, in the same encoding.

No EIP defines Grumpkin. The encoding and checks are EIP-196's G1 with Grumpkin's field and
equation. Barretenberg writes a point the same way, except that it marks infinity in x's top
bit; a caller converts that to (0, 0).

### Side-effects

None beyond the output memory and φ'7, φ'8. No state is read or written. The input is read
in full before any output is written, so an MSM's output range may overlap its input.

## Gas pricing

**The invariant.** A call charges at least what the GP's own sizing charges for the same time.
The GP sizes a core as "a regular CPU core executing at somewhere between 25% and 50% of
regular speed" over the six-second slot (`discussion.tex:77`). It meters a full package at 5e9
gas, so the conservative end is 5e9 gas for 1.5 s of a core: R_GP = 10/3 gas per nanosecond.
A call's gas is

    G = ⌈t × R_GP × m⌉,  with m = 2

where t is the call's p99 time on one thread of the reference machine with the reference
implementation, the worst over the benchmark's input cases. m covers slower clients and
hardware. The price depends on the GP's sizing and the benchmark, not on how fast any client's
PVM recompiler runs.

**The benchmark** (in the reference implementation) times each call end to end from its encoded input,
with every check above. Each size runs on random group elements and adversarial cases: one base
repeated, P with −P, half the points at infinity, scalars r − 1 and 2^256 − 1, a pair repeated,
and a G2 point outside the subgroup last. Two implementations a curve must agree on every
input: arkworks and substrate-bn for BN254, blst and arkworks for BLS12-381. The faster mature
one is the reference, arkworks for BN254 and blst (single-threaded) for BLS12-381.

**The constants below** come from one run on GitHub's 4-vCPU x86-64 runner (AMD EPYC 9V45),
where the implementations agreed on all 144 inputs. They are provisional (*Open questions*, 1).

**Pairing checks**, an affine bound at or above every measured k = 1, 2, 4, 8:

    G_BN_PAIR(k)  = 4,196,895 + 2,030,652 · k
    G_BLS_PAIR(k) = 1,827,334 + 1,727,240 · k

**MSMs**, a table. Between listed sizes, gas is linear in n, rounded up: for listed sizes
a < n < b, G(n) = G(a) + ⌈(G(b) − G(a)) · (n − a) / (b − a)⌉. Past the last listed size, each
point adds the listed per-point constant (the last segment's slope, rounded up). Each entry is
the largest at its size or any smaller size. [`vectors/jip8-gas.jsonl`](https://github.com/abutlabs/aztec-on-jam/blob/6f7761c1f4a033b3fe02cfc9c48ab1d025133888/docs/jip/vectors/jip8-gas.jsonl) pins these rules: the gas
of each call at listed, interpolated, partial, zero and past-the-table lengths.

| n | G_BN_MSM(n) | G_BLS_MSM(n) |
|---|---|---|
| 1 | 703,667 | 842,540 |
| 2 | 879,660 | 1,198,814 |
| 4 | 1,135,920 | 5,997,400 |
| 8 | 3,178,140 | 5,997,400 |
| 16 | 3,178,140 | 14,589,460 |
| 32 | 6,411,967 | 16,341,307 |
| 64 | 8,007,640 | 30,107,254 |
| 128 | 12,852,234 | 66,119,300 |
| 256 | 30,809,314 | 82,714,587 |
| 512 | 39,471,280 | 208,961,727 |
| each point past 512 | 33,836 | 493,153 |

**Grumpkin's MSM** (105) was priced the same way, on GitHub's 4-vCPU x86-64 runner (AMD EPYC
9V74). arkworks and halo2curves agreed on all 96 inputs;
arkworks, faster at every size but one, is the reference. Its table runs to 2^15 points, the
size of Aztec's IPA, because Pippenger's cost a point falls with n and a slope from 512 would
overcharge there:

| n | G_GK_MSM(n) | n | G_GK_MSM(n) |
|---|---|---|---|
| 1 | 1,091,714 | 512 | 58,925,247 |
| 2 | 1,188,054 | 1,024 | 97,223,727 |
| 4 | 1,604,020 | 2,048 | 169,894,200 |
| 8 | 2,386,260 | 4,096 | 304,263,534 |
| 16 | 3,778,494 | 8,192 | 569,925,254 |
| 32 | 6,151,940 | 16,384 | 1,034,632,760 |
| 64 | 10,875,667 | 32,768 | 1,926,735,634 |
| 128 | 17,737,614 | each point past 32,768 | 54,450 |
| 256 | 31,408,427 | | |

The 2^15-point call, 1.93B gas, replaces 9.0 to 9.8B of PVM code.

Each BN254 call costs less than its own computation as PVM code: a 1-point MSM is 703,667 as a
call against 9,378,971 as PVM code, and across the sizes measured the saving is 7.2–13.3×
in the reference implementation's profile.

## Activation and detection

The calls are a protocol change: a GP amendment adopted at an activation timeslot T_A, not an
optional extension. Every guarantor and auditor of a package must agree on a call's result and
gas.

- **In Ψ_R,** the calls are available if and only if the work-package's lookup-anchor timeslot
  ((p_context)_lookupanchortime) is at least T_A. Refine already binds its view of state to that
  slot (`pvm_invocations.tex`, the Refine invocation and `historical_lookup`), so an execution
  before activation and its audit after it see the same answer.
- **In Ψ_A,** they are available if and only if the block's timeslot is at least T_A.
- **Before activation,** each index is an unknown host call: `WHAT`, for Cgasunknown = 1000 gas.
- **Detection.** A service probes with `bn254_pairing_check` and φ8 = 0. Before activation it
  gets `WHAT`. After, it gets `OK` with φ'8 = 1, for G_BN_PAIR(0) = 4,196,895 gas.
- **Indices.** 101–105 are this JIP's. Later cryptographic calls would take subsequent
  indices, each by its own JIP.

## Determinism and consensus

Implementations must agree byte for byte on outputs, on `HUH`, and on gas. The specification
above is normative on its own: curve parameters, encodings, canonicality, subgroup membership
and infinity. It does not depend on any library. Before activation, at least two independent
implementations of each call are to be fuzzed against each other and against the vectors.
BN254 has several mature independent implementations (arkworks, substrate-bn, Barretenberg's
`ecc`, gnark-crypto, mcl), as BLS12-381 does (blst, arkworks, and others), and Grumpkin does
(arkworks, halo2curves, Barretenberg). Clients link one of
them rather than write pairing code from scratch. BN254 is a new dependency for a client that
implements only what GP 0.8.0 needs. BLS12-381 already is one, for BEEFY (`beefy.tex`).

## Compatibility

New indices only. Existing services are unaffected. A service can keep its PVM implementation
and switch on the probe's result.

## Test vectors

- **JAM-native:** [`vectors/jip8-core.jsonl`](https://github.com/abutlabs/aztec-on-jam/blob/6f7761c1f4a033b3fe02cfc9c48ab1d025133888/docs/jip/vectors/jip8-core.jsonl), 103 vectors. Each gives the call, the case, the
  input, the result code, and the output or φ'8, and both implementations of each curve
  produce them. They cover the benchmark's cases at n ≤ 4 and k ≤ 4, k = 0, and the malformed
  inputs above: a wrong length, x equal to the modulus (and for 105, y), a point off the
  curve, nonzero top bytes in an EIP-2537 field, and a G1 point outside the subgroup. For each
  MSM, a valid point with the field modulus added to x, and to y: a client that reduces a
  coordinate instead of rejecting it answers `OK` there. 105's 25 are also produced by
  Barretenberg's own Grumpkin, a third implementation.
- **Gas:** [`vectors/jip8-gas.jsonl`](https://github.com/abutlabs/aztec-on-jam/blob/6f7761c1f4a033b3fe02cfc9c48ab1d025133888/docs/jip/vectors/jip8-gas.jsonl), each call's gas at the lengths above, computed from this
  text's schedules alone.
- **Ethereum's,** where the semantics coincide (none exist for Grumpkin):
  - EIP-197's pairing vectors, for 102;
  - EIP-196's MUL vectors, for 101 with n = 1;
  - EIP-2537's MSM and pairing vectors, for 103 and 104.

## Reference implementation

The proposal is implemented and running end to end ([Aztec on JAM](https://abutlabs.github.io/aztec-on-jam/));
the code and every measurement's script are in [the repository](https://github.com/abutlabs/aztec-on-jam).

- **A client.** lasair, an independent GP 0.8.0 client, implements all five calls behind
  activation flags (*Activation and detection*), and passes the JAM-native vectors in its gate.
- **Services that adopt them.** The services detect the calls and fall back to PVM
  code where a net lacks them, so one service blob runs on clients with and without this JIP.
  The same blobs run unchanged on PolkaJam v0.8.0, which does not implement it.
- **Aztec on JAM, live.** Aztec v6's released software — its wallet libraries, sequencer node
  and prover — settles on six lasair validators with the calls on. Measured there, refine gas:

  | Check in refine | Without the calls | With the calls |
  |---|---|---|
  | A checkpoint: its header, blob and KZG checks | 251,459,989 | 64,057,914 |
  | One Aztec client proof, by Barretenberg's own verifier (a token transfer) | 11.36B | 3,456,104,321 |
  | One Aztec client proof (a 7-layer storage proof, the heaviest flow) | over a package | 3,729,308,707 |
  | The same transfer with one byte flipped: refused | — | 625,503,052 |

  Both client proofs and the refusal were accumulated and finalized by GRANDPA; every report
  was audited by more than two thirds of the validators, re-executing refine.

## Suggestions & examples

**KZG point evaluation (EIP-4844) from 103 and 104.** The proof π shows a commitment C opens to y
at z if e(π, [s − z]G2) = e(C − [y]G1, G2). Equivalently, with no G2 scalar multiplication:

    e(C − [y]G1 + [z]π, −G2) · e(π, [s]G2) = 1

- **The work:** one 3-point `bls12_381_g1_msm` (points C, G1, π; scalars 1, −y, z), then one
  2-pair `bls12_381_pairing_check`. That is about 3.6M + 5.3M gas, against 199,453,527 as PVM
  code today.
- **What the service supplies:** the versioned-hash check (0x01 ‖ sha256(C)[1..]) and [s]G2,
  from Ethereum's KZG ceremony, which it holds as data. Neither enters the protocol.
- **Vectors:** c-kzg-4844's 122 `verify_kzg_proof` vectors test the composition.

**UltraHonk.** zkVerify's verifier routes every curve operation through its `CurveHooks` trait
(`ark-bn254-ext`). A service adopts calls 101 and 102 by implementing that one trait.

## Process

A JIP standardises beside the GP; this one changes it. The JIP is the rationale and the
specification. A companion diff to the GP would add the five functions to Appendix B, the
constants to `definitions.tex`, and the dispatch entries to Ψ_R's and Ψ_A's host-call
functions. It is for the GP's maintainers, who decide the activation and the version.

## Open questions

1. **The reference machine and m.** One shared VM's p99 is noisy: the BLS12-381 MSM's 4-point
   time exceeds its 8-point one. The constants should come from pinned-frequency bare metal,
   more than one machine, with m chosen against the slowest client the network intends to
   support.
2. **Per-pair G2 cost.** Proof systems pair against fixed G2 points, so the per-pair price
   includes a subgroup check and line computation they don't need. A fixed-point variant would
   put a particular trusted setup into the protocol, so this JIP does not propose one.
3. **The instruction-level route** (*Alternatives considered*, 2): measured on the curve
   kernels, a widening multiply and an add-with-carry together save about 11 to 16% of their
   gas. They would help the uncovered 36M and the curves this JIP leaves out by about as much.

**Found implementing the calls in a client** (lasair, all five; tested against the JIP's
vectors, go-ethereum v1.14.12's EIP-196/197 and EIP-2537 vectors, and a second library over
10,000 random and adversarial inputs a call):

4. **The price against a second machine** (1, measured). On one Apple M-series Mac (arm64),
   through lasair, the price ÷ (p99 × 10/3) against the margin of 2:
   - `bn254_pairing_check`: 1.93–2.43;
   - `bn254_g1_msm`: 1.4–2.1 up to 512 points, and about 1.0 from 2,048 to the largest n 5e9
     gas pays. Past 512 the table's slope (33,836 a point) is the 256→512 segment's, while
     this machine's per-point time stays about twice the x86 runner's;
   - `bls12_381_g1_msm`: mostly ≥ 2;
   - `bls12_381_pairing_check`: 1.3–1.7;
   - `grumpkin_g1_msm` is the least stable: against p99, 1.25–1.65 from 8,192 points up, but
     0.6–1.9 below that, with run-to-run tails as low as 0.2; at 2^15 points this machine is
     about 1.2× slower than the runner's p99 allows, as for `bn254_g1_msm`. The full runs are
     in the repository.
   The prices are the x86 runner's p99 × 2 with no further headroom, so a machine slower than
   the runner on any one operation falls under the margin there.

**Found building the reference implementation:**

5. **Aztec's public functions in refine.** Refine checks each transaction's private part: its
   client proof. Its public part (calls to contracts' public functions, compiled to Aztec's AVM
   bytecode) is executed by Aztec's sequencer and proved by Aztec's AVM prover, and JAM sees it
   only through the epoch proof. In the demo, deploying a token, minting, and moving funds
   between private and public balances all have public parts. Two routes, neither measured:
   - **Re-execute:** Barretenberg's AVM simulator (C++, about 20,000 lines) built for the PVM,
     re-running each transaction's public calls from the hints Aztec's prover collects, every
     hint checked against the state trees. Refine does the work; its gas, and whether it fits
     the PVM's code limit beside the verifier, are unknown.
   - **Verify Aztec's AVM proof:** a BN254 proof with KZG commitments, 14,532 fields (about
     465 KB), whose verifier would use calls 101 and 102. Refine checks Aztec's prover instead
     of executing.
   Which route a JAM rollup should take, and so whether AVM proof verification belongs among
   this JIP's examples, is open.
