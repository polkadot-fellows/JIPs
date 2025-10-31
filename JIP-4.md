# JIP-4: Chainspec file

A chain specification collects information that describes a JAM-based network. It identifies the network a blockchain node connects to, the other nodes it initially communicates with, and the initial state that nodes must agree on to produce blocks. The chain specification can be defined in a JSON file.

## Chain Specification JSON Format

The chain specification root object includes the following keys:

- `id` - the machine-readable identifier for the network. This may be used as part of the network protocol identifier in the future version of the network protocol.
- `bootnodes` - an optional list of the nodes accepting connections. Each entry is a string in the following format: `<name>@<ip>:<port>` where `<name>` is the 53-character DNS name consisting of "e" followed by the Ed25519 public key, base-32 encoded using the alphabet "abcdefghijklmnopqrstuvwxyz234567". `<ip>` is a string containing IPv4 or IPv6 address of the node. IPv6 address may optionally be specified in square brackets (`[]`). `<port>` is an IP port number.
- `genesis_header` - A hex string containing JAM-serialized genesis block header.
- `genesis_state` - An object defining genesis state. Each key is a 62-character hex string defining the 31-byte state key. The values are arbitrary length hex strings.
- `protocol_parameters` - A hex string containing JAM-serialized protocol parameters. Encoding matches protocol parameters returned by the [`fetch`](https://graypaper.fluffylabs.dev/#/7e6ff6a/32e400324e01?v=0.6.7) host call defined in the Gray Paper (B.6)

The example below shows a basic chain specification file. Note that this does not contain a valid header or state for brevity.
```json
{
  "id": "testnet",
  "bootnodes": ["evysk4p563r2kappaebqykryquxw5lfcclvf23dqqhi5n765h4kkb@192.168.50.18:62061", "egy5qba5fyjf7hn7bxeroo7ncqfk5otxvo6or77k23o6pjqnxdoxb@192.168.50.20:63747"],
  "genesis_header": "1ee155ace9c40292074cb6aff8c9ccdd273c81648ff1149ef36bcea6ebb8a3e25bb30a42c1e62f0afda5f0a4e8a562f7a13a24cea00ee81917b86b89e801314aa4aa54d1a89973300d7e2493a1b512fecd848f4e8a63fb3a59d38a6b2c1610d9a2c98544eeb3df",
  "genesis_state": {
    "01000000000000000000000000000000000000000000000000000000000000": "08b647818aef53ffdf401882ab552f3ea21a57bdfe3fb4554a518a6fea139ca894b0",
    "09000000000000000000000000000000000000000000000000000000000000": "4aa54d1a89973300d7e2493a1b512fecd848f4e8a63fb3a59d38a6b2c1610d9a2c98",
    "0f000000000000000000000000000000000000000000000000000000000000": "000000000000000000000000",
    "0a000000000000000000000000000000000000000000000000000000000000": "0000",
    "00fe00ff00ff00ffa61a1135d89447673d804e5619daab939cd9c8936d4171": "5000156a616d2d626f6f7473747261702d7365727669636506302e312e32310a"
  },
  "protocol_parameters": "0a00000000000000010000000000000064000000000000000200004b00000c000000809698000000000080f0fa020000000000ca9a3b00000000002d310100000000080000001000080003004038000003000800060050000400000080000500060000fa0000017cd20000093d0004000000000c00000204000000c0000080000000000c00000a000000"
}
```

