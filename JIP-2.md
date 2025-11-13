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
