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

    Block Outline =
        u32 (Size in bytes) ++
        Header Hash ++
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
        u32 (Number of transfers processed) ++
        u32 (Number of items accumulated) ++
        Exec Cost (Total) ++
        Exec Cost (read/write calls) ++
        Exec Cost (lookup calls) ++
        Exec Cost (query/solicit/forget/provide calls) ++
        Exec Cost (info/new/upgrade/eject calls) ++
        Exec Cost (transfer calls) ++
        Gas (Total gas charged for transfer processing by destination services) ++
        Exec Cost (Other host calls)

    Root Identifier = Segments-Root OR Work-Package Hash
    Import Spec =
        Root Identifier ++
        u16 (Export index, plus 2^15 if Root Identifier is a Work-Package Hash)
    Import Segment ID = u16 (Index in overall list of work-package imports, or for a proof page,
        2^15 plus index of a proven page)
    Work-Item Outline =
        Service ID ++
        u32 (Payload size) ++
        Gas (Refine gas limit) ++
        Gas (Accumulate gas limit) ++
        u32 (Sum of extrinsic lengths) ++
        len++[Import Spec] ++
        u16 (Number of exported segments)
    Work-Package Outline =
        u32 (Work-package size in bytes, excluding extrinsic data) ++
        Work-Package Hash ++
        Header Hash (Anchor) ++
        Slot (Lookup anchor slot) ++
        len++[Work-Package Hash] (Prerequisites) ++
        len++[Work-Item Outline]

    Work-Report Outline =
        Work-Report Hash ++
        u32 (Bundle size in bytes) ++
        Erasure-Root ++
        Segments-Root

    Guarantee Outline =
        Work-Report Hash ++
        Slot ++
        len++[Validator Index] (Guarantors)
    Guarantee Discard Reason =
        0 (Work-package reported on-chain) OR
        1 (Replaced by better guarantee) OR
        2 (Cannot be reported on-chain) OR
        3 (Too many guarantees) OR
        4 (Other)
        (Single byte)

    Announced Preimage Forget Reason =
        0 (Provided on-chain) OR
        1 (Not requested on-chain) OR
        2 (Failed to acquire preimage) OR
        3 (Too many announced preimages) OR
        4 (Bad length) OR
        5 (Other)
        (Single byte)
    Preimage Discard Reason =
        0 (Provided on-chain) OR
        1 (Not requested on-chain) OR
        2 (Too many preimages) OR
        3 (Other)
        (Single byte)

## Node information message

The first message sent on each connection to the telemetry server should contain information about
the connecting node:

    0 (Single byte, telemetry protocol version)
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

    u32 (Total number of peers)
    u32 (Number of validator peers)
    u32 (Number of peers with a block announcement stream open)
    [u8; C] (Number of guarantees in pool, by core; C is the total number of cores)
    u32 (Number of shards in availability store)
    u64 (Total size of shards in availability store, in bytes)
    u32 (Number of preimages in pool, ready to be included in a block)
    u32 (Total size of preimages in pool, in bytes)

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
    Option<Connection Side> (Terminator of the connection, may be omitted in case of eg a timeout)
    Reason

### 28: Peer misbehaved

Emitted when a peer misbehaves. Misbehaviour is any behaviour which is objectively not compliant
with the network protocol or the GP. This includes for example sending a malformed message or an
invalid signature. This does _not_ include, for example, timing out (timeouts are subjective) or
prematurely closing a stream (this is permitted by the network protocol).

    Peer ID
    Reason

## Block authoring/importing events

These events concern the block authoring and importing pipelines. Note that some events are common
to both authoring and importing, eg "block executed".

### 40: Authoring

Emitted when authoring of a new block begins.

    Slot
    Header Hash (Of the parent block)

### 41: Authoring failed

Emitted if block authoring fails for some reason.

    Event ID (ID of the corresponding "authoring" event)
    Reason

### 42: Authored

Emitted when a new block has been authored. This should be emitted as soon the contents of the
block have been determined, ideally before accumulation is performed and the new state root is
computed (which is included only in the following block).

    Event ID (ID of the corresponding "authoring" event)
    Block Outline

### 43: Importing

Emitted when importing of a block begins. This should not be emitted by the block author; the
author should emit the "authoring" event instead.

    Slot
    Block Outline

### 44: Block verification failed

Emitted if verification of a block being imported fails for some reason. This includes if the block
is determined to be invalid, ie it does not satisfy all the validity conditions listed in the GP.
In this case, a "peer misbehaved" event should also be emitted for the peer which sent the block.

This event should never be emitted by the block author (authors should emit the "authoring failed"
event instead).

    Event ID (ID of the corresponding "importing" event)
    Reason

### 45: Block verified

Emitted once a block being imported has been verified. That is, the block satisfies all the
validity conditions listed in the GP. This should be emitted as soon as this has been determined,
ideally before accumulation is performed and the new state root is computed. This should not be
emitted by the block author (the author should emit the "authored" event instead).

    Event ID (ID of the corresponding "importing" event)

### 46: Block execution failed

Emitted if execution of a block fails after authoring/verification. This can happen if, for
example, there is a collision amongst created service IDs during accumulation.

    Event ID (ID of the corresponding "authoring" or "importing" event)
    Reason

### 47: Block executed

Emitted following successful execution of a block. This should be emitted by both the block author
and importers.

    Event ID (ID of the corresponding "authoring" or "importing" event)
    len++[Service ID ++ Accumulate Cost] (Accumulated services and the cost of their accumulate calls)

Each service should be listed at most once in the accumulated services list.

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

### 63: Block request failed immediately

Emitted if a block request (CE 128) fails before the details of the request are sent/received. This
could happen for example if the node receiving the request is overloaded.

    Peer ID
    Connection Side (Requester)
    Reason

### 64: Blocks requested

Emitted when a sequence of blocks is requested from or by a peer (CE 128).

    Peer ID
    Connection Side (Requester)
    Header Hash
    0 (Ascending exclusive) OR 1 (Descending inclusive) (Direction, single byte)
    u32 (Maximum number of blocks)

### 65: Block request failed

Emitted if a block request received from or sent to a peer fails (CE 128).

    Event ID (ID of the corresponding "blocks requested" event)
    Reason

### 66: Block transferred

Emitted when a block has been fully sent to or received from a peer (CE 128).

In the case of a received block, this event may be emitted before any checks are performed. If the
block is found to be invalid or to not match the request, a "peer misbehaved" event should be
emitted; emitting a "block request failed" event is optional.

    Event ID (ID of the corresponding "blocks requested" event)
    Slot
    Block Outline
    bool (Last block for the request?)

## Safrole ticket events

These events concern generation and distribution of tickets for the Safrole lottery.

### 80: Generating tickets

Emitted when generation of a new set of Safrole tickets begins.

    Epoch Index (The epoch the tickets are to be used in)

### 81: Ticket generation failed

Emitted if Safrole ticket generation fails.

    Event ID (ID of the corresponding "generating tickets" event)
    Reason

### 82: Tickets generated

Emitted once a set of Safrole tickets has been generated.

    Event ID (ID of the corresponding "generating tickets" event)
    len++[[u8; 32]] (Ticket VRF outputs, index is attempt number)

### 83: Ticket transfer failed

Emitted when a Safrole ticket send or receive fails (CE 131/132).

    Peer ID
    Connection Side (Sender)
    bool (Was CE 132 used?)
    Reason

### 84: Ticket transferred

Emitted when a Safrole ticket is sent to or received from a peer (CE 131/132).

In the case of a received ticket, this should be emitted before the ticket is checked. If the
ticket is found to be invalid, a "peer misbehaved" event should be emitted.

    Peer ID
    Connection Side (Sender)
    bool (Was CE 132 used?)
    Epoch Index (The epoch the ticket is to be used in)
    0 OR 1 (Single byte, ticket attempt number)
    [u8; 32] (VRF output)

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
received work-package. This event may be emitted at any point in the guarantor pipeline. In
particular, it may be emitted instead of a "work-package received" event. An efficient
implementation should check for duplicates early on to avoid wasted effort!

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
This should be emitted _before_ authorization is checked, and ideally before the extrinsic data and
imports are received/fetched.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)
    Core Index
    Work-Package Outline

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

### 99: Work-package sharing failed

Emitted if sharing a work-package with another guarantor fails (CE 134). Possible failures include
failure to send the bundle, failure to receive a work-report signature, or receipt of an invalid
work-report signature (in this case, a "peer misbehaved" event should also be emitted for the
secondary guarantor). This event should only be emitted by the primary guarantor; the secondary
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
    len++[Refine Cost] (Cost of refine call for each work item)

### 102: Work-report built

Emitted once a work-report has been built for a work-package. This should be emitted by both
primary and secondary guarantors.

    Event ID (ID of the corresponding "work-package submission" or "work-package being shared" event)
    Work-Report Outline

### 103: Work-report signature sent

Emitted once a work-report signature for a shared work-package has been sent to the primary
guarantor (CE 134). This is the final event in the guaranteeing pipeline for secondary guarantors.

    Event ID (ID of the corresponding "work-package being shared" event)

### 104: Work-report signature received

Emitted by the primary guarantor once a valid work-report signature has been received from a
secondary guarantor (CE 134). If an invalid work-report signature is received, a "work-package
sharing failed" event should be emitted instead, as well as a "peer misbehaved" event.

    Event ID (ID of the corresponding "work-package submission" event)
    Peer ID (Secondary guarantor)

### 105: Guarantee built

Emitted by the primary guarantor once a work-report guarantee has been built. If a secondary
guarantor is slow to send their signature, this event may be emitted twice: once for the guarantee
with just two signatures, and again for the guarantee with all three signatures.

    Event ID (ID of the corresponding "work-package submission" event)
    Guarantee Outline

### 106: Sending guarantee

Emitted when a guarantor begins sending a work-report guarantee to another validator, for potential
inclusion in a block (CE 135). This should reference the "guarantee built" event corresponding to
the guarantee that is being sent.

    Event ID (ID of the corresponding "guarantee built" event)
    Peer ID (Recipient)

### 107: Guarantee send failed

Emitted if sending a work-report guarantee fails (CE 135).

    Event ID (ID of the corresponding "sending guarantee" event)
    Reason

### 108: Guarantee sent

Emitted if sending a work-report guarantee succeeds (CE 135).

    Event ID (ID of the corresponding "sending guarantee" event)

### 109: Guarantees distributed

Emitted by the primary guarantor once they have finished distributing the work-report guarantee(s).
This is the final event in the guaranteeing pipeline for primary guarantors.

This event may be emitted even if the guarantor was not successful in sending the guarantee(s) to
any other validator, although the guarantor may prefer to emit a "work-package failed" event in
that case.

    Event ID (ID of the corresponding "work-package submission" event)

### 110: Receiving guarantee

Emitted by the recipient when a guarantor begins sending a work-report guarantee (CE 135).

    Peer ID (Sender)

### 111: Guarantee receive failed

Emitted if receiving a work-report guarantee fails (CE 135).

    Event ID (ID of the corresponding "receiving guarantee" event)
    Reason

### 112: Guarantee received

Emitted if receiving a work-report guarantee succeeds (CE 135). This should be emitted before the
guarantee is checked. If the guarantee is found to be invalid, a "peer misbehaved" event should be
emitted.

    Event ID (ID of the corresponding "receiving guarantee" event)
    Guarantee Outline

### 113: Guarantee discarded

Emitted when a guarantee is discarded from the local guarantee pool.

    Guarantee Outline
    Guarantee Discard Reason

## Availability events

These events concern availability shard distribution and retrieval, and reconstruction of
bundles/segments.

### 120: Shard request failed immediately

Emitted if a shard request (CE 137) fails before the details of the request are sent/received. This
could happen for example if the node receiving the request is overloaded.

    Peer ID
    Connection Side (Requester)
    Reason

### 121: Shards requested

Emitted when an assurer requests their shards from a guarantor (CE 137). This event should be
emitted by both the assurer and the guarantor.

    Peer ID
    Connection Side (Requester)
    Erasure-Root
    Shard Index

### 122: Shard request failed

Emitted when a shard request fails (CE 137). This should be emitted by both sides, ie the assurer
and the guarantor.

    Event ID (ID of the corresponding "shards requested" event)
    Reason

### 123: Shards transferred

Emitted when a shard request completes successfully (CE 137). This should be emitted by both sides,
ie the assurer and the guarantor.

    Event ID (ID of the corresponding "shards requested" event)

### 124: Bundle shard request failed immediately

Emitted if a bundle shard request (CE 138) fails before the details of the request are
sent/received. This could happen for example if the node receiving the request is overloaded.

    Peer ID
    Connection Side (Requester)
    Reason

### 125: Bundle shard request sent

Emitted by auditors after they have sent a bundle shard request to an assurer (CE 138).

    Event ID (TODO, should reference auditing event)
    Peer ID (Assurer)
    Shard Index

### 126: Bundle shard request received

Emitted by assurers when they receive a bundle shard request from an auditor (CE 138).

    Peer ID (Auditor)
    Erasure-Root
    Shard Index

### 127: Bundle shard request failed

Emitted when a bundle shard request fails (CE 138). This should be emitted by both sides, ie the
auditor and the assurer.

    Event ID (ID of the corresponding "bundle shard request sent" or "bundle shard request received" event)
    Reason

### 128: Bundle shard transferred

Emitted when a bundle shard request completes successfully (CE 138). This should be emitted by both
sides, ie the auditor and the assurer.

    Event ID (ID of the corresponding "bundle shard request sent" or "bundle shard request received" event)

### 129: Reconstructing bundle

Emitted when reconstruction of a bundle from shards received from assurers begins.

    Event ID (TODO, should reference auditing event)
    bool (Is this a trivial reconstruction, using only original-data shards?)

### 130: Bundle reconstruction failed

Emitted if reconstruction of a bundle from shards fails.

    Event ID (ID of the corresponding "reconstructing bundle" event)
    Reason

### 131: Bundle reconstructed

Emitted once a bundle has been successfully reconstructed from shards.

    Event ID (ID of the corresponding "reconstructing bundle" event)

### 132: Segment shard request failed immediately

Emitted if a segment shard request (CE 139/140) fails before the details of the request are
sent/received. This could happen for example if the node receiving the request is overloaded.

    Peer ID
    Connection Side (Requester)
    Reason

### 133: Segment shard request sent

Emitted by guarantors after they have sent a segment shard request to an assurer (CE 139/140).

    Event ID (ID of the corresponding "work-package submission" event)
    Peer ID (Assurer)
    bool (Was CE 140 used?)
    len++[Import Segment ID ++ Shard Index] (Requested segment shards)

### 134: Segment shard request received

Emitted by assurers when they receive a segment shard request from a guarantor (CE 139/140).

    Peer ID (Guarantor)
    bool (Was CE 140 used?)
    u16 (Number of segment shards requested)

### 135: Segment shard request failed

Emitted when a segment shard request fails (CE 139/140). This should be emitted by both sides, ie
the guarantor and the assurer.

    Event ID (ID of the corresponding "segment shard request sent" or "segment shard request received" event)
    Reason

### 136: Segment shards transferred

Emitted when a segment shard request completes successfully (CE 139/140). This should be emitted by
both sides, ie the guarantor and the assurer.

    Event ID (ID of the corresponding "segment shard request sent" or "segment shard request received" event)

### 137: Reconstructing segments

Emitted when reconstruction of a set of segments from shards received from assurers begins.

    Event ID (ID of the corresponding "work-package submission" event)
    len++[Import Segment ID] (Segments being reconstructed)
    bool (Is this a trivial reconstruction, using only original-data shards?)

### 138: Segment reconstruction failed

Emitted if reconstruction of a set of segments fails.

    Event ID (ID of the corresponding "reconstructing segments" event)
    Reason

### 139: Segments reconstructed

Emitted once a set of segments has been successfully reconstructed from shards.

    Event ID (ID of the corresponding "reconstructing segments" event)

### 140: Segment verification failed

Emitted if, following reconstruction of a segment and its proof page, extraction or verification of
the segment proof fails. This should only be possible in two cases:

- CE 139 was used to fetch some of the segment shards. CE 139 responses are not justified;
  requesters cannot verify that returned shards are consistent with their erasure-roots.
- The erasure-root or segments-root is incorrect. This implies an invalid work-report for the
  exporting work-package.

For efficiency, multiple segments may be reported in a single event.

    Event ID (ID of the corresponding "work-package submission" event)
    len++[u16] (Indices of the failed segments in the import list)
    Reason

### 141: Segments verified

Emitted once a reconstructed segment has been successfully verified against the corresponding
segments-root. For efficiency, multiple segments may be reported in a single event.

    Event ID (ID of the corresponding "work-package submission" event)
    len++[u16] (Indices of the verified segments in the import list)

### 142: Distributing assurance

Emitted when an assurer begins distributing an assurance to other validators, for potential
inclusion in a block.

    Header Hash (Assurance anchor)
    [u8; ceil(C / 8)] (Availability bitfield; one bit per core, C is the total number of cores)

### 143: Assurance send failed

Emitted when an assurer fails to send an assurance to another validator (CE 141).

    Event ID (ID of the corresponding "distributing assurance" event)
    Peer ID (Recipient)
    Reason

### 144: Assurance sent

Emitted by assurers after sending an assurance to another validator (CE 141).

    Event ID (ID of the corresponding "distributing assurance" event)
    Peer ID (Recipient)

### 145: Assurance receive failed

Emitted when a validator fails to receive an assurance from a peer (CE 141).

    Peer ID (Sender)
    Reason

### 146: Assurance received

Emitted when an assurance is received from a peer (CE 141). This should be emitted as soon as the
assurance is received, before checking if it is valid. If the assurance is found to be invalid, a
"peer misbehaved" event should be emitted.

    Peer ID (Sender)
    Header Hash (Assurance anchor)

## Preimage distribution events

These events concern distribution of preimages for inclusion in blocks.

### 160: Preimage announcement failed

Emitted when a preimage announcement fails (CE 142).

    Peer ID
    Connection Side (Announcer)
    Reason

### 161: Preimage announced

Emitted when a preimage announcement is sent to or received from a peer (CE 142).

    Peer ID
    Connection Side (Announcer)
    Service ID (Requesting service)
    Hash
    u32 (Preimage length)

### 162: Announced preimage forgotten

Emitted when a preimage announced by a peer is forgotten about. This event should not be emitted
for preimages the node managed to acquire (if such a preimage is discarded, a "preimage discarded"
event should be emitted instead).

    Service ID (Requesting service)
    Hash
    u32 (Preimage length)
    Announced Preimage Forget Reason

### 163: Preimage request failed immediately

Emitted if a preimage request (CE 143) fails before the details of the request are sent/received.
This could happen for example if the node receiving the request is overloaded.

    Peer ID
    Connection Side (Requester)
    Reason

### 164: Preimage requested

Emitted when a preimage is requested from or by a peer (CE 143).

    Peer ID
    Connection Side (Requester)
    Hash

### 165: Preimage request failed

Emitted if a preimage request received from or sent to a peer fails (CE 143).

    Event ID (ID of the corresponding "preimage requested" event)
    Reason

### 166: Preimage transferred

Emitted when a preimage has been fully sent to or received from a peer (CE 143).

    Event ID (ID of the corresponding "preimage requested" event)
    u32 (Preimage length)

### 167: Preimage discarded

Emitted when a preimage is discarded from the local preimage pool.

Note that in the case where the preimage was requested by multiple services, there may not be a
unique discard reason. For example, the preimage may have been provided to one service, while
another service may have stopped requesting it. In this case, either reason may be reported.

    Hash
    u32 (Preimage length)
    Preimage Discard Reason

## Auditing events

TODO.

## Finality events

TODO.
