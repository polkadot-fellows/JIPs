# JIP-2: Node RPC

RPC specification for JAM nodes to ensure JAM tooling which relies on being an RPC client is implementation-agnostic.

## Notes

RPCs are evil and should generally not be used: they lead to chronic centralisation and trust-maximisation (just see Ethereum for a great example of this trap).

However, there are not at present any light clients for JAM and the resources needed to run a regular client make the inclusion of a full-client inside of tooling to be unrealistic. We must therefore reluctantly presume that the tool-user has access to a trustworthy full node and can use its RPCs.

As soon as a light-client implementation is viable, the use of RPCs should be phased out immediately in favour of embedded light-clients in the tooling.

## RPC specification

Both arguments, results and notification types are passed as an array, which may be `null` for certain results (marked as such).

Types:
- `Hash`: a 32 item array with each item numeric between 0 and 255 inclusive.
- `Slot`: a single numeric item between 0 and $2^{32}-1$ inclusive.
- `Blob`: an arbitrary length array each item numeric between 0 and 255 inclusive.
- `ServiceId`: a single numeric item between 0 and $2^{32}-1$ inclusive.
- `Parameters`: an object describing the parameters of the JAM chain. See below for more information.

Generally JSON RPC can be used over a variety of media and we don't make any assumptions of this, but it is envisaged that Websockets will be the usual medium, on port 19800.

Subscriptions are handled in the usual way for JSON-RPC. A counterpart `unsubscribe...` RPC should be assumed for all `subscribe` RPCs; these are ommitted for brevity.

### `parameters`
Returns the parameters of the current node/chain.
- Type: __Method__
- Arguments: None
- Result:
  - `V1`: The (version 1) `Parameters` object. See below.

### `bestBlock`
Returns the header hash and slot of the head of the "best" chain.
- Type: __Method__
- Arguments: None
- Result:
  - `Hash`: The header hash.
  - `Slot`: The slot.

### `subscribeBestBlock`
Subscribe to updates of the head of the "best" chain, as returned by `bestBlock`.
- Type: __Subscription__
- Notification type:
  - `Hash`: The header hash.
  - `Slot`: The slot.

### `finalizedBlock`
Returns the header hash and slot of the latest finalized block.
- Type: __Method__
- Arguments: None
- Result:
  - `Hash`: The header hash.
  - `Slot`: The slot.

### `subscribeFinalizedBlock`
Subscribe to updates of the latest finalized block, as returned by `finalizedBlock`.
- Type: __Subscription__
- Notification type:
  - `Hash`: The header hash.
  - `Slot`: The slot.

### `parent`
Returns the header hash and slot of the parent of the block with the given header hash, or `null` if this is not known.
- Type: __Method__
- Arguments: 
  - `Hash`: The hash of a child's header.
- Result: Either `null` or:
  - `Hash`: The parent's header hash.
  - `Slot`: The slot.

### `stateRoot`
Returns the posterior state root of the block with the given header hash, or `null` if this is not known.
- Type: __Method__
- Arguments: 
  - `Hash`: The header hash.
- Result: Either `null` or:
  - `Hash`: state_root

### `statistics`
Returns the activity statistics stored in the posterior state of the block with the given header hash. The statistics are encoded as per the GP. `null` is returned if the block's posterior state is not known.
- Type: __Method__
- Arguments: 
  - `Hash`: The header hash indicating the block whose posterior state should be used for the query.
- Result: `Blob`

### `subscribeStatistics`
Subscribe to updates of the activity statistics stored in chain state. If `finalized` is true, the subscription will track the latest finalized block. If `finalized` is false, the subscription will track the head of the "best" chain. Note that in the latter case the reported statistics may never be included in the finalized chain. The statistics are encoded as per the GP.
- Type: __Subscription__
- Notification type: `Blob`

### `serviceData`
Returns the service data for the given service ID. The data are encoded as per the GP. `null` is returned if the block's posterior state is not known. `Some(None)` is returned if there is no value associated with the given service ID.
- Type: __Method__
- Arguments: 
  - `Hash`: The header hash indicating the block whose posterior state should be used for the query.
  - `ServiceId`: The ID of the service.
- Result: Either `null` or `Blob`

### `subscribeServiceData`
Subscribe to updates of the service data for the given service ID. If `finalized` is true, the subscription will track the latest finalized block. If `finalized` is false, the subscription will track the head of the "best" chain. Note that in the latter case the reported service info may never be included in the finalized chain. The data are encoded as per the GP.
- Type: __Subscription__
- Notification type: `Blob`

### `serviceValue`
Returns the value associated with the given service ID and key in the posterior state of the block with the given header hash. `null` is returned if there is no value associated with the given service ID and key.
- Type: __Method__
- Arguments: 
  - `Hash`: The header hash indicating the block whose posterior state should be used for the query.
  - `ServiceId`: The ID of the service.
  - `Blob`: The key.
- Result: Either `null` or `Blob`

### `subscribeServiceValue`
Subscribe to updates of the value associated with the given service ID and key. If `finalized` is true, the subscription will track the latest finalized block. If `finalized` is false, the subscription will track the head of the "best" chain. Note that in the latter case reported value changes may never be included in the finalized chain. The `value` field of subscription messages will be `null` when there is no value associated with the given service ID and key.
- Type: __Subscription__
- Notification type: Either `null` or `Blob`

### `servicePreimage`
Returns the preimage associated with the given service ID and hash in the posterior state of the block with the given header hash. `null` is returned if there is no preimage associated with the given service ID and hash.
- Type: __Method__
- Arguments: 
  - `Hash`: The header hash indicating the block whose posterior state should be used for the query.
  - `ServiceId`: The ID of the service.
  - `Hash`: The hash.
- Result: Either `null` or `Blob`

### `subscribeServicePreimage`
Subscribe to updates of the preimage associated with the given service ID and hash. If `finalized` is true, the subscription will track the latest finalized block. If `finalized` is false, the subscription will track the head of the "best" chain. Note that in the latter case reported preimage changes may never be included in the finalized chain. The `preimage` field of subscription messages will be `null` when there is no preimage associated with the given service ID and hash.
- Type: __Subscription__
- Notification type: Either `null` or `Blob`
    
### `serviceRequest`
Returns the preimage request associated with the given service ID and hash/len in the posterior state of the block with the given header hash. `null` is returned if there is no preimage request associated with the given service ID, hash and length.
- Type: __Method__
- Arguments: 
  - `Hash`: The header hash indicating the block whose posterior state should be used for the query.
  - `ServiceId`: The ID of the service.
  - `Hash`: The hash.
  - `u32`: The preimage length.
- Result: Either `null` or array of `Slot`

### `subscribeServiceRequest`
Subscribe to updates of the preimage associated with the given service ID and hash. If `finalized` is true, the subscription will track the latest finalized block. If `finalized` is false, the subscription will track the head of the "best" chain. Note that in the latter case reported preimage changes may never be included in the finalized chain. The `request` field of subscription messages will be `null` when there is no preimage request associated with the given service ID, hash and length.
- Type: __Subscription__
- Notification type: Either `null` or array of `Slot`
    
### `beefyRoot`
Returns the BEEFY root of the block with the given header hash, or `null` if this is not known.
- Type: __Method__
- Arguments: 
	`Hash`: The header hash.
- Result: Either `null` or:
  - `Hash`: The BEEFY root.

### `submitWorkPackage`
Submit a work-package to the guarantors currently assigned to the given core.
- Type: __Method__
- Arguments: 
  - `CoreIndex`: The index of the core.
  - `Blob`: The encoded work-package.
  - array of `Blob`: The extrinsics.
- Result: None

### `submitPreimage`
Submit a preimage which is being requested by a given service.
- Type: __Method__
- Arguments: 
  - `ServiceId`: The ID of the service which has an outstanding request.
  - `Blob`: The preimage requested.
  - `Hash`: The block which must be in the best-chain for the preimage request to be valid.
- Result: None

### `listServices`
Returns a list of all services currently known to be on JAM. This is a best-effort list and may not reflect the true state. Nodes could e.g. reasonably hide services which are not recently active from this list.
- Type: __Method__
- Arguments: 
  - `Hash`: The header hash indicating the block whose posterior state should be used for the query.
- Result: array of `ServiceId`

## Chain Parameters

The `Parameters` object describes a JAM chain parameterization, which may not be equivalent to the canonical parameterization of the Gray Paper. All fields are numeric.

- $\mathsf{B}_S$ `deposit_per_account`: The base deposit required to retain an account.
- $\mathsf{B}_I$ `deposit_per_item`: The additional deposit required for each preimage or storage item in an account.
- $\mathsf{B}_L$ `deposit_per_byte`: The additional deposit required for each byte of each storage item in an account and preimage of an account.
- $\mathsf{D}$ `min_turnaround_period`: Minimum period in blocks between going from becoming `Available` to `Zombie`, and then again from `Zombie` to non-existent.
- $\mathsf{E}$ `epoch_period`: The epoch period, defined in number of slots.
- $\mathsf{G}_A$ `max_accumulate_gas`: Maximum gas which may be used to Accumulate a single work-report.
- $\mathsf{G}_I$ `max_is_authorized_gas`: Maximum gas which may be used to Authorize a single work-package.
- $\mathsf{G}_R$ `max_refine_gas`: Maximum gas which may be used to Refine a single work-report.
- $\mathsf{G}_T$ `block_gas_limit`: Maximum gas which can be processed in a single block.
- $\mathsf{H}$ `recent_block_count`: The number of blocks which are kept in the recent block cache.
- $\mathsf{I}$ `max_work_items`: Maximum number of Work Items in a Work Package.
- $\mathsf{J}$ `max_dependencies`: Maximum number of dependencies (total of prerequisites and SR lookup entries).
- $\mathsf{K}$ `max_tickets_per_block`: Max tickets allowed to be embedded in each block extrinsic.
- $\mathsf{L}$ `max_lookup_anchor_age`: Maximum age, in blocks, that the lookup anchor may be, taken from the regular anchor.
- $\mathsf{N}$ `tickets_attempts_number`: The number of distinct tickets which may be created and submitted by each validator on each epoch.
- $\mathsf{O}$ `auth_window`: Number of items in the authorization window.
- $\mathsf{Q}$ `auth_queue_len`: Number of authorizations in a queue allocated to a core.
- $\mathsf{R}$ `rotation_period`: The rotation period, defined in number of slots.
- $\mathsf{T}$ `max_extrinsics`: Maximum number of extrinsics in a Work Package.
- $\mathsf{U}$ `availability_timeout`: The period in timeslots after which reported but unavailable work may be replaced.
- $\mathsf{V}$ `val_count`: Total number of validators.
- $\mathsf{W}_B$ `max_input`: Maximum size of a Work Package together with all extrinsic data and imported segments.
- $\mathsf{W}_C$ `max_refine_code_size`: The maximum size of Refine/Accumulate code.
- $\mathsf{W}_E$ `basic_piece_len`: Number of octets in a erasure-coded piece.
- $\mathsf{W}_M$ `max_imports`: Maximum number of imports in a Work Package.

There are additional parameters which are not (yet) described in the Gray Paper, but which clients may find useful to know:
- $\mathsf{W}_I$ `max_is_authorized_code_size`: The maximum size of Is-Authorized code.
- $\mathsf{W}_X$ `max_exports`: Maximum number of exports in a Work Package.
- `max_refine_memory`: The maximum amount of RAM which may be used by Refine/Accumulate code.
- `max_is_authorized_memory`: The maximum amount of RAM which may be used by IsAuthorized code.

All parameters not described are assumed to be their canonical values. Some parameters are dependent on other values:
- $\mathsf{C} = \frac{\mathsf{V}}{3}$: The number of validators per core is always 3.
- $\mathsf{W}_G = 4,104$: The size of a (reconstructed) segment is fixed.
- $\mathsf{W}_P = \frac{\mathsf{W}_G}{\mathsf{W}_E}$: The number of EC pieces in a segment.
