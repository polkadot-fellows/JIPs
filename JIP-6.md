# JIP-6: Program metadata

PVM code blobs as defined in the Gray Paper begin with a variable-length metadata section. This JIP
proposes a standard format for this metadata.

## Format

Given three octet sequences which are UTF-8 encoded strings program-name $\mathbf{p} \in \mathbb{Y}$, version $\mathbf{p} \in \mathbb{Y}$ and licence $\mathbf{p} \in \mathbb{Y}$, together with a sequence of authors, each of them such strings $\mathbf{A} \in [\mathbb{Y}]$, we define the encoded metadata $\mathbf{m}$ as:

$$
\mathbf{m} \equiv [0] \frown \mathcal{E}(\left\updownarrow\mathbf{p}\right., \left\updownarrow\mathbf{p}\right., \left\updownarrow\mathbf{p}\right., \left\updownarrow[\left\updownarrow\mathbf{a}\right.\mid\mathbf{a} \in \mathbf{A}]\right.)
$$

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
