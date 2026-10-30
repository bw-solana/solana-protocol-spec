# 2. Normative language and conventions

## 2.1 Requirement force

The uppercase terms **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY**
carry normative force only inside a paragraph bearing a requirement identifier. “Must”
indicates an absolute condition; “should” indicates a condition for which a documented,
interoperability-safe exception can exist; “may” grants an option. Lowercase forms and
“shall” have no normative force in this specification.

## 2.2 Numeric conventions

`u8`, `u16`, `u32`, `u64`, and `u128` denote unsigned integers of the stated width.
`i64` denotes a two's-complement signed 64-bit integer. `LE(x,n)` and `BE(x,n)` denote
the `n`-byte little- and big-endian encodings of integer `x`. Byte intervals use
half-open notation `[start,end)`. Unless a requirement states otherwise, mathematical
pseudocode uses unbounded integers; conversion to a fixed-width type is permitted only
after its stated range check.

Arithmetic that contributes to validity, state, scheduling, or a commitment uses the
checked, saturating, wrapping, or truncating behavior named by the applicable algorithm;
host-language default overflow behavior never determines a protocol result.

Byte strings concatenate with `||`. `len(x)` is a nonnegative mathematical integer.
Array indices are zero-based. A public-key comparison is lexicographic comparison of its
32 raw bytes unless stated otherwise.

## 2.3 Ordering and ties

When a threshold is written as `a/b > p`, equality does not pass. When written
`a/b >= p`, equality passes. Implementations use the exact integer cross-multiplication
form where one is specified; a floating-point expression quoted as implementation
evidence is not itself permission to substitute a numerically different comparison.

## 2.4 Diagrams, examples, and evidence

Tables and pseudocode labeled Normative are authoritative. Diagrams, rationale,
examples, source anchors, and implementation notes are informative unless an identified
requirement incorporates a value or relation they contain. The sources in Appendix D and
the SIMDs in Appendix C do not silently extend the numbered requirements.
