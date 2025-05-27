# JIP-6: Program metadata

PVM code blobs as defined in the Gray Paper begin with a variable-length metadata section. This JIP
proposes a standard format for this metadata.

## Format

The metadata section should begin with the following, serialized as per the codec defined in the
Gray Paper, with `len++[...]` meaning a length-prefixed sequence and `String` meaning a
byte-length-prefixed UTF-8 string:

    0 (Single byte)
    String (Program name)
    String (Version)
    String (License)
    len++[String] (Authors)

Writers of metadata should not include any data beyond this, however any such data should be
ignored by parsers.

## Further conventions

This section describes further conventions on the format of individual metadata fields. Metadata
parsers should gracefully handle metadata that does not conform to these conventions.

The program name should contain only alphanumeric characters, underscores, and hyphens. Rust
programs should use the crate name as the program name.

The license field should contain an SPDX license expression.

Each member of the authors sequence should identify a person or organization. An email address may
be included within angled brackets at the end of each author entry. For example, `Parity
Technologies <admin@parity.io>`.
