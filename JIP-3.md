# JIP-3: Structured logging

Specification for structured logging of JAM nodes allowing integration into JAM Tart (Testing,
Analytics and Research Telemetry).

## Connection to telemetry server

Nodes shall provide a CLI option `--telemetry HOST:PORT`. Nodes, with said CLI option given, shall
make a TCP/IP connection to said endpoint.

## Message encoding

Data sent over the connection to the telemetry server shall consist of variable-length messages.
Each message is sent in two parts. First, the size (in bytes) of the message content is sent,
encoded as a little-endian 32-bit unsigned integer. Second, the message content itself is sent.
Note that this message encoding matches JAMNP-S.

Message content shall be encoded as per regular JAM serialization. That is, a message with fields
$a$, $b$, $c$, etc shall be encoded as $\mathcal{E}(a, b, c, ...)$. Fields of type `u8`, `u16`,
`u32`, and `u64` shall be encoded using $\mathcal{E}_1$, $\mathcal{E}_2$, $\mathcal{E}_4$, and
$\mathcal{E}_8$ respectively (ie they should use fixed-length encoding).

## Notation

In the type/message definitions below:

- Fields are identified by their type, eg `u32`, with their meaning given in parentheses when not
  self-evident.
- `[T; N]` means a fixed-length sequence of $N$ elements each of type $T$.
- `A ++ B ++ ...` means a tuple $(A, B, ...)$.
- `len++` preceding a sequence type indicates that the sequence should be explicitly prefixed by
  its length when encoding. The length should be encoded using variable-length general natural
  number serialization.

## Common types

The following "common" types are defined:

    bool = 0 (False) OR 1 (True)
    Option<T> = 0 OR (1 ++ T)
    String<N> = len++[u8] (len <= N, byte sequence must be valid UTF-8)
    Reason = String<128> (Freeform reason for eg failure of an operation, may be empty if unknown)

    Timestamp = u64 (Microseconds since the beginning of the Jam "Common Era")
    Event ID = u64

    Peer ID = [u8; 32] (Ed25519 public key)
    Peer Address = [u8; 16] ++ u16 (IPv6 address plus port)
    Connection Side = 0 (Local) OR 1 (Remote)

    Slot = u32
    Epoch Index = u32 (Slot / E)
    Validator Index = u16
    Core Index = u16
    Service ID = u32
    Shard Index = u16

    Hash = [u8; 32]
    Header Hash = Hash
    Work-Package Hash = Hash
    Work-Report Hash = Hash
    Erasure-Root = Hash
    Segments-Root = Hash
    Ed25519 Signature = [u8; 64]

    Block Summary =
        u32 (Size in bytes) ++
        u32 (Number of tickets) ++
        u32 (Number of preimages) ++
        u32 (Total size of preimages in bytes) ++
        u32 (Number of guarantees) ++
        u32 (Number of assurances) ++
        u32 (Number of dispute verdicts)

    Gas = u64
    Exec Cost =
        Gas (Gas used) ++
        u64 (Elapsed wall-clock time in nanoseconds)
    Is-Authorized Cost =
        Exec Cost (Total) ++
        Exec Cost (Host calls)
    Refine Cost =
        Exec Cost (Total) ++
        Exec Cost (historical_lookup calls) ++
        Exec Cost (machine/expunge calls) ++
        Exec Cost (peek/poke/pages calls) ++
        Exec Cost (invoke calls) ++
        Exec Cost (Other host calls)
    Accumulate Cost =
        u32 (Number of accumulate calls) ++
        u32 (Number of items accumulated) ++
        Exec Cost (Total) ++
        Exec Cost (read/write calls) ++
        Exec Cost (lookup/query/solicit/forget/provide calls) ++
        Exec Cost (info/new/upgrade/transfer/eject calls) ++
        Gas (Total gas charged for on-transfer execution) ++
        Exec Cost (Other host calls)
    On-Transfer Cost =
        u32 (Number of on-transfer calls) ++
        Exec Cost (Total) ++
        Exec Cost (read/write calls) ++
        Exec Cost (lookup calls) ++
        Exec Cost (Other host calls)

    Work-Item Summary =
        Service ID ++
        u32 (Payload size) ++
        Gas (Refine gas limit) ++
        Gas (Accumulate gas limit) ++
        u32 (Sum of extrinsic lengths) ++
        u16 (Number of imported segments) ++
        u16 (Number of exported segments)
    Work-Package Summary =
        u32 (Work-package size in bytes) ++
        Header Hash (Anchor) ++
        Slot (Lookup anchor slot) ++
        len++[Work-Package Hash] (Prerequisites) ++
        len++[Work-Item Summary]

    Work-Report Summary =
        Core Index ++
        Work-Package Hash ++
        u32 (Bundle size in bytes) ++
        Erasure-Root ++
        Segments-Root

    Guarantee Summary =
        Work-Report Hash ++
        Slot ++
        len++[Validator Index] (Guarantors)
    Guarantee Discard Reason =
        0 (Work-report included in block) OR
        1 (Replaced by better guarantee) OR
        2 (Too old) OR
        3 (Too many guarantees) OR
        4 (Other)
        (Single byte)

    Announced Preimage Forget Reason =
        0 (Preimage not requested on-chain) OR
        1 (Failed to acquire preimage) OR
        2 (Too many announced preimages) OR
        3 (Other)
        (Single byte)
    Preimage Discard Reason =
        0 (No longer requested on-chain) OR
        1 (Too many preimages) OR
        2 (Other)
        (Single byte)

## Node information message

The first message sent on each connection to the telemetry server should contain information about
the connecting node:

    Peer ID
    Peer Address
    u32 (Node flags)
    String<32> (Name of node implementation, eg "PolkaJam")
    String<32> (Version of node implementation, eg "1.0")
    String<512> (Freeform note with additional information about the node)

The node flags field should be treated as a bitmask. The following bits are defined:

- Bit 0 (LSB): 1 means the node uses a PVM recompiler, 0 means the node uses a PVM interpreter.

All other bits should be set to 0.

## Event messages

Following the initial node information message, a message should be sent every time one of the
events defined below occurs.

For ease of implementation, "sent" events (such as "bundle sent") may be emitted once all of the
data has been queued at the QUIC level; it is not necessary to wait for the data to actually be
sent over the network or acknowledged by the peer.

### Universal fields

All event messages begin with a `Timestamp` field indicating the time of the event, followed by a
single-byte discriminator identifying the event type. For brevity, the timestamp and discriminator
are omitted in the event definitions below.

The discriminator value for each event type is given in the relevant section heading. For example,
the "connection refused" event has discriminator 20.

### Event IDs

Each event sent over a connection is implicitly given an ID:

- The first event is given ID 0.
- The event immediately following an event E is given the ID of event E plus N, where N is the
  number of dropped events if E is a "dropped" event, or 1 otherwise.

Event IDs are used to link related events. For example, the "connected out" event contains the ID
of the corresponding "connecting out" event.

## Meta events

### 0: Dropped

If, due to for example limited buffer space, a node needs to drop a contiguous series of events
before they can be transmitted over the connection to the telemetry server, the dropped events
should be replaced with a single "dropped" event:

    Timestamp (Timestamp of the last dropped event)
    u32 (Number of dropped events)

Dropped events should not be common and are expected to only be produced if the node or network
become overloaded.

Note that each dropped event message contains _two_ `Timestamp` fields: the universal `Timestamp`
field included in all event messages, which should be taken from the _first_ dropped event, and the
`Timestamp` field defined above, which should be taken from the _last_ dropped event.

## Status events

### 10: Status

Emitted periodically (approximately every 2 seconds), to provide a summary of the node's current
state. Note that most of this information can be derived from other events.

    u32 (Number of validator peers)
    u32 (Number of non-validator peers)
    u32 (Number of peers with a block announcement stream open)
    [u8; C] (Number of guarantees in pool, by core; C is the total number of cores)
    u32 (Number of shards in availability store)
    u64 (Total size of shards in availability store, in bytes)
    u32 (Number of announced preimages pending acquisition from peers)
    u32 (Number of preimages in pool, ready to be included in a block)

### 11: Best block changed

Emitted when the node's best block changes.

    Slot (New best slot)
    Header Hash (New best header hash)

### 12: Finalized block changed

Emitted when the latest finalized block (from the node's perspective) changes.

    Slot (New finalized slot)
    Header Hash (New finalized header hash)

### 13: Sync status changed

Emitted when the node's sync status changes. This status is subjective, indicating whether or not
the node believes it is sufficiently synced with the network to be able to perform all of the
duties of a validator node (authoring, guaranteeing, assuring, auditing, and so on).

    bool (Does the node believe it is sufficiently in sync with the network?)

## Networking events

These events concern JAMNP-S connections.

### 20: Connection refused

Emitted when a connection attempt from a peer is refused.

    Peer Address

### 21: Connecting in

Emitted when a connection attempt from a peer is accepted. This event should be emitted as soon as
possible. In particular it should be emitted _before_ the connection handshake completes.

    Peer Address

### 22: Connect in failed

Emitted when an incoming connection attempt fails.

    Event ID (ID of the corresponding "connecting in" event)
    Reason

### 23: Connected in

Emitted when an incoming connection attempt succeeds.

    Event ID (ID of the corresponding "connecting in" event)
    Peer ID

### 24: Connecting out

Emitted when an outgoing connection attempt is initiated.

    Peer ID
    Peer Address

### 25: Connect out failed

Emitted when an outgoing connection attempt fails.

    Event ID (ID of the corresponding "connecting out" event)
    Reason

### 26: Connected out

Emitted when an outgoing connection attempt succeeds.

    Event ID (ID of the corresponding "connecting out" event)

### 27: Disconnected

Emitted when a connection to a peer is broken.

    Peer ID
    Peer Address
    Option<Connection Side> (Terminator of the connection, may be omitted in case of eg a timeout)
    Reason

## Block authoring/importing events

These events concern the block authoring and importing pipelines. Note that some events are common
to both authoring and importing, eg "block executed".

### 40: Authoring

Emitted when authoring of a new block begins.

    Slot
    Header Hash (Of the parent block)

### 41: Author failed

Emitted if block authoring fails for some reason.

    Event ID (ID of the corresponding "authoring" event)
    Reason

### 42: Authored

Emitted when a new block has been authored. This should be emitted as soon the contents of the
block have been determined, ideally before accumulation is performed and the new state root is
computed (which is included only in the following block).

    Event ID (ID of the corresponding "authoring" event)
    Header Hash (Of the new block)
    Block Summary

### 43: Importing

Emitted when importing of a block begins. This should not be emitted by the block author; the
author should emit the "authoring" event instead.

    Slot
    Header Hash
    Block Summary

### 44: Block verify failed

Emitted if verification of a block being imported fails for some reason. This includes if the block
is determined to be invalid, ie it does not satisfy all the validity conditions listed in the GP.
This should not be emitted by the block author (the author should emit the "author failed" event
instead).

    Event ID (ID of the corresponding "importing" event)
    Reason

### 45: Block verified

Emitted once a block being imported has been verified. That is, the block satisfies all the
validity conditions listed in the GP. This should be emitted as soon as this has been determined,
ideally before accumulation is performed and the new state root is computed. This should not be
emitted by the block author (the author should emit the "authored" event instead).

    Event ID (ID of the corresponding "importing" event)

### 46: Block execute failed

Emitted if execution of a block fails after authoring/verification. This can happen if, for
example, there is a collision amongst created service IDs during accumulation.

    Event ID (ID of the corresponding "authoring" or "importing" event)
    Reason

### 47: Block executed

Emitted following successful execution of a block. This should be emitted by both the block author
and importers.

    Event ID (ID of the corresponding "authoring" or "importing" event)
    len++[Service ID ++ Accumulate Cost] (Accumulated services and the cost of their accumulate calls)
    len++[Service ID ++ On-Transfer Cost] (Services which received transfers and the cost of their on-transfer calls)

Each service should be listed at most once in the accumulated services list, and at most once in
the list of services which received transfers.

## Block distribution events

These events concern announcement and transfer of blocks between peers.

### 60: Block announcement stream opened

Emitted when a block announcement stream (UP 0) is opened.

    Peer ID
    Connection Side (The side that opened the stream)

### 61: Block announcement stream closed

Emitted when a block announcement stream (UP 0) is closed. This need not be emitted if the stream
is closed due to disconnection.

    Peer ID
    Connection Side (The side that closed the stream)
    Reason

### 62: Block announced

Emitted when a block announcement is sent to or received from a peer (UP 0).

    Peer ID
    Connection Side (Announcer)
    Slot
    Header Hash

### 63: Blocks requested

Emitted when a sequence of blocks is requested from or by a peer (CE 128).

    Peer ID
    Connection Side (Requester)
    Header Hash
    0 (Ascending exclusive) OR 1 (Descending inclusive) (Direction, single byte)
    u32 (Maximum number of blocks)

### 64: Block request failed

Emitted if a block request received from or sent to a peer fails (CE 128).

    Event ID (ID of the corresponding "blocks requested" event)
    Reason

### 65: Block transferred

Emitted when a block has been fully sent to or received from a peer (CE 128).

    Event ID (ID of the corresponding "blocks requested" event)
    Slot
    Header Hash
    Block Summary
    bool (Last block for the request?)

## Ticket distribution events

These events concern distribution of tickets for the Safrole lottery.

### 80: Ticket transfer failed

Emitted when a Safrole ticket send or receive fails (CE 131/132).

    Peer ID
    Connection Side (Sender)
    bool (Was CE 132 used?)
    Reason

### 81: Ticket transferred

Emitted when a valid Safrole ticket is sent to or received from a peer (CE 131/132).

    Peer ID
    Connection Side (Sender)
    bool (Was CE 132 used?)
    Epoch Index (The epoch the ticket is to be used in)
    0 OR 1 (Single byte, ticket attempt number)
    [u8; 32] (VRF output)

### 82: Invalid ticket received

Emitted when an _invalid_ Safrole ticket is received from a peer (CE 131/132).

    Peer ID (Sender)
    bool (Was CE 132 used?)
    Epoch Index (The epoch the ticket was to be used in)
    0 OR 1 (Single byte, ticket attempt number)
    [u8; 784] (Bandersnatch ring VRF proof)
    Reason (Why is the ticket considered invalid?)

## Guaranteeing events

These events concern the guaranteeing pipeline and guarantee pool.

### 90: Work-package submission

Emitted when a builder opens a stream to submit a work-package (CE 133). This should be emitted as
soon as the stream is opened, before the work-package is read from the stream.

    Peer ID (Builder)

### 91: Work-package being shared

Emitted by the secondary guarantor when a work-package sharing stream is opened (CE 134). This
should be emitted as soon as the stream is opened, before any messages are read from the stream.

    Peer ID (Primary guarantor)

### 92: Work-package failed

Emitted if receiving a work-package from a builder or another guarantor fails, or processing of a
received work-package fails. This may be emitted at any point in the guarantor pipeline.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)
    Reason

### 93: Duplicate work-package

Emitted when a "duplicate" work-package is received from a builder or shared by another guarantor
(CE 133/134). A duplicate work-package is one that exactly matches (same hash) a previously
received work-package. This event should be emitted instead of a "work-package received" event.

In the case of a duplicate work-package received from a builder, no further events should be
emitted referencing the submission.

In the case of a duplicate work-package shared by another guarantor, only one more event should be
emitted referencing the "work-package being shared" event: either a "work-package failed" event
indicating failure or a "work-report signature sent" event indicating success.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)
    Core Index
    Work-Package Hash

### 94: Work-package received

Emitted once a work-package has been received from a builder or a primary guarantor (CE 133/134).
This should be emitted _before_ authorization is checked, and ideally before the extrinsic data is
received.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)
    Core Index
    Work-Package Hash
    Work-Package Summary

### 95: Authorized

Emitted once basic validity checks have been performed on a received work-package, including the
authorization check. This should be emitted by both primary (received the work-package from a
builder) and secondary (received the work-package from a primary) guarantors.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)
    Is-Authorized Cost

### 96: Extrinsic data received

Emitted once the extrinsic data for a work-package has been received from a builder or a primary
guarantor (CE 133/134) and verified as consistent with the extrinsic hashes in the work-package.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)

### 97: Imports received

Emitted once all the imports for a work-package have been fetched/received and verified.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)

### 98: Sharing work-package

Emitted by the primary guarantor when a work-package sharing stream is opened (CE 134).

    Event ID (ID of the corresponding "work-package submission" event)
    Peer ID (Secondary guarantor)

### 99: Work-package share failed

Emitted if sharing a work-package with another guarantor fails (CE 134). Possible failures include
failure to send the bundle, failure to receive a work-report signature, or receipt of an invalid
work-report signature. This event should only be emitted by the primary guarantor; the secondary
guarantor should emit the "work-package failed" event on failure.

    Event ID (ID of the corresponding "work-package submission" event)
    Peer ID (Secondary guarantor)
    Reason

### 100: Bundle sent

Emitted by the primary guarantor once a work-package bundle has been sent to a secondary guarantor
(CE 134).

    Event ID (ID of the corresponding "work-package submission" event)
    Peer ID (Secondary guarantor)

### 101: Refined

Emitted once a work-package has been refined locally. This should be emitted by both primary and
secondary guarantors.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)
    Work-Report Hash
    Work-Report Summary
    len++[Refine Cost] (Cost of refine call for each work item)

### 102: Work-report signature sent

Emitted once a work-report signature for a shared work-package has been sent to the primary
guarantor (CE 134).

    Event ID (ID of the corresponding "work-package being shared" event)

### 103: Work-report signature received

Emitted by the primary guarantor once a valid work-report signature has been received from a
secondary guarantor (CE 134). If an invalid work-report signature is received, a "work-package
share failed" event should be emitted instead.

    Event ID (ID of the corresponding "work-package submission" event)
    Peer ID (Secondary guarantor)

### 104: Distributing guarantee

Emitted when a guarantor begins distributing a work-report guarantee to other validators, for
potential inclusion in a block.

    Guarantee Summary

### 105: Sending guarantee

Emitted when a guarantor begins sending a work-report guarantee to another validator, for potential
inclusion in a block (CE 135).

    Event ID (ID of the corresponding "distributing guarantee" event)
    Peer ID (Recipient)

### 106: Guarantee send failed

Emitted if sending a work-report guarantee fails (CE 135).

    Event ID (ID of the corresponding "sending guarantee" event)
    Reason

### 107: Guarantee sent

Emitted if sending a work-report guarantee succeeds (CE 135).

    Event ID (ID of the corresponding "sending guarantee" event)

### 108: Receiving guarantee

Emitted by the recipient when a guarantor begins sending a work-report guarantee (CE 135).

    Peer ID (Sender)

### 109: Guarantee receive failed

Emitted if receiving a work-report guarantee fails (CE 135).

    Event ID (ID of the corresponding "receiving guarantee" event)
    Reason

### 110: Guarantee received

Emitted if receiving a work-report guarantee succeeds, and the guarantee is deemed valid (CE 135).

    Event ID (ID of the corresponding "receiving guarantee" event)
    Guarantee Summary

### 111: Invalid guarantee received

Emitted when an _invalid_ work-report guarantee is received from a peer (CE 135). Note that a
"guarantee receive failed" event should _not_ be emitted in this case.

    Event ID (ID of the corresponding "receiving guarantee" event)
    Guarantee Summary
    Reason (Why is the guarantee considered invalid?)

### 112: Guarantee discarded

Emitted when a guarantee is discarded from the local guarantee pool.

    Guarantee Summary
    Guarantee Discard Reason

## Availability events

These events concern availability shard distribution and retrieval.

### 120: Shard request sent

Emitted by assurers when they send a request for their shards to a guarantor (CE 137).

    Peer ID (Guarantor)
    Work-Report Hash (The work-report the shards are for)
    Erasure-Root (The erasure root of the work-report)
    Shard Index

### 121: Shard request received

Emitted by guarantors when they receive a shard request from an assurer (CE 137).

    Peer ID (Assurer)
    Erasure-Root
    Shard Index

### 122: Shard request failed

Emitted when a shard request fails (CE 137). This should be emitted by both sides, ie the assurer
and the guarantor.

    Event ID (ID of the corresponding "shard request sent" or "shard request received" event)
    Reason

### 123: Shards transferred

Emitted when a shard request completes successfully (CE 137). This should be emitted by both sides,
ie the assurer and the guarantor.

    Event ID (ID of the corresponding "shard request sent" or "shard request received" event)

### 124: Bundle shard request sent

Emitted by auditors when they send a bundle shard request to an assurer (CE 138).

    Peer ID (Assurer)
    Work-Report Hash (The work-report the shard is for)
    Erasure-Root (The erasure root of the work-report)
    Shard Index

### 125: Bundle shard request received

Emitted by assurers when they receive a bundle shard request from an auditor (CE 138).

    Peer ID (Auditor)
    Erasure-Root
    Shard Index

### 126: Bundle shard request failed

Emitted when a bundle shard request fails (CE 138). This should be emitted by both sides, ie the
auditor and the assurer.

    Event ID (ID of the corresponding "bundle shard request sent" or "bundle shard request received" event)
    Reason

### 127: Bundle shard transferred

Emitted when a bundle shard request completes successfully (CE 138). This should be emitted by both
sides, ie the auditor and the assurer.

    Event ID (ID of the corresponding "bundle shard request sent" or "bundle shard request received" event)

### 128: Segment shard request sent

Emitted by guarantors when they send a segment shard request to an assurer (CE 139/140).

    Peer ID (Assurer)
    bool (Was CE 140 used?)
    Work-Package Hash (The work-package importing the segments)
    u16 (Number of segment shards requested)

### 129: Segment shard request received

Emitted by assurers when they receive a segment shard request from a guarantor (CE 139/140).

    Peer ID (Guarantor)
    bool (Was CE 140 used?)
    u16 (Number of segment shards requested)

### 130: Segment shard request failed

Emitted when a segment shard request fails (CE 139/140). This should be emitted by both sides, ie
the guarantor and the assurer.

    Event ID (ID of the corresponding "segment shard request sent" or "segment shard request received" event)
    Reason

### 131: Segment shards transferred

Emitted when a segment shard request completes successfully (CE 139/140). This should be emitted by
both sides, ie the guarantor and the assurer.

    Event ID (ID of the corresponding "segment shard request sent" or "segment shard request received" event)

### 132: Distributing assurance

Emitted when an assurer begins distributing an assurance to other validators, for potential
inclusion in a block.

    Header Hash (Assurance anchor)
    [u8; ceil(C / 8)] (Availability bitfield; one bit per core, C is the total number of cores)

### 133: Assurance send failed

Emitted when an assurer fails to send an assurance to another validator (CE 141).

    Event ID (ID of the corresponding "distributing assurance" event)
    Peer ID (Recipient)
    Reason

### 134: Assurance sent

Emitted by assurers after sending an assurance to another validator (CE 141).

    Event ID (ID of the corresponding "distributing assurance" event)
    Peer ID (Recipient)

### 135: Assurance receive failed

Emitted when a validator fails to receive an assurance from a peer (CE 141).

    Peer ID (Sender)
    Reason

### 136: Assurance received

Emitted when a valid assurance is received from a peer (CE 141).

    Peer ID (Sender)
    Header Hash (Assurance anchor)

### 137: Invalid assurance received

Emitted when an _invalid_ assurance is received from a peer (CE 141).

    Peer ID (Sender)
    Header Hash (Assurance anchor)
    [u8; ceil(C / 8)] (Availability bitfield; one bit per core, C is the total number of cores)
    Ed25519 Signature (Assurance signature)
    Reason (Why is the assurance considered invalid?)

## Preimage distribution events

These events concern distribution of preimages for inclusion in blocks.

### 150: Preimage announce failed

Emitted when a preimage announcement fails (CE 142).

    Peer ID
    Connection Side (Announcer)
    Reason

### 151: Preimage announced

Emitted when a preimage announcement is sent to or received from a peer (CE 142).

    Peer ID
    Connection Side (Announcer)
    Service ID (Requesting service)
    Hash
    u32 (Preimage length)

### 152: Announced preimage forgotten

Emitted when a preimage announced by a peer is forgotten about. This event should not be emitted
for preimages the node managed to acquire (if such a preimage is discarded, a "preimage discarded"
event should be emitted instead).

    Service ID (Requesting service)
    Hash
    u32 (Preimage length)
    Announced Preimage Forget Reason

### 153: Preimage requested

Emitted when a preimage is requested from or by a peer (CE 143).

    Peer ID
    Connection Side (Requester)
    Hash

### 154: Preimage request failed

Emitted if a preimage request received from or sent to a peer fails (CE 143).

    Event ID (ID of the corresponding "preimage requested" event)
    Reason

### 155: Preimage transferred

Emitted when a preimage has been fully sent to or received from a peer (CE 143).

    Event ID (ID of the corresponding "preimage requested" event)
    u32 (Preimage length)

### 156: Preimage discarded

Emitted when a preimage is discarded from the local preimage pool.

    Hash
    u32 (Preimage length)
    Preimage Discard Reason

## Auditing events

TODO.

## Finality events

TODO.
