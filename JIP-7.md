# JIP-7: PVM Ecalli Trace

This JIP describes a text format of the PVM execution Input-Output (ecalli) trace data.

## Goals

- Human-readable, newline-delimited text.
- Self-contained format to allow stateless re-execution.
- Additional logging of read memory to identify discrepancies between implementations.

## File structure

Each trace is an UTF-8 text file. Lines that do not match the expected format
are ignored. Note the line does not need to match fully to the format - it may have additional data prepended or appended, which should allow usage of existing logging infrastructure.

### Formatting guideliness

Hex-encoded data must be prefixed with `0x` for visual distinction from decimal numbers. If number formatting is not explicitly specified, decimal encoding must be used. The format also uses hex encoding for large data blobs, despite these being larger than e.g. base64, to allow simpler sub-blob searches (e.g. a particular constant value can be searched in read/written memory blobs).

### Comment lines (optional)

Comment lines may appear anywhere in the trace file. They are typically used to record:
1. Implementation metadata (e.g. implementation/pvm version, build hash, etc)
2. Execution environment (e.g. protocol parameters set (tiny/full), host call environment (refine, accumulate, etc))
3. Semantic details about the host call (name, changes, etc)

Each comment line must start with the `comment` keyword and everything following is treated as the comment content. Note that for textual comparison
of two traces originating from different implementations comments are best stripped out.

### Required prelude

The first mandatory log line must contain the program blob being executed, including metadata (if any). In JAM context this will be the service's code hash preimage value.

```
program {hex-encoded-program-with-metadata}
```

Next, any initial memory writes are expected. Each line uses the IO
syntax defined later. In JAM context this will contain SPI arguments.

```
memwrite {hex-encoded-address} len={blob-byte-length} <- {hex-encoded-bytes}
```

`{hex-encoded-address}` is a 0x-prefixed 32-bit hex value. `{blob-byte-length}` is the decimal length in bytes of the data blob. `{hex-encoded-bytes}` is an
even-length lowercase hex string containing full data that should be written to program memory after its initialization.

Finally, the initial execution state must be recorded:

```
start pc={pc} gas={gas} {register-dump}
```

- `{pc}`: decimal program counter at which execution begins.
- `{gas}`: initial gas value.
- `{register-dump}`: space-delimited list of `r{idx}={value}` pairs covering all non-zero registers at the start of execution. Registers with a zero value MUST be omitted. Register indices are 0-padded decimal values sorted ascending. Values are 0x-prefixed 64-bit hex with no padding and leading zeros trimmed.

## Ecalli entry format

An IO log entry begins whenever the VM executes an `ecalli` opcode. Entries are
multiple log lines consisting of the `ecalli` line with gas and register dump, followed by the host actions
performed while executing the call.

### `ecalli` line

```
ecalli={index} pc={pc} gas={gas} {register-dump}
```

- `{index}`: decimal host call index being invoked.
- `{pc}`: decimal program counter pointing to the `ecalli` opcode.
- `{gas}`: gas value before executing the host call.
- `{register-dump}`: space-delimited list of `r{idx}={value}` pairs covering all
  non-zero registers at the time of the call. Registers with a
  zero value MUST be omitted. Register indices are 0-padded decimal values sorted ascending. Values are 0x-prefixed 64-bit hex with no padding and leading zeros trimmed.

### Host actions

Each host action is a dedicated line after the `ecalli` line. There are
four action types, emitted in the following order:

1. **Memory reads** - ordered by increasing address

   `memread {hex-encoded-address} len={blob-byte-length} -> {hex-encoded-data-read}`

2. **Memory writes** - ordered by increasing address

   `memwrite {hex-encoded-address} len={blob-byte-length} <- {hex-encoded-data-to-write}`

3. **Register writes** - ordered by ascending register index

   `setreg r{idx} <- {hex-encoded-value}`

4. **Gas overwrite** - single entry altering the remaining gas. It's the last expected line so it's assumed that after that line the VM resumes.

   `setgas <- {gas}`

Addresses and values follow the same encoding rules as specified earlier.

### Multiple host actions

If the host performs multiple reads or writes for the same opcode, emit a
separate line per operation. Reads of overlapping ranges MUST be logged as the
implementation performs them; the address ordering requirement only applies to
the final list, so aggregate if necessary before emitting the lines.

## Requirements

- **Line order**: `ecalli` line first, then reads, memory writes followed by register writes and gas.
- **Sorting**: When multiple operations share the same address or register,
  break ties by comparing the full hex payload lexicographically.
- **Completeness**: Every observable side effect MUST be logged. Implementations
  cannot skip unchanged registers if the host explicitly
  touched them.
- **Whitespace**: Fields are separated by single spaces.

It should be possible to compare two execution logs from different implementations using a simple textual diff tool after stripping any unknown lines and "logger prefix/suffix".

## Execution Termination

The log MUST end with a termination line indicating how program execution concluded. The termination line follows a similar format to the `ecalli` line:

```
PANIC={argument} pc={pc} gas={gas} {register-dump}
OOG pc={pc} gas={gas} {register-dump}
HALT pc={pc} gas={gas} {register-dump}
```

- `PANIC={argument}` - Program terminated abnormally (trap code, invalid instruction code, invalid memory access, etc.)
- `OOG` - Program ran out of gas
- `HALT` - Program terminated normally
- `{pc}`: Decimal program counter at termination
- `{gas}`: Remaining gas at termination, saturated at 0 in case of underflow
- `{register-dump}`: Space-delimited list of `r{idx}={value}` pairs following the same encoding rules as the `ecalli` line (zero values omitted, 0-padded decimal indices, 0x-prefixed hex values)

## Example

Simplified synthetic example of the syntax. Note it's not semantically correct, hence it cannot be replayed.

```
comment implementation typeberry 0.8.3
comment chain-id fluffy-testnet
comment accumulate
program 0x0102aabbccddeeff
memwrite 0x00001000 len=8 <- 0x0000000000000001
start pc=0 gas=10000 r07=0x10 r09=0x10000

ecalli=10 pc=42 gas=9980 r01=0x1 r03=0x1000
memread 0x00001000 len=4 -> 0x01020304
memread 0x00001020 len=8 -> 0x0000000000000040
memwrite 0x00002000 len=2 <- 0xffee
setreg r00 <- 0x100
setreg r02 <- 0x4
setgas <- 9950

HALT pc=42 gas=9920 r00=0x100 r02=0x4
```

More examples can be found in [JAM Ecalli Trace](https://github.com/fluffylabs/jam-ecalli-trace) repository.


## Tooling support

[Anan-as PVM](https://github.com/tomusdrw/anan-as) already supports replaying trace files.

```
# Replay an ecalli trace
npx @fluffylabs/anan-as replay-trace trace.log
```

It is also possible to load the trace files in [PVM Debugger](https://pvm.fluffylabs.dev/) and replay the execution.


