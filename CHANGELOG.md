# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.3

Documentation: the reference is generated from the code, and the
examples in it are doctests.  No code changed — every identifier this
mints is what 0.1.2 minted, and the dependency range is unchanged.

- **Every `pub` item is documented under Go's rule**, the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  The methods on `Uuid` carry
  their own.  `novo doc` turns the lot into
  [the package's page](https://novo-lang.org/packages/uuid-nv).
- **Ten worked examples, and they run.**  RFC 9562's own v4 and v7
  values are parsed and printed back, the seeded pair is shown minting
  the same identifier twice, and a v7 is shown sorting by the
  millisecond it was made.  The two that read the operating system's
  generator write their own `main`, since minting needs more than
  `[io]`.  A fenced `novo` block in a documentation comment is compiled
  by `novo doc` and run by `novo test src/uuid.nv`, so an example that
  stopped being true is a failing test rather than a reader's
  afternoon.

## 0.1.2

Developed in its own repository from this version.  `novolang/uuid-nv` is
where the sources live, where CI runs and where releases are tagged;
the novo-lang monorepo no longer carries a copy.  No code changed —
every signature and every byte on the wire is what 0.1.1 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball now carries the Apache-2.0 text rather
  than only naming it in the manifest.

## 0.1.1

A patch: the same identifiers, from the same bits.  RFC 9562's example
values, the version and variant fields on a v4 and a v7, and the
round trip through text and bytes all still pass.

- **The bit layout is written with the operators.**  The section that
  stamps the version and variant fields is where a reader checks this
  package against RFC 9562 §4.1 and §4.2, and the specification writes
  masks and shifts: `(self.hi >>> 12) & 0xf` for the version,
  `(1 << 63) | (lo & keep_62)` for the variant.  Nineteen `bits.*`
  calls in all, with the intermediates named — `stamp`, `kept`,
  `field`, `keep_62` — so each line says which field it is about.
  `std.bits` is no longer imported.

## 0.1.0

First release: `v4`, `v7`, `v4_from`, `v7_from`, `parse`, `from_bytes`,
`nil`, `max`, and `to_str`, `to_bytes`, `version`, `variant`,
`unix_ms` on a `Uuid`.

- **Two versions, chosen for what they are for.**  v4 for an identifier
  that must reveal nothing; v7 for one a database indexes, whose first
  48 bits are the Unix millisecond it was minted in.
- **A `Uuid` is two `Int`s**, so it costs sixteen bytes in a struct with
  no indirection, and `==` compares it.
- **Reproducible on demand.**  `v4_from` and `v7_from` take the
  generator — and the timestamp — so a fixture is the same on every run
  and a time-ordered identifier is testable at all.
- **Parses either case and both spellings**, canonical and hyphenless;
  prints the canonical lowercase form the specification asks for.
- **The vectors are RFC 9562's own example values**, so the suite is
  evidence about the specification's fields rather than about this
  implementation.
- **Host only for `v4` and `v7`**, which read the operating system's
  CSPRNG through `rand-nv`; a target without one answers `None` rather
  than falling back to something guessable.
- **Depends on `rand-nv ^0.1.0`**, the widest range this package
  supports: it calls only what that package shipped first, so a consumer
  already holding a `0.1.x` keeps it.
