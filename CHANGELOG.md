# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

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
