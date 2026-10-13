# JIP-6: Compact state proofs

A compact Merkle proof for the JAM state trie. Together with the state entries it accompanies, it
proves those entries, and the absence of other keys, against a trusted state root.

## Motivation

A verifier that trusts a block's state root may need to check a small set of state values without
trusting the prover that provides them. Existing whole-node proofs, such as the CE 129 range proof,
work well for state synchronisation but are inefficient for a few unrelated keys. They ship
redundant branch data, repeat common trie paths across requests, and require rebuilding a partial
trie during verification.

The proof defined here is designed for this smaller, selective access pattern. The requested keys
and values travel beside the proof, not inside it. The proof holds only the trie structure, the
hashes of the subtrees it does not open, and the few leaves the verifier needs but did not ask
for. The verifier rebuilds every other subtree from the requested keys and values.

One proof covers multiple keys and ranges, and proves both presence and absence. It is verified in
a single pass with a stack. For a range, its size grows with the depth of the trie, not with the
number of keys in the range.

The proof format is independent of the transport that carries it.

## Terms

The state trie is defined by the state Merklization appendix of the Gray Paper (GP). This
document uses the following terms for the state trie:

- The _path to a node_ is the sequence of bits walked from the root to reach it: `0` to the left
  child, `1` to the right child. The path to the root is empty, and the _depth of a node_ is the
  length of its path.
- The _hash of a node_ is the blake2b-256 hash of the node's encoding as per the GP. The hash of an
  empty subtree is the zero hash. The GP calls this hash the identity of the (sub-)trie; this
  document calls it the hash.
- A key _lies under_ a node when the path to the node is a prefix of the key, that is, when the key
  belongs to the subtree rooted at the node. The term applies to any key of 248 bits, whether or
  not it is in the state.
- _Rebuilding a subtree_ at depth $d$ from a set of (key, value) pairs means applying the GP's
  function $M$ to those pairs from bit $d$ onwards. The leaf node of a pair is the GP's $L(k, v)$.
  The recursion is at most $248 - d$ levels deep; an implementation may use an explicit stack.

This document uses the following terms for proofs:

- A _query_ is what a verifier asks a prover to prove: a set of listed keys and a set of key
  ranges. Its exact form and constraints are given under [Proving a query](#proving-a-query).
- An _entry_ is a (key, value) pair of the state. The verifier's entries are the pairs it received
  with the proof together with those it already held. Their keys are unique and ascending.
- The _claims_ of a proof are what it proves, as given under [Proof subtree](#proof-subtree). A
  query names the claims a verifier wants.

The prover holds the state. It answers a query with two things: the entries of the query, and a
proof that claims at least what the query asks for. The proof never carries the key or the value of
an entry. A listed key that is absent from the state has no entry; its absence follows from the
proof.

The verifier holds a trusted state root. It checks the entries against that root with the proof, as
defined under [Verification](#verification). This document defines the entries as an input of the
verifier, not how they are encoded. A transport carrying the proof may let the prover omit entries
that the verifier already holds.

## Proof subtree

A proof subtree is a part of the state trie that contains the root and, for every other node in it,
the node's parent. Each node of a proof subtree has one of four tags:

| Tag | Name | Meaning | The proof ships |
|---|---|---|---|
| `B` | expanded branch | a branch whose two children are in the proof subtree | nothing |
| `H` | hashed subtree | a non-empty subtree that is not opened | its hash |
| `K` | known subtree | a subtree under which every state key is the key of an entry | nothing |
| `R` | raw leaf | a leaf whose key is not the key of an entry | key suffix; value or value hash |

A `K` under which no state key lies is an empty subtree. The verifier computes the hash of a `K` by
rebuilding it from the entries under it.

Together with the entries, a proof subtree proves that:

- every entry is in the state: its key is present and holds its value;
- the key of every `R` is present and holds its value, or a value with the shipped hash;
- a key under a `K` is absent from the state unless it is the key of an entry;
- a key under an `R` is absent from the state unless it is the key of the `R`.

It proves nothing about a key under an `H`.

## Encoding

The encoded proof consists of a version octet followed by four sections, with no explicit section
lengths:

```
proof      = version tags hashes raw_keys raw_values
version    = 0x00
tags       = subtree, then zero bits up to the next octet boundary
subtree    = B subtree subtree | H | K | R
hashes     = 32 octets per H, in tag order
raw_keys   = for each R, in tag order, the last 248 - d bits of its key, d being its depth;
             concatenated most significant bit first, then zero bits up to the next octet boundary
raw_values = for each R, in tag order: one form octet, then its data
```

The `version` octet identifies the encoding defined here, version 0. Later revisions of this
document may define further versions.

The `tags` section lists the nodes of the proof subtree in pre-order: a `B` is followed by the tags
of its left subtree and then those of its right subtree. This order is called tag order, and the
other sections follow it.

Each tag is two bits:

| Tag | Code |
|---|---|
| `B` | `00` |
| `H` | `01` |
| `K` | `10` |
| `R` | `11` |

Tags are packed from the most significant bits of each octet down. The first tag thus occupies bits
7 and 6 of the first octet of the `tags` section.

The `tags` section starts with one subtree open. Each `H`, `K` or `R` closes one, while each `B`
closes one and opens two more. The section ends when none remain open.

The `hashes` section holds one 32-octet hash per `H`, in tag order. If the node an `H` stands for
is a left child, its hash is encoded as the GP's branch encoding stores it: with bit 7 of octet 0
cleared.

The `raw_keys` section holds, for each `R` in tag order, the last $248 - d$ bits of its key, $d$
being the depth of the `R`. The first $d$ bits of the key are the path to the `R`, so the key is
that path followed by these bits. The suffixes are concatenated without padding between them. The
section is padded with zero bits to an octet boundary at its end only. With no `R`, the section is
empty.

The `raw_values` section holds, for each `R` in tag order, a form octet and then its data. The form
octet says what the data is:

| Form octet | Data |
|---|---|
| `0` to `32` | the value itself, of that many octets |
| `33` | the 32-octet hash of the value, which is longer than 32 octets |

These are the two contents of a GP leaf node: the value itself when it has at most 32 octets, the
hash of the value otherwise. Form octets from `34` to `255` are invalid.

The section lengths follow from the tags. The `hashes` section holds 32 octets per `H`. The
`raw_keys` section holds $\lceil \sum (248 - d) / 8 \rceil$ octets, the sum running over the `R`
tags. The `raw_values` section takes the rest of the proof, and its form octets delimit it.

## Canonical form

A verifier must report a proof whose version octet is not 0 as having an unknown version. This is a
distinct error from an invalid proof, so that a verifier can tell a later version from a corrupt
proof.

A verifier must reject a proof of version 0 if any of the following holds:

1. The proof is empty, or the `tags` section ends before the subtree is complete.
2. A padding bit of the `tags` section is set.
3. A `B` is at depth 248 or deeper.
4. A `B` has one of these pairs of children, in either order:
   - two `H`;
   - two `K`;
   - a `K` under which no entry lies, and an `R`.
5. An `H` carries the zero hash.
6. An `H` that is a left child has bit 7 of octet 0 of its hash set.
7. A padding bit of the `raw_keys` section is set.
8. A form octet is greater than 33.
9. The key of an `R` is the key of an entry.
10. A section is shorter than its contents require, or octets remain after the data of the last
    `R`.
11. The verifier's entries are not in strictly ascending order of key.
12. An entry lies under no `K`.
13. The hash of the root of the proof subtree differs from the trusted state root.

Rule 4 rejects branches that the rules under [Construction](#construction) never produce:

- two `H`: construction rule 3 opens a branch only for a key that lies under one of its children,
  and that child is then not an `H`;
- two `K`: construction rule 2 makes such a branch a `K` itself;
- an empty `K` and an `R`: the state trie has no such branch, because a node with one key under it
  is a leaf.

These rules give every proof subtree exactly one encoding. The verifier accepts any canonically
encoded proof subtree whose root hash is the state root.

The verifier does not check that the proof subtree is the one the construction rules give for a
query. A proof may therefore open more of the trie than the query needs, which proves more keys,
never fewer. The checks a verifier makes against its own query are listed under
[Checking a reply against the query](#checking-a-reply-against-the-query).

A verifier may bound the size of the proofs it accepts.

## Verification

The verifier takes three inputs: the proof, a trusted state root, and its entries. While reading
the tags in order, it keeps the path to the current node and a stack of open branches, those whose
right child is still to come. Each slot of the stack is empty or holds a left child's tag and hash.

Reading a `B` opens a branch: a slot is pushed and the left child is read next. When a node
completes and the top slot is empty, the node is a left child. Its tag and hash are stored in the
slot, and the right child is read next. When a node completes and the top slot already holds a left
child, the node is the right child. The branch's hash is computed from the two, the slot is popped,
and the branch is itself a completed node, to be handled the same way by the slot below. The proof
is valid when the final completed node is the root of the proof subtree and its hash equals the
state root:

    stack = empty, path = empty
    loop:
        tag = next tag
        match tag:
            B: push an empty slot; append 0 to path; continue
            H: hash = next hash; path is not covered
            K: S = the entries whose keys lie under path
               hash = rebuild of S at depth (length of path)
               the entries of S are consumed
            R: key = path followed by the next 248 - (length of path) bits of raw_keys
               (form, data) = next form octet and its data from raw_values
               hash = hash of the leaf node of key and data
               key is present
        loop:
            if stack is empty:
                require hash = state root; done
            if top of stack is empty:
                top of stack = (tag, hash); set the last bit of path to 1; break
            (left tag, left hash) = pop stack; remove the last bit of path
            check rule 4 on (left tag, tag)
            hash = hash of the branch node with children left hash and hash; tag = B
    require every entry is consumed
    require every section is consumed exactly

The pseudo-code omits the other canonical-form rules, which are checked as each tag, hash, key and
value is read.

For a `K`, the entries whose keys lie under the path form a contiguous part of the ascending
entries, so the verifier can take them in order.

For an `R`, the leaf node depends on the form octet:

- form `0` to `32`: the leaf node embeds the data as the value;
- form `33`: the leaf node holds the data as the hash of the value.

The result of verification is the set of present keys and the set of paths of the `H` nodes. The
present keys are:

- the keys of the entries, with their values, and
- the keys of the `R` leaves, each with its value or the hash of its value.

A key is then:

- Present, if it is a present key.
- Absent, if it is not present and its path in the proof subtree ends at a `K` or at an `R` holding
  a different key.
- Not covered, otherwise. Its path leaves the proof subtree through an `H`.

A verifier must treat a key that is not covered as a failed proof, never as an absent key.

## Proving a query

A query consists of:

- Listed keys: a strictly ascending sequence of state keys.
- Ranges: a sequence of pairs `[start, end]` of prefix bounds, each bound 0 to 31 octets long.

A range contains every key from `start` padded to 31 octets with `0x00` up to `end` padded to 31
octets with `0xFF`, both inclusive. `[p, p]` is thus every key starting with `p`, and `[empty,
empty]` is every key.

A query must satisfy the following, with bounds compared after padding:

- each range's `start` does not exceed its `end`;
- each range's `start` exceeds the previous range's `end`;
- no listed key lies within a range.

A prover must reject a query that violates these constraints.

The entries of a query are the pairs of the state whose key is a listed key or lies within a range.
The prover builds the proof subtree of the query with the rules under [Construction](#construction);
its claims then include the presence of every entry, the absence of every listed key that is not in
the state, and the contents of every range.

### Construction

The proof subtree follows only the paths relevant to the query. Starting at the root, the prover
opens a branch into both children only when a listed key or a key within a range lies under it.
Every other node it reaches gets a single tag, `K`, `R` or `H`, and is not opened further. For each
node, the first of the following rules that applies gives the tag:

1. If no state key lies under the node, the tag is `K`.
2. If every state key under the node is the key of an entry, the tag is `K`.
3. If a listed key lies under the node, or a key within a range lies under the node:
   - if the node is a leaf, the tag is `R`;
   - if the node is a branch, the tag is `B`, and the prover applies these rules to both children.
4. Otherwise, the tag is `H`.

Rule 3 tests keys of the whole key space: a listed key or a key within a range counts whether or
not it is in the state.

The following consequences hold:

- For an empty state, the proof subtree is a single `K`.
- For a non-empty state and a query with no listed keys and no ranges, the proof subtree is a
  single `H` carrying the state root.
- For a state with one key that is the key of an entry, the proof subtree is a single `K`.
- For a query of one listed key that is present, the path of the key is a chain of `B` tags that
  ends at a `K` at the key's leaf. Each sibling along the path is an `H`, or a `K` if it is empty.
  Vector 1 shows this as `B H B H B K H`.
- The path of a listed key that is absent from the state ends at one of:
  - an empty `K`;
  - a non-empty `K`, whose state keys are entries of other listed keys or of ranges;
  - an `R` holding a different key.
- A subtree whose state keys all lie within ranges is a single `K`, however many keys it holds.
- A leaf whose key lies outside every range is an `R` when a key within a range lies under it.
- Two listed keys whose paths end at the same leaf need no special rule. The leaf is a `K` if its
  key is the key of an entry, and an `R` otherwise.

### Checking a reply against the query

A verifier that checks a proof against a query it made must also check that the claims of the proof
cover the query:

- no `H` stands for a node under which a listed key lies;
- no `H` stands for a node under which a key within a range lies;
- no `R` holds a listed key or a key within a range.

The first check is the lookup of each listed key. The second is needed because the verifier cannot
look up the keys of a range one by one: such an `H` could hide keys of the range, which would then
be neither present nor absent. The third is needed because such an `R` would prove a requested key
without delivering it as an entry, and might ship only the hash of its value.

## Appendix A: Prover contract

A prover that serves the proof over a transport must follow this appendix. A verifier that only
checks proofs does not need it.

A transport, such as the `stateProof` method of JIP-2 or a CE message, carries queries to a prover
and replies back. This appendix defines the exchange independently of the transport, so that every
transport limits and cuts replies the same way.

The request and the reply have this shape:

    request = query, limit
    query   = listed keys, ranges       as defined under Proving a query
    limit   = a number of octets, or none
    reply   = proof, entries, status
    status  = complete | cut at a key

A transport maps the request and the reply onto its own messages. It must carry the status. It may
let the verifier ask for the entries to be omitted, may clamp the limit, and may reject a query it
considers too large.

### Charged keys

The prover chooses the cut among the charged keys of the query: its listed keys and the keys of
the state that lie within its ranges. Since the listed keys are sorted, the ranges are sorted and
disjoint, and no listed key lies within a range, the charged keys form one ascending sequence. A
range containing no key of the state adds no charged keys.

### Size of a reply

The size of a reply is the number of octets of its encoded proof plus the octets of its entries,
$31$ plus the length of the value for each entry. The entries count only when the transport sends
them.

### Cutting a reply

The query cut at a key $k$ is a shorter query derived from the original:

- it keeps the listed keys that do not exceed $k$;
- it removes every range whose padded `start` exceeds $k$;
- it ends every remaining range whose padded `end` exceeds $k$ at $k$.

The prover must choose a cut key $k$ among the charged keys such that the size of the reply for
the query cut at $k$ does not exceed the limit. It may choose an earlier charged key than the last
one for which this holds. It must include the first charged key even if the reply for the query
cut at it exceeds the limit.

If the cut key is the last charged key, the reply is that of the whole query and the status is
complete. Otherwise the status is cut at $k$. With no limit, the prover includes every charged key
and the status is complete.

A cut reply is not a special form of proof. Its entries and its proof are those of the query cut at
the cut key, and the verifier checks it as the reply to that query. A verifier that holds entries
of its own must use only those of the cut query.

The cut cannot be inferred from the proof. A cut proof may still cover the whole query, because an
absent listed key can open the region the cut removed. The verifier must take the cut from the
status, not from the proof.

A verifier may continue with a new query made of the listed keys that exceed the cut key and the
parts of the ranges that lie after it.

A prover can find the cut in one walk over the charged keys, keeping the size of the reply for the
query cut at the current key, and then build the proof once.

## Appendix B: Test vectors

These vectors use a state of five keys, named by their first three bits: `000`, `001`, `100`, `110`
and `111`. Octet 0 of each key is those three bits followed by `11010`, and octets 1 to 30 are
`0x5A`; key `110` is thus `0xDA` followed by thirty `0x5A` octets. The value under each key is the
nine octets of the ASCII string `value ` followed by the key's three bits, e.g. `value 110`. The
state root is `9b9760b1a0bbb685177ad5ddd98b1ae446307255aca511c22e4c2e778cec425f`.

The hashes used below, as they appear in the `hashes` section, are those of the leaves holding
keys `000`, `100`, `110` and `111`, and of the subtrees under the paths `0` (keys `000` and `001`),
`00` (the same two keys) and `11` (keys `110` and `111`):

    leaf 000     40f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea6
    leaf 100     6b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df
    leaf 110     2af306b851cf153d7c58cf43682fed3b75f2cdd7dfc9a547dcfe990ccaea77ab
    leaf 111     883a0c1f05cbac875dae2e53c5c15dfad106d7272bf3cb07015aae7e8656f7f9
    subtree 0    50918ec4ad4465ee3baa8a272129bd2534ad911f63d22abff4716baa78bc97c3
    subtree 00   41dd8fddabce96f7a9e0b737297a41b55924bb4e5e7c7e2f0772ef948a60debd
    subtree 11   4edb3501f7717e134d548ba9825c4e98a8fe174ec15b2d51667d3688dca3f7e9

Subtree `0` is a left child, so its hash appears with bit 7 of octet 0 cleared; its full hash
starts with `d0`. The other hashes have that bit clear already.

Each proof is shown in hex, followed by its version octet, tags, hashes, raw keys and raw values,
separated by `|`. The hex is the concatenation of these components.

1. Listed key `110`. Entries: `110`. Tags `B H B H B K H`. Hashes: subtree 0, leaf 100, leaf 111.
   No raw leaves.

       00112450918ec4ad4465ee3baa8a272129bd2534ad911f63d22abff4716baa78bc97c36b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df883a0c1f05cbac875dae2e53c5c15dfad106d7272bf3cb07015aae7e8656f7f9

   `00` | `1124` = 7 tags and 2 padding bits | subtree 0, leaf 100, leaf 111 | none | none

2. Listed keys `001` and `111`. Entries: `001`, `111`. Tags `B B B H K K B H B H K`. Hashes: leaf
   000, leaf 100, leaf 110. No raw leaves.

       0001a11840f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea66b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df2af306b851cf153d7c58cf43682fed3b75f2cdd7dfc9a547dcfe990ccaea77ab

   `00` | `01a118` = 11 tags and 2 padding bits | leaf 000, leaf 100, leaf 110 | none | none

   The `K` at path `001` rebuilds to the leaf of `001`, and the `K` at path `01` is an empty
   subtree.

3. Range `[001, 110]`, both bounds being full keys. Entries: `001`, `100`, `110`. Tags
   `B B B H K K B K B K H`. Hashes: leaf 000, leaf 111. No raw leaves.

       0001a22440f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea6883a0c1f05cbac875dae2e53c5c15dfad106d7272bf3cb07015aae7e8656f7f9

   `00` | `01a224` = 11 tags and 2 padding bits | leaf 000, leaf 111 | none | none

4. Range `[100, 111]`, both bounds being full keys. Its state keys, `100`, `110` and `111`, are
   exactly the keys under path `1`. Entries: `100`, `110`, `111`. Tags `B H K`. Hashes: subtree 0.
   No raw leaves.

       001850918ec4ad4465ee3baa8a272129bd2534ad911f63d22abff4716baa78bc97c3

   `00` | `18` = 3 tags and 2 padding bits | subtree 0 | none | none

   The whole subtree under path `1` is a single `K`. The proof would stay this size for any number
   of keys under that path.

5. Listed keys `010` and `101`, both absent. No entries. Tags `B B H K B R H`. Hashes: subtree 00,
   subtree 11. Raw leaf: key `100` at depth 2, value `value 100`.

       00063441dd8fddabce96f7a9e0b737297a41b55924bb4e5e7c7e2f0772ef948a60debd4edb3501f7717e134d548ba9825c4e98a8fe174ec15b2d51667d3688dca3f7e9696969696969696969696969696969696969696969696969696969696969680976616c756520313030

   `00` | `0634` = 7 tags and 2 padding bits | subtree 00, subtree 11 | key `100` minus its first 2
   bits: 246 bits and 2 padding bits = 31 octets | `09` `value 100`

   Key `010` is absent because its path ends at the empty `K` at path `01`. Key `101` is absent
   because its path ends at the `R` at path `10`, which holds key `100`.

6. The empty state, any query. No entries. Tags `K`. No hashes. No raw leaves. The state root is
   the zero hash.

       0080

   `00` | `80` = 1 tag and 6 padding bits | none | none | none

7. The five-key state, a query with no listed keys and no ranges. No entries. Tags `H`. Hashes: the
   state root. No raw leaves.

       00409b9760b1a0bbb685177ad5ddd98b1ae446307255aca511c22e4c2e778cec425f

   `00` | `40` = 1 tag and 6 padding bits | the state root | none | none

8. A state holding only key `110`, listed key `110`. Entries: `110`. Tags `K`. No hashes. No raw
   leaves. The state root is the hash of leaf 110.

       0080

   `00` | `80` = 1 tag and 6 padding bits | none | none | none

   This proof is the same as in vector 6. The entries and the state root differ.
