# JIP-6: Compact state proofs

A compact Merkle proof for the JAM state trie, proving the values of some state keys and the absence
of others against a trusted state root.

## Motivation

A client that trusts a block's state root, for example from the block header, often needs a few
values from that block's state without trusting the node that serves them. A state proof lets the
node hand over the values together with enough of the state trie for the client to recompute the
state root and compare. Any tampering with a value, or any omission of a key the client asked about,
changes the recomputed root.

The first JIP-2 draft reused the range proof of the CE 129 state-request protocol for this. That
proof was designed for state synchronisation, where a reply carries thousands of consecutive
key-value pairs and the trie nodes are a small appendix. For a client asking for a handful of
unrelated keys it fits badly. It ships every branch node on the path as a whole 64-octet node,
although the verifier recomputes one of the two child identities in each node anyway, so about half
of the proof is redundant. It answers one contiguous key range per request, so unrelated keys need
one request each, each repeating the top of the trie. And it is verified by rebuilding a partial
trie, which is more machinery than a verifier running inside a PolkaVM service should need.

The compact proof defined here ships, for every branch on a path, only the identity of the child the
path does not enter, and describes the shape of the proof with two bits per node. It answers a set
of keys and a set of key ranges in one proof, proves absence as readily as presence, lets a client
that already holds keys or values leave them out, and is verified in a single pass with a stack. For
a single key at realistic state sizes it is about half the size of the range proof; for a range
whose contents the client already holds it is a small fraction. The encoding is canonical, so one
proof has one byte sequence, and a version octet leaves room for later revisions.

JIP-2 uses this proof in its `stateProof` method. The format does not depend on how it is requested
or transported, so other protocols may carry it as well.

## Proof subtree

A proof subtree is built from a set of trie nodes, called expanded nodes, that contains the root
and, for every other node, its parent. It consists of those nodes and both children of each expanded
branch. Each node is represented as one of:

- `B`: an expanded branch, followed by its left child and then its right child.
- `L`: an expanded leaf.
- `E`: an empty subtree, expanded or not.
- `H`: a node that is neither expanded nor empty, given by its identity.

A proof subtree proves:

- for every `L` it contains, the leaf's key and entry, where the entry is either a value or a value
  hash;
- the absence of every key whose path ends at an `E` or at an `L` holding a different key.

It proves nothing about a key whose path reaches an `H`. Two degenerate subtrees exist: a single `E`
for an empty state, and a single `H` carrying the state root when nothing is expanded.

Which nodes a server expands for a given request is defined under [Queries](#queries).

Throughout, the path to a node is the sequence of bits walked from the root to reach it, and the
node's depth is the length of that path.

## Encoding

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
octet of the `tags` section. The `tags` section ends when the subtree is complete: start with one
subtree open; every tag closes one, and every `B` opens two more; the section ends when none is
open.

The `hashes` section contains 32 octets for each `H` tag and for each hash-only leaf. The `keys`
section contains, for each kind octet whose leaf ships its key, $248 - d$ bits, $d$ being the depth
of the corresponding `L`, rounded up to whole octets once at the end. The `values` section consumes
all remaining octets.

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
value is longer than 32 octets, with the given hash in place of the value's hash; when a server uses
this form is defined under [Queries](#queries).

The `hashes` section is a sequence of 32-octet entries, one per `H` tag and one per hash-only leaf,
in tag order. The entry for an `H` is the identity of the node it stands for. If that node is a left
child, its identity is encoded as in its parent: with the most significant bit (bit 7 of octet 0)
cleared. The entry for a leaf of the hash-only form is the hash of its value, and takes its place in
the sequence at the position of the leaf's `L` tag.

The `keys` section holds, for each full leaf in tag order, the last $248 - d$ bits of its key, $d$
being the leaf's depth and the first $d$ bits being the path to it. These suffixes are concatenated,
most significant bit first, without padding between them; the section is padded with zero bits to an
octet boundary at its end only. The leaf's key is the path to it followed by its suffix.

The `values` section holds, for each leaf in tag order, the data its value form announces, and
nothing else.

## Canonical form

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
verifier accepts any canonically encoded proof subtree whose root identity is the state root.

It checks neither that the subtree is minimal for the query nor that the leaf kinds conform to the
query. A proof may therefore expand more of the trie than the query needs, which proves more keys,
never fewer. The subtree and kinds a server produces are defined under [Queries](#queries).

A client may bound the size of the proofs it accepts.

## Verification

The verifier takes the proof and a trusted state root. If any leaf is elided, it also takes the
client's known keys with their values. While reading the tags in order it keeps the path to the
current node and a stack of open branches, those whose right child is still to come.

Reading a `B` opens a branch: an entry is pushed and the left child is read next. When a node
completes and the top entry is still empty, the node is the left child: its identity and tag are
stored in the entry, and the right child is read next. When a node completes and the top entry
already holds a left child, the node is the right child: the branch's identity is computed from the
two, the entry is popped, and the branch is itself a completed node, to be handled the same way by
the entry below. The proof is valid when the final completed node is the root of the proof subtree
and its identity equals the state root:

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
is read or completed. To read a leaf, the verifier takes the next kind octet and then, by kind:

- full: the key is the path followed by the next $248 - d$ bits of the `keys` section, $d$ being the
  length of the path; the entry is read from the `values` section as the value form says, or is
  the next hash for the hash-only form;
- key-elided: the key is the single known key starting with the path; the entry is read as for a
  full leaf, and any known value is ignored;
- fully elided: the key is the single known key starting with the path, and the entry is its
  known value.

The result is the set of present keys with their entries, each either a value or a value hash,
and the set of covered paths. A key is then:

- Present, if it is a present key.
- Absent, if it is not present and a covered path is a prefix of it: its path in the proof
  subtree ends at an `E` or at an `L` holding a different key.
- Not covered, otherwise: its path leaves the proof subtree through an `H`.

A client must treat a key that is not covered as a failed proof, never as an absent key. For a
listed key this is the outcome of looking it up. For a range, the client cannot individually look up
keys it does not know. It must therefore check that no `H` stands for a node whose path is a prefix
of a key within the range. Such an `H` could hide keys of the range, which would then be neither
present nor absent in the result.

## Queries

A query consists of:

- Listed keys: a strictly ascending sequence of State Keys.
- Ranges: pairs `[start, end]` of prefix bounds, each 0 to 31 octets long. The range contains
  every key from `start` padded to 31 octets with `0x00` up to `end` padded to 31 octets with
  `0xFF`, both inclusive; `[p, p]` is thus every key starting with `p`, and `[empty, empty]` is
  the whole state.
- Known mode: one of `none`, `keys` and `keys_and_values`. `keys` declares that the client
  already holds every listed key that is in the state and every key of the state within a range;
  `keys_and_values` declares that it holds these keys with their values. The server does not
  check the declaration.

After padding, each range's `start` must not exceed its `end`, each range's `start` must exceed the
previous range's `end`, and no listed key may lie within a range.

The proof subtree for a query expands a node if a listed key starts with the path to it, or if a key
starting with the path to it lies within a listed range. It follows that:

- the root is expanded whenever the query has a listed key or a range;
- a listed key that is not in the state has a path ending at an `E`, or at an `L` holding a
  different key;
- for a state with a single key, any non-empty query gives a single `L`;
- an empty query gives a single `H` carrying the state root.

A leaf is eligible for elision if its key is a listed key or lies within a range, and no other
listed key starts with the path to it. The second condition is there because the verifier identifies
an elided leaf by the one known key that starts with the path to it, as described under
[Verification](#verification). That identification is ambiguous in one situation: a listed key that
is absent from the state, whose walk ends at the leaf of another listed key. Both keys then start
with that leaf's path, so that leaf is not eligible and stays full. The listed keys meant here are
those of the request as sent; a key the cut drops is still among the client's known keys.

A leaf's kind follows from its eligibility and the `known` mode. An eligible leaf is full under
`none`, key-elided under `keys` and fully elided under `keys_and_values`. A leaf that is not
eligible is full whatever the mode.

A full leaf's value form is embedded if the value is at most 32 octets long, and otherwise long.

There is one exception. A leaf whose key is neither a listed key nor within a range is in the proof
only because a listed key's path ends at it, or because a key within a range starts with the path to
it while its own key lies outside every range; its value was not asked for, so such a leaf uses the
hash-only form and ships the value's hash instead.

The charged keys of a query are its listed keys and the keys of the state that lie within its
ranges. Since the listed keys are sorted, the ranges are sorted and disjoint, and no listed key lies
within a range, the charged keys form one ascending sequence. A range containing no key of the state
adds no charged keys. The size limit applies to this sequence.

The charge for a charged key is the size of the leaf at which its lookup ends. For an absent listed
key, that leaf holds a different key. A listed key whose lookup ends at an empty subtree is charged
nothing. A leaf's charged size is:

- 1, for its kind octet;
- $\lceil (248 - d) / 8 \rceil$ if its key suffix is shipped;
- the length of its data in the `values` section, or 32 for a hash-only leaf.

The charge is computed per leaf and is deliberately conservative: key suffixes are packed without
per-leaf padding, so the charged total may exceed the octets the leaves actually add.

The server includes charged keys in order while their total charge stays within the limit; the first
charged key is always included. If a charged key does not fit, the server stops there: `"complete"`
is False and `"proven_through"` is the last charged key included. If every charged key fits,
`"complete"` is True.

The query cut at a key $k$ is a shorter query derived from the request: it keeps the listed keys
that do not exceed $k$, removes every range whose padded `start` exceeds $k$, and ends every
remaining range whose padded `end` exceeds $k$ at $k$. A truncated reply is not a special form of
proof: it carries the proof subtree of the query cut at `"proven_through"`, and the client verifies
it as such. Elision in that proof is decided against the request as sent, not the cut query.

A client receiving a truncated reply must verify it against the query cut at `"proven_through"`, and
may continue with the remaining listed keys and the ranges cut to start after it.

The cut cannot be inferred from the proof: a truncated proof may still cover the whole query, as an
absent listed key can expand the region the cut removed. `"complete"` and `"proven_through"` are
authoritative.

## Test vectors

These vectors use a state of five keys, named by their first three bits: `000`, `001`, `100`, `110`
and `111`. Octet 0 of each key is those three bits followed by `11010`, and octets 1 to 30 are
`0x5A`; key `110` is thus `0xDA` followed by thirty `0x5A` octets. The value under each key is the
nine octets of the ASCII string `value ` followed by the key's three bits, e.g. `value 110`. The
state root is `9b9760b1a0bbb685177ad5ddd98b1ae446307255aca511c22e4c2e778cec425f`.

The identities used below, as they appear in the `hashes` section, are those of the leaves holding
keys `000`, `100`, `110` and `111`, and of the subtrees under the prefixes `0` (keys `000` and
`001`), `00` (the same two keys) and `11` (keys `110` and `111`):

    leaf 000     40f3854ff47a42a159d21e8275e15df4937d19063426e38cfdc30878dbc22ea6
    leaf 100     6b177808b75d08e8e68fc330d42ebdda1eadbd804db18d1e4679cbcb349c52df
    leaf 110     2af306b851cf153d7c58cf43682fed3b75f2cdd7dfc9a547dcfe990ccaea77ab
    leaf 111     883a0c1f05cbac875dae2e53c5c15dfad106d7272bf3cb07015aae7e8656f7f9
    subtree 0    50918ec4ad4465ee3baa8a272129bd2534ad911f63d22abff4716baa78bc97c3
    subtree 00   41dd8fddabce96f7a9e0b737297a41b55924bb4e5e7c7e2f0772ef948a60debd
    subtree 11   4edb3501f7717e134d548ba9825c4e98a8fe174ec15b2d51667d3688dca3f7e9

Each proof is shown in hex, with its version octet, tags, kinds, hashes, keys and values separated
by `|`.

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
