# JIP-2: Node RPC

RPC specification for JAM nodes to ensure JAM tooling which relies on being an RPC client is implementation-agnostic.

## Notes

RPCs are evil and should generally not be used: they lead to chronic centralisation and trust-maximisation (just see Ethereum for a great example of this trap).

However, there are not at present any light clients for JAM and the resources needed to run a regular client make the inclusion of a full-client inside of tooling to be unrealistic. We must therefore reluctantly presume that the tool-user has access to a trustworthy full node and can use its RPCs.

As soon as a light-client implementation is viable, the use of RPCs should be phased out immediately in favour of embedded light-clients in the tooling.

## Protocol

JSON-RPC 2.0 is used, as defined by <https://www.jsonrpc.org/specification>. Except for
subscription Notifications, all method parameters are passed by-position, i.e. the `"params"`
member of Request objects should be an Array.

Generally JSON-RPC can be used over a variety of media and we don't make any assumptions of this,
but it is envisaged that Websockets will be the usual medium, on port 19800.

### Subscriptions

A subscription is created by calling a `subscribe` method, e.g. `subscribeFinalizedBlock`. On
success, the ID of the subscription is returned (a Number). A subscription can be stopped by
calling the corresponding `unsubscribe` method (e.g. `unsubscribeFinalizedBlock`), passing the
subscription ID as the sole parameter. For brevity these unsubscribe methods are not listed below.

Subscription updates are sent as Notifications, i.e. Requests without an `"id"` member. The method
name of such a Notification should match the name of the `subscribe` method originally used to
create the subscription, e.g. `subscribeFinalizedBlock`. The `"params"` member should be an Object
with a `"subscription"` member giving the subscription ID. This Object should also contain either a
subscription-specific `"result"` member, or an `"error"` member with a String containing a
human-readable error message.

## Common types

For convenience the following common types are defined:

- Blob: A String, containing padded Base64-encoded binary data, as per RFC 4648. The decoded data
  can have an arbitrary length.
- Hash: A String, containing padded Base64-encoded binary data, as per RFC 4648. The decoded data
  must be 32 bytes in length.
- Block Descriptor: An Object with the following members:
  - `"header_hash"`: Hash. Hash of the block's header.
  - `"slot"`: Number. The block's slot; this must match the slot field in the block's header.
- Chain Subscription Update: An Object with the following members:
  - `"header_hash"`: Hash. Header hash of the block that triggered this update.
  - `"slot"`: Number. Slot of the block that triggered this update.
  - `"value"`: Subscription-specific.
- State Key: A String, containing padded Base64-encoded binary data, as per RFC 4648. The decoded
  data must be exactly 31 bytes in length: a raw state key, as defined by the state Merklization
  appendix of the GP.
- State Proof: A Blob, containing a compact Merkle proof of part of the state of some block. The
  decoded data is as defined in [State proofs](#state-proofs).

## State proofs

A State Proof proves the values under some state keys, and the absence of other keys, in the state
of some block. It carries only the parts of the state trie needed for the keys it proves, and is
verified against a state root the client already trusts.

### Proof subtree

A proof subtree is built from a set of trie nodes, called expanded nodes, that contains the root
and, for every other node, its parent. It consists of those nodes and both children of each expanded
branch. Each node is represented as one of:

- `B`: an expanded branch, followed by its left child and then its right child.
- `L`: an expanded leaf.
- `E`: an empty subtree, expanded or not.
- `H`: a node that is neither expanded nor empty, given by its identity.

A proof subtree proves the key and entry, a value or a value hash, of every `L` it contains, and the
absence of every key whose path ends at an `E` or at an `L` holding a different key. It proves
nothing about a key whose path reaches an `H`. Two degenerate subtrees exist: a single `E` for an
empty state, and a single `H` carrying the state root when nothing is expanded.

Which nodes a server expands for a given request is defined under [Queries](#queries).

Throughout, the path to a node is the sequence of bits walked from the root to reach it, and the
node's depth is the length of that path.

### Encoding

The decoded data of a State Proof consists of a version octet followed by five sections, with no
lengths:

```
proof   = version tags kinds hashes keys values
version = 0x00
tags    = subtree, then zero bits up to the next octet boundary
subtree = B subtree subtree | H | E | L
kinds   = one kind octet per L, in tag order
hashes  = 32 octets per H and per hash-only leaf, in tag order
keys    = key suffix bits per full leaf, in tag order,
          then zero bits up to the next octet boundary
values  = value data per leaf, as per its kind octet, in tag order
```

The `version` octet identifies the encoding defined here, version 0; later revisions of this
document may define further versions.

The `tags` section lists the nodes of the proof subtree in pre-order: a `B` is followed by the tags
of its left subtree and then those of its right subtree. This order is called tag order, and the
other sections follow it.

Each tag is two bits: `B` is `00`, `H` is `01`, `E` is `10` and `L` is `11`. Tags are packed from
the most significant bits of each octet down, so the first tag occupies bits 7 and 6 of the first
octet of the `tags` section. The `tags` section ends when the subtree is complete: starting with
one subtree owed, each tag pays for one and each `B` owes two more, and the subtree is complete
when nothing is owed.

The number of `L` tags is the length of the `kinds` section. The `hashes` section is 32 octets for
each `H` tag and each kind octet of the hash-only form. The `keys` section is, for
each kind octet whose leaf ships its key, $248 - d$ bits, $d$ being the depth of the corresponding
`L`, rounded up to whole octets once at the end. The `values` section is the remainder.

The `kinds` section is a sequence of kind octets, one per `L` tag in tag order. A kind octet
describes one leaf: bit 7 is set for a fully elided leaf and bit 6 for a key-elided leaf, and bits 5
to 0 give the value form:

| Value form | Bits 5 to 0 | Data in the `values` section |
|---|---|---|
| embedded | 0 to 32, the value's length | The value |
| long | 33 | `len`, then the value; `len` is the value's length |
| hash-only | 34 | None; the value's hash is in the `hashes` section |
| invalid | 35 to 63 | |

| Bit 7 | Bit 6 | Kind | Key | Value |
|---|---|---|---|---|
| 0 | 0 | Full | Suffix in the `keys` section | As per the value form |
| 0 | 1 | Key-elided | Supplied by the client | As per the value form |
| 1 | 0 | Fully elided | Supplied by the client | Supplied by the client; embedded form, 0 |
| 1 | 1 | Invalid | | |

`len` is encoded as per the GP's variable-length serialization of natural numbers, and must be
greater than 32 and less than $2^{32}$. A hash-only leaf is encoded as per the GP as a leaf whose
value is longer than 32 octets, with the given hash in place of the value's hash.

The `hashes` section is a sequence of 32-octet entries, one per `H` tag and one per hash-only leaf,
in tag order. The entry for an `H` is the identity of the node it stands for. If that node is a left
child, its identity is given as its parent's encoding stores it: with the most significant bit (bit
7 of octet 0) cleared. The entry for a leaf of the hash-only form is the hash of its value, and
takes its place in the sequence at the position of the leaf's `L` tag.

The `keys` section holds, for each full leaf in tag order, the last $248 - d$ bits of its key, $d$
being the leaf's depth and the first $d$ bits being the path to it. These suffixes are concatenated,
most significant bit first, without padding between them; the section is padded with zero bits to an
octet boundary at its end only. The leaf's key is the path to it followed by its suffix.

The `values` section holds, for each leaf in tag order, the data its value form announces, and
nothing else.

### Canonical form

A verifier must reject a proof if any of the following holds:

1. The version octet is not 0.
2. The proof is empty, or the `tags` section ends before the subtree is complete.
3. A padding bit of the `tags` section is set.
4. A `B` is at depth 248 or deeper.
5. A `B` has two `H` children, two `E` children, or an `E` and an `L` child in either order. A `B`
   with an `H` and an `E` child is valid: it is how the path of an absent listed key ends at an
   empty child beside an unexpanded sibling.
6. An `H` carries the zero hash, or an `H` which is a left child has the most significant bit of
   its identity set.
7. A kind octet has an invalid value form (35 or more), has both bits 7 and 6 set, or has bit 7 set
   and a non-zero value form.
8. A long value's `len` is 32 or less, is $2^{32}$ or more, or is not the octets the GP's
   encoding gives for that number (e.g. `80 28` in place of `28` for 40).
9. A padding bit of the `keys` section is set.
10. For a key-elided or fully elided leaf, the client's known keys contain no key, or more than one
    key, starting with the path to the leaf.
11. A section is shorter than its contents require, or octets remain after the value data of the
    last leaf.
12. The identity of the root of the proof subtree differs from the trusted state root.

These rules give every proof subtree, with a given kind for each leaf, exactly one encoding. The
verifier accepts any canonically encoded proof subtree whose root identity is the state root. It
checks neither that the subtree is minimal for the query nor that the leaf kinds conform to the
query. A proof may therefore expand more of the trie than the query needs, which proves more keys,
never fewer; the subtree and kinds a server produces are defined under [Queries](#queries), and a
client may bound the size of the proofs it accepts.

### Verification

The verifier takes the proof, a trusted state root and, if any leaf is elided, the client's known
keys with their values. It reads the tags in order and keeps two things: the path to the node being
read, and a stack of the branches that are still open, i.e. whose right child has not been read yet.
When a branch's left child is complete, its identity and tag are stored in the branch's stack entry
and the right child is read next. When the right child is complete too, the branch's identity is
computed from the two children and its entry is popped; the branch is then itself a complete child
of the branch below it, so the same step repeats. The proof is valid when the last branch to close
is the root and its identity equals the state root:

    stack = empty, path = empty
    loop:
        tag = next tag
        match tag:
            B: push an empty entry; append 0 to path; continue
            H: id = next hash
            E: id = zero hash; path is covered
            L: (key, entry) = next leaf at path; id = identity of the leaf
               key is present with entry; path is covered
        loop:
            if stack is empty:
                require id = state root; done
            if top of stack is empty:
                top of stack = (tag, id); set the last bit of path to 1; break
            (left tag, left) = pop stack; remove the last bit of path
            check rule 5 on (left tag, tag)
            id = identity of the branch with children left and id; tag = B

The pseudo-code omits the other canonical-form rules, which are checked as each tag, leaf and branch
is read or completed. To read a leaf, the verifier takes the next kind octet. For a full leaf, the
key is the path followed by the next $248 - d$ bits of the `keys` section, $d$ being the length of
the path; otherwise it is the single known key starting with the path (rule 10). The entry is the
value read from the `values` section as per the value form, the next hash for a hash-only value, or,
for a fully elided leaf, the known value of the key; for a key-elided leaf, any known value is
ignored.

The result is the set of present keys with their entries, each either a value or a value hash,
and the set of covered paths. A key is then:

- Present, if it is a present key.
- Absent, if it is not present and a covered path is a prefix of it: its path in the proof
  subtree ends at an `E` or at an `L` holding a different key.
- Not covered, otherwise: its path leaves the proof subtree through an `H`.

A client must treat a key which is not covered as a failed proof, never as an absent key. For a
listed key this is the outcome of looking it up. For a range the client cannot look up keys it does
not know, so it must instead check that no `H` stands for a node whose path is a prefix of some key
within the range: such an `H` could hide keys of the range, which would then be neither present nor
absent in the result.

### Queries

A query consists of:

- Listed keys: State Keys, strictly ascending.
- Ranges: pairs `[start, end]` of prefix bounds, each 0 to 31 octets long. The range contains
  every key from `start` padded to 31 octets with `0x00` up to `end` padded to 31 octets with
  `0xFF`, both inclusive; `[p, p]` is thus every key starting with `p`, and `[empty, empty]` is
  the whole state. After padding, `start` must not exceed `end`, each range's `start` must exceed
  the previous range's `end`, and no listed key may lie within a range.
- Known mode: one of `none`, `keys` and `keys_and_values`. `keys` declares that the client
  already holds every listed key that is in the state and every key of the state within a range;
  `keys_and_values` declares that it holds these keys with their values. The server does not
  check the declaration.

The proof subtree for a query expands a node if a listed key starts with the path to it, or if a key
starting with the path to it lies within a listed range. It follows that:

- the root is expanded whenever the query has a listed key or a range;
- a listed key that is not in the state has a path ending at an `E`, or at an `L` holding a
  different key;
- for a state with a single key, any non-empty query gives a single `L`;
- an empty query gives a single `H` carrying the state root.

A leaf is eligible for elision if its key is a listed key or lies within a range, and no other
listed key starts with the path to it.  The second condition is there because the verifier
identifies an elided leaf by the one known key that starts with the path to it, as described under
[Verification](#verification). That identification fails in one situation: a listed key that is
absent from the state, whose walk ends at the leaf of another listed key. Both keys then start with
that leaf's path, so that leaf is not eligible and stays full. The listed keys meant here are those
of the request as sent; a key the cut drops is still among the client's known keys.

A leaf's kind follows from its eligibility and the `known` mode. An eligible leaf is full under
`none`, key-elided under `keys` and fully elided under `keys_and_values`. A leaf that is not
eligible is full whatever the mode.

A full leaf's value form is embedded if the value is at most 32 octets long, and otherwise long,
with one exception: a leaf whose key is neither a listed key nor within a range is in the proof only
because a listed key's path ends at it or because it borders a range, so its value was not asked
for, and it uses the hash-only form, shipping the value's hash instead.

The listed keys and the ranges form one ascending sequence of items, since the keys are sorted, the
ranges are sorted and disjoint, and no key lies within a range. The size limit applies to that
sequence as a whole: the server considers listed keys and range leaves together, in ascending key
order.

Each item is charged the size of the leaf the proof contains for it: a listed key is charged for the
leaf its path ends at, which holds either that key or, if the key is absent, another key, and
nothing if its path ends at an empty subtree; a range is charged for each leaf within it, and a
range containing no key of the state contributes no item. A leaf's charged size is:

- 1, for its kind octet;
- $\lceil (248 - d) / 8 \rceil$ if its key suffix is shipped;
- the length of its data in the `values` section, or 32 for a hash-only leaf.

The charge is computed per leaf and is deliberately conservative: key suffixes are packed without
per-leaf padding, so the charged total may exceed the octets the leaves actually add.

The server includes items in order while their charged total stays within the limit; the first item
is always included. If an item does not fit, the server stops there: `"complete"` is False and
`"proven_through"` is the key of the last included item, whether it is a listed key or a key within
a range. If every item fits, `"complete"` is True.

The query cut at a key $k$ is a shorter query derived from the request: it keeps the listed keys
that do not exceed $k$, removes every range whose padded `start` exceeds $k$, and ends every
remaining range whose padded `end` exceeds $k$ at $k$. A truncated reply is not a special form of
proof: it carries the proof subtree of the query cut at `"proven_through"`, with elision decided
against the request as sent, and the client verifies it as such.

A client receiving a truncated reply must verify it against the query cut at `"proven_through"`, and
may continue with the remaining listed keys and the ranges cut to start after it. The cut cannot be
inferred from the proof, and a truncated proof may still cover the whole query, as an absent listed
key can expand the region the cut removed; `"complete"` and `"proven_through"` are authoritative.

### Test vectors

These vectors use a state of five keys, named by their first three bits: `000`, `001`, `100`,
`110` and `111`. Octet 0 of each key is those three bits followed by `11010`, and octets 1 to 30
are `0x5A`; key `110` is thus `0xDA` followed by thirty `0x5A` octets. The value under each key is
the nine octets of the ASCII string `value ` followed by the key's three bits, e.g. `value 110`.
The state root is `9b9760b1a0bbb685177ad5ddd98b1ae446307255aca511c22e4c2e778cec425f`. The
identities used below, as they appear in the `hashes` section, are those of the leaves holding
keys `000`, `100`, `110` and `111`, and of the subtrees under the prefixes `0` (keys `000` and
`001`), `00` (the same two keys) and `11` (keys `110` and `111`):

    leaf 000     40f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea6
    leaf 100     6b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df
    leaf 110     2af306b851cf153d7c58cf43682fed3b75f2cdd7dfc9a547dcfe990ccaea77ab
    leaf 111     883a0c1f05cbac875dae2e53c5c15dfad106d7272bf3cb07015aae7e8656f7f9
    subtree 0    50918ec4ad4465ee3baa8a272129bd2534ad911f63d22abff4716baa78bc97c3
    subtree 00   41dd8fddabce96f7a9e0b737297a41b55924bb4e5e7c7e2f0772ef948a60debd
    subtree 11   4edb3501f7717e134d548ba9825c4e98a8fe174ec15b2d51667d3688dca3f7e9

Each proof below is given in hex, followed by its version octet, tags, kinds, hashes, keys and
values, separated by `|`.

- Key `110`, known mode `none`:

      0011340950918ec4ad4465ee3baa8a272129bd2534ad911f63d22abff4716baa78bc97c36b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df883a0c1f05cbac875dae2e53c5c15dfad106d7272bf3cb07015aae7e8656f7f9d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d076616c756520313130

  `00` | `1134` = `B H B H B L H` and 2 padding bits | `09` = full, embedded, 9 octets |
  subtree 0, leaf 100, leaf 111 | key `110` at depth 3: 245 bits and 3 padding bits = 31 octets
  `d2…d0` | `value 110`.

- Keys `001` and `111`, known mode `none`:

      0001e11c090940f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea66b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df2af306b851cf153d7c58cf43682fed3b75f2cdd7dfc9a547dcfe990ccaea77abd2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d69696969696969696969696969696969696969696969696969696969696968076616c75652030303176616c756520313131

  `00` | `01e11c` = `B B B H L E B H B H L` and 2 padding bits | `0909` = full, embedded, 9
  octets, twice | leaf 000, leaf 100, leaf 110 | keys `001` and `111` at depth 3: 245 and 245
  bits and 6 padding bits = 62 octets | `value 001`, `value 111`.

- Keys `001` and `111`, known mode `keys`, verified with known keys `001` and `111`:

      0001e11c494940f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea66b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df2af306b851cf153d7c58cf43682fed3b75f2cdd7dfc9a547dcfe990ccaea77ab76616c75652030303176616c756520313131

  `00` | `01e11c` as above | `4949` = key-elided, embedded, 9 octets, twice | leaf 000, leaf
  100, leaf 110 | empty | `value 001`, `value 111`.

- The range whose bounds are keys `001` and `110`, known mode `none`:

      0001e33409090940f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea6883a0c1f05cbac875dae2e53c5c15dfad106d7272bf3cb07015aae7e8656f7f9d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d2d34b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b4b5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a76616c75652030303176616c75652031303076616c756520313130

  `00` | `01e334` = `B B B H L E B L B L H` and 2 padding bits | `090909` = full, embedded, 9
  octets, three times | leaf 000, leaf 111 | keys `001` at depth 3, `100` at depth 2 and `110`
  at depth 3: 245, 246 and 245 bits, no padding = 92 octets | `value 001`, `value 100`,
  `value 110`.

- Keys `010` and `101`, neither in the state (octet 0 `0x5A` and `0xBA`, then thirty `0x5A`
  octets), known mode `none`:

      0006340941dd8fddabce96f7a9e0b737297a41b55924bb4e5e7c7e2f0772ef948a60debd4edb3501f7717e134d548ba9825c4e98a8fe174ec15b2d51667d3688dca3f7e96969696969696969696969696969696969696969696969696969696969696876616c756520313030

  `00` | `0634` = `B B H E B L H` and 2 padding bits | `09` = full, embedded, 9 octets |
  subtree 00, subtree 11 | key `100` at depth 2: 246 bits and 2 padding bits = 31 octets
  `69…68` | `value 100`. Key `010` ends at the `E`, and key `101` at the leaf holding key `100`.

- The empty state, any query: `0080` = `00` | `80` = `E` and 6 padding bits.

- The state holding only key `110` with the value `value 110`, whose root is the identity of
  leaf 110 above; key `110`, known mode `none`:

      00c009da5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a5a76616c756520313130

  `00` | `c0` = `L` and 6 padding bits | `09` = full, embedded, 9 octets | none | key `110` at
  depth 0: 248 bits = 31 octets | `value 110`.

## Error codes

The following error codes are defined:

- 1: Block unavailable. The `"data"` member of the error Object should be the Hash of the block's
  header.
- 2: Work-report unavailable. The `"data"` member of the error Object should be the Hash of the
  work-report.
- 3: DA segment unavailable.
- 0: Other error.

Later revisions of this specification may define further error codes, as such:

- RPC clients should not assume this list is exhaustive.
- RPC servers should only use error codes defined here or in the JSON-RPC specification; they
  should not invent additional codes.

## Chain subscriptions

The `subscribe` methods which create subscriptions tracking chain state all take a Boolean argument
indicating which chain to track: True meaning track the latest finalized block, False meaning track
the head of the "best" chain.

As the "best" chain may switch to a different fork at any time:

- Updates yielded by a subscription following the best chain are not guaranteed to ever be included
  in the finalized chain.
- Subscriptions following the best chain may yield "impossible" update sequences. For example, a
  subscription created with `subscribeWorkPackageStatus(..., false)` may yield a `"Reported"`
  status followed by a `"Reportable"` status, if the best chain switches from a fork where the
  package has been reported to a fork where it has not.

If these behaviours are unacceptable, use subscriptions tracking the latest finalized block
instead. Such subscriptions are well-behaved, but may be significantly delayed compared to
best-chain subscriptions.

## Methods

### `parameters()`
Returns the chain parameters.
#### Result
An Object describing the JAM chain parameterization, which may not be equivalent to the canonical
parameterization of the Gray Paper. The Object has a single `"V1"` member, which itself is an
Object with the following members, all Numbers:

- `"deposit_per_item"`: $\mathsf{B}_I$, the additional minimum balance required per item of
  elective service state.
- `"deposit_per_byte"`: $\mathsf{B}_L$, the additional minimum balance required per octet of
  elective service state
- `"deposit_per_account"`: $\mathsf{B}_S$, the basic minimum balance which all services require.
- `"core_count"`: $\mathsf{C}$, the total number of cores.
- `"min_turnaround_period"`: $\mathsf{D}$, the period in timeslots after which an unreferenced
  preimage may be expunged.
- `"epoch_period"`: $\mathsf{E}$, the length of an epoch in timeslots.
- `"max_accumulate_gas"`: $\mathsf{G}_A$, the gas allocated to invoke a work-report’s Accumulation
  logic.
- `"max_is_authorized_gas"`: $\mathsf{G}_I$, the gas allocated to invoke a work-package’s
  Is-Authorized logic.
- `"max_refine_gas"`: $\mathsf{G}_R$, the gas allocated to invoke a work-package’s Refine logic.
- `"block_gas_limit"`: $\mathsf{G}_T$, the total gas allocated for all Accumulation in a block.
- `"recent_block_count"`: $\mathsf{H}$, the size of recent history, in blocks.
- `"max_work_items"`: $\mathsf{I}$, the maximum amount of work items in a package.
- `"max_dependencies"`: $\mathsf{J}$, the maximum sum of dependency items in a work-report.
- `"max_tickets_per_block"`: $\mathsf{K}$, the maximum number of tickets which may be submitted in
  a single extrinsic.
- `"max_lookup_anchor_age"`: $\mathsf{L}$, the maximum age in timeslots of the lookup anchor.
- `"tickets_attempts_number"`: $\mathsf{N}$, the number of ticket entries per validator.
- `"auth_window"`: $\mathsf{O}$, the maximum number of items in the authorizations pool.
- `"slot_period_sec"`: $\mathsf{P}$, the slot period, in seconds.
- `"auth_queue_len"`: $\mathsf{Q}$, the number of items in the authorizations queue.
- `"rotation_period"`: $\mathsf{R}$, the rotation period of validator-core assignments, in
  timeslots.
- `"max_extrinsics"`: $\mathsf{T}$, the maximum number of extrinsics in a work-package.
- `"availability_timeout"`: $\mathsf{U}$, the period in timeslots after which reported but
  unavailable work may be replaced.
- `"val_count"`: $\mathsf{V}$, the total number of validators.
- `"max_authorizer_code_size"`: $\mathsf{W}_A$, the maximum size of is-authorized code in octets.
- `"max_input"`: $\mathsf{W}_B$, the maximum size of the concatenated variable-size blobs,
  extrinsics and imported segments of a work-package, in octets.
- `"max_service_code_size"`: $\mathsf{W}_C$, the maximum size of service code in octets.
- `"basic_piece_len"`: $\mathsf{W}_E$, the basic size of erasure-coded pieces in octets.
- `"max_imports"`: $\mathsf{W}_M$, the maximum number of imports in a work-package.
- `"segment_piece_count"`: $\mathsf{W}_P$, the number of erasure-coded pieces in a segment.
- `"max_report_elective_data"`: $\mathsf{W}_R$, the maximum total size of all unbounded blobs in a
  work-report, in octets.
- `"transfer_memo_size"`: $\mathsf{W}_T$, the size of a transfer memo in octets.
- `"max_exports"`: $\mathsf{W}_X$, the maximum number of exports in a work-package.
- `"epoch_tail_start"`: $\mathsf{Y}$, the number of slots into an epoch at which ticket-submission
  ends.

All parameters not described are assumed to be their canonical values. Some parameters are
dependent on other values:

- $\mathsf{W}_G = 4,104$: The size of a (reconstructed) segment is fixed.
- $\mathsf{W}_P = \frac{\mathsf{W}_G}{\mathsf{W}_E}$: The number of EC pieces in a segment.

### `bestBlock()`
Returns the header hash and slot of the head of the "best" chain.
#### Result
Block Descriptor.

### `subscribeBestBlock()`
Subscribe to updates of the head of the "best" chain, as returned by `bestBlock`.
#### Subscription update `"result"`
Block Descriptor.

### `finalizedBlock()`
Returns the header hash and slot of the latest finalized block.
#### Result
Block Descriptor.

### `subscribeFinalizedBlock()`
Subscribe to updates of the latest finalized block, as returned by `finalizedBlock`.
#### Subscription update `"result"`
Block Descriptor.

### `parent(header_hash)`
Returns the header hash and slot of the parent of the block with the given header hash.
#### Parameters
1. `header_hash`: Hash.
#### Result
Block Descriptor: The parent of the block with the given header hash.

### `stateRoot(header_hash)`
Returns the posterior state root of the block with the given header hash.
#### Parameters
1. `header_hash`: Hash.
#### Result
Hash: The state root.

### `stateValue(header_hash, key)`
Returns the value stored under the given raw state key in the posterior state of the block with
the given header hash. Unlike `serviceValue`, this method takes a full 31-byte state key, so it
can be used to read chain-level state components as well as any service state whose key the
client can compute.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
2. `key`: State Key.
#### Result
Null if there is no value under the given key, otherwise a Blob containing the value.

### `subscribeStateValue(key, finalized)`
Subscribe to updates of the value stored under the given raw state key. An update is sent only
when the value changes.
#### Parameters
1. `key`: State Key.
2. `finalized`: Boolean: True to track the latest finalized block, False to track the head of the
   "best" chain.
#### Subscription update `"result"`
Chain Subscription Update. The `"value"` member is Null when there is no value under the given
key, otherwise a Blob containing the value.

### `stateProof(header_hash, keys, ranges, known, size_limit)`
Returns a State Proof for the given query in the posterior state of the block with the given
header hash. The query is as defined in [Queries](#queries), and the proof as defined in
[State proofs](#state-proofs).

The server rejects the request with the JSON-RPC invalid params error if the listed keys are not
strictly ascending, a range bound is longer than 31 octets, a range's padded `start` exceeds its
padded `end`, a range's padded `start` does not exceed the previous range's padded `end`, a listed
key lies within a range, `known` is not one of the Strings below, or `size_limit` is not a
non-negative integer. Servers may lower `size_limit` to a cap of their choosing, and may cap the
number of listed keys plus ranges, rejecting a request over that cap with the same error. A server
may reject, with the same error, a request whose first item alone would make the reply exceed the
server's response size cap.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
2. `keys`: Array of State Keys: The listed keys, strictly ascending.
3. `ranges`: Array of `[start, end]` Arrays of Blobs: The ranges, ascending. Each bound must
   decode to between 0 and 31 octets; both bounds are inclusive.
4. `known`: String: The known mode, one of `"none"`, `"keys"` and `"keys_and_values"`.
5. `size_limit`: Number: A non-negative integer: soft limit on the total charged size of the leaves
   in the proof, in octets. The first item is included even if it alone exceeds the limit.
#### Result
An Object with the following members:
- `"proof"`: State Proof.
- `"complete"`: Boolean. False if the size limit cut the query short.
- `"proven_through"`: State Key. Present only if `"complete"` is False: the key of the last item
  included. The proof is the proof of the query cut at this key.

### `beefyRoot(header_hash)`
Returns the BEEFY root of the block with the given header hash.
#### Parameters
1. `header_hash`: Hash.
#### Result
Hash: The BEEFY root.

### `statistics(header_hash)`
Returns the activity statistics stored in the posterior state of the block with the given header
hash.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
#### Result
Blob: Activity statistics encoded as per the GP.

### `subscribeStatistics(finalized)`
Subscribe to updates of the activity statistics stored in chain state.
#### Parameters
1. `finalized`: Boolean: True to track the latest finalized block, False to track the head of the
   "best" chain.
#### Subscription update `"result"`
Chain Subscription Update. The `"value"` member is a Blob, containing activity statistics encoded
as per the GP.

### `serviceData(header_hash, id)`
Returns the storage data for the service with the given ID.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
2. `id`: Number: The ID of the service.
#### Result
Null if there is no service with the given ID, or Blob, containing the service data encoded as per
the GP.

### `subscribeServiceData(id, finalized)`
Subscribe to updates of the storage data for the service with the given ID.
#### Parameters
1. `id`: Number: The ID of the service.
2. `finalized`: Boolean: True to track the latest finalized block, False to track the head of the
   "best" chain.
#### Subscription update `"result"`
Chain Subscription Update. The `"value"` member is Null when there is no service with the given ID,
otherwise it is a Blob containing the service data encoded as per the GP.

### `serviceValue(header_hash, id, key)`
Returns the value associated with the given service ID and key in the posterior state of the block
with the given header hash. This method can be used to query arbitrary key-value pairs set by
service accumulation logic.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
2. `id`: Number: The ID of the service.
3. `key`: Blob: The key.
#### Result
Null if there is no value associated with the given service ID and key, otherwise a Blob containing
the value.

### `subscribeServiceValue(id, key, finalized)`
Subscribe to updates of the value associated with the given service ID and key.
#### Parameters
1. `id`: Number: The ID of the service.
2. `key`: Blob: The key.
3. `finalized`: Boolean: True to track the latest finalized block, False to track the head of the
   "best" chain.
#### Subscription update `"result"`
Chain Subscription Update. The `"value"` member is Null when there is no value associated with the
given service ID and key. Otherwise, it is a Blob containing the value.

### `servicePreimage(header_hash, id, hash)`
Returns the preimage of the given hash, if it has been provided to the given service in the
posterior state of the block with the given header hash.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
2. `id`: Number: The ID of the service.
3. `hash`: Hash: The hash whose preimage is being requested.
#### Result
Null if the preimage has not been provided to the given service, otherwise a Blob containing the
preimage.

### `subscribeServicePreimage(id, hash, finalized)`
Subscribe to updates of the preimage associated with the given service ID and hash.
#### Parameters
1. `id`: Number: The ID of the service.
2. `hash`: Hash. The hash whose preimage is of interest.
3. `finalized`: Boolean: True to track the latest finalized block, False to track the head of the
   "best" chain.
#### Subscription update `"result"`
Chain Subscription Update. The `"value"` member is Null if the preimage has not been provided to
the service, otherwise it is a Blob containing the preimage.

### `serviceRequest(header_hash, id, hash, len)`
Returns the preimage request associated with the given service ID and hash/length in the posterior
state of the block with the given header hash.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
2. `id`: Number: The ID of the service.
3. `hash`: Hash: The hash of the preimage.
4. `len`: Number: The preimage length.
#### Result
Null if the preimage with the given hash/length has neither been requested by nor provided to the
given service. An empty Array if the preimage has been requested, but not yet provided. Otherwise,
i.e. if the preimage has been provided, an Array of between 1 and 3 Numbers. The meaning of the
Numbers is as follows:
- The first Number is the slot in which the preimage was provided.
- The second Number, if present, is the slot in which the preimage was "forgotten".
- The third Number, if present, is the slot in which the preimage was requested again.

### `subscribeServiceRequest(id, hash, len, finalized)`
Subscribe to updates of the preimage request associated with the given service ID and hash/length.
#### Parameters
1. `id`: Number: The ID of the service.
2. `hash`: Hash: The hash of the preimage.
3. `len`: Number: The preimage length.
4. `finalized`: Boolean: True to track the latest finalized block, False to track the head of the
   "best" chain.
#### Subscription update `"result"`
Chain Subscription Update. The `"value"` member is either Null or an Array of Numbers, with the
same semantics as the result of the `serviceRequest` method.

### `workReport(hash)`
Returns the work-report with the given hash.
#### Parameters
1. `hash`: Hash: Hash of the work-report.
#### Result
Blob: The work-report with the given hash, encoded as per the GP.

### `submitWorkPackage(core, package, extrinsics)`
Submit a work-package to the guarantors currently assigned to the given core. This method will
return successfully if the work-package is submitted to at least one guarantor. It will not wait
for the package to be refined, reported, or accumulated. You should use e.g.
`subscribeWorkPackageStatus` to monitor the status of submitted work-packages.
#### Parameters
1. `core`: Number: The index of the core.
2. `package`: Blob: The work-package, encoded as per the GP.
3. `extrinsics`: Array of Blobs: The extrinsics.
#### Result
Null.

### `submitWorkPackageBundle(core, bundle)`
Submit a work-bundle to the guarantors currently assigned to the given core. This method will
return successfully if the bundle is submitted to at least one guarantor. It will not wait for the
package to be refined, reported, or accumulated. You should use e.g. `subscribeWorkPackageStatus`
to monitor the status of submitted work-packages.
#### Parameters
1. `core`: Number: The index of the core.
2. `bundle`: Blob: The work-bundle, encoded as per the GP.
#### Result
Null.

### `workPackageStatus(header_hash, hash, anchor)`
Returns the status of the given work-package following execution of the block with the given header
hash.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
2. `hash`: Hash: The hash of the work-package.
3. `anchor`: Hash: The hash of the work-package's anchor block's header. If this does not match the
   anchor specified in the work-package then an error or an incorrect status may be returned. An
   error may also be returned if this anchor block is too old.
#### Result
An Object with one of the following structures:

-     {"Reportable": {
          "remaining_blocks": Number
      }}

  This means the work-package has not yet been reported, but could be reported in a descendant block.

  `"remaining_blocks"` is the number of blocks remaining until the work-package can no longer be
  reported. 1 for example means that the next block is the last block in which the work-package can
  be reported.

-     {"Reported": {
          "reported_in": Block Descriptor,
          "core": Number,
          "report_hash": Hash
      }}

  This means the work-package has been reported but is not yet available.

  `"reported_in"` identifies the block in which the work-package was reported. `"core"` is the core
  on which the work-package was reported. `"report_hash"` is the hash of the work-report that was
  included on-chain.

-     {"Ready": {
          "reported_in": Block Descriptor,
          "core": Number,
          "report_hash": Hash,
          "ready_in": Block Descriptor
      }}

  This means the work-package is ready, i.e. it is either available or has been audited. A ready
  work-package is queued for accumulation once its prerequisites are met. Accumulation of a ready
  work-package is not guaranteed, in particular its prerequisites may never be met. Note that there
  is no `"Accumulated"` status to indicate when accumulation has happened. To determine if/when a
  work-package is accumulated, you should monitor service state for the expected changes using e.g.
  `subscribeServiceValue`.

  `"reported_in"`, `"core"`, and `"report_hash"` have the same meaning as for the `"Reported"`
  status. `"ready_in"` identifies the block in which the work-package became ready.

-     {"Failed": String}

  This means the work-package cannot become ready _on this fork_. This could be because:

  - Its anchor is on a different fork.
  - It was not reported in time.
  - It did not become available in time.

  The String is a freeform message giving details.

### `subscribeWorkPackageStatus(hash, anchor, finalized)`
Subscribe to status updates for the given work-package.
#### Parameters
1. `hash`: Hash: The hash of the work-package.
2. `anchor`: Hash: The hash of the work-package's anchor block's header. If this does not match the
   anchor specified in the work-package then the subscription may fail or yield incorrect statuses.
   The subscription may also fail if this anchor block is too old.
4. `finalized`: Boolean: True to track the latest finalized block, False to track the head of the
   "best" chain.
#### Subscription update `"result"`
Chain Subscription Update. The `"value"` member has the same structure and semantics as the result
of the `workPackageStatus` method.

### `submitPreimage(requester, preimage)`
Submit a preimage which is being requested by the given service. Note that this method does not
wait for the preimage to be distributed or integrated on-chain; it returns immediately.
#### Parameters
1. `requester`: Number: The ID of the service which has an outstanding request.
2. `preimage`: Blob: The preimage requested.
#### Result
Null.

### `listServices(header_hash)`
Returns a list of all services currently known to be on JAM. This is a best-effort list and may not
reflect the true state. Nodes could e.g. reasonably hide services which are not recently active
from this list.
#### Parameters
1. `header_hash`: Hash: The header hash indicating the block whose posterior state should be used
   for the query.
#### Result
Array of Numbers: The IDs of the services currently known to be on JAM.

### `fetchWorkPackageSegments(wp_hash, indices)`
Fetches a list of segments from the DA layer, exported by the work-package with the given hash.
#### Parameters
1. `wp_hash`: Hash: Hash of the exporting work-package.
2. `indices`: Array of Numbers: Indices into the list of segments exported by the work-package.
#### Result
Array of Blobs: The requested segments. Each Blob should be 4104 bytes long and the length of the
Array should match the length of the `indices` Array passed in to the method.

### `fetchSegments(segment_root, indices)`
Fetches a list of segments from the DA layer, exported by a work-package with the given segment
root hash.
#### Parameters
1. `segment_root`: Hash: Segment tree root hash of a work-package.
2. `indices`: Array of Numbers: Indices into the list of segments exported by the work-package.
#### Result
Array of Blobs: The requested segments. Each Blob should be 4104 bytes long and the length of the
Array should match the length of the `indices` Array passed in to the method.

### `syncState()`
Returns the sync state of the node.
#### Result
An Object with the following members:
- `"num_peers"`: Number of peers with an active UP 0 (block announcement) stream.
- `"status"`: A String that is either `"InProgress"` or `"Completed"`.

### `subscribeSyncStatus()`
Subscribe to changes in sync status.
#### Subscription update `"result"`
String: Either `"InProgress"` or `"Completed"`.
