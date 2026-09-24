# uuid-nv

A UUID is a 128-bit identifier that two machines can produce at the same
moment without asking each other first. It is specified in
[RFC 9562](https://www.rfc-editor.org/rfc/rfc9562), which obsoletes RFC
4122. This package brings the two versions in current use to novo-lang,
version 4 and version 7, and reads a value of any version.

## What a UUID is

A UUID is 128 bits with two fields set aside inside them. Four bits are
the **version**, which names the rule that produced the other 124. Two
bits are the **variant**, which names the layout family the whole value
belongs to. RFC 9562 section 4.2 places the version field and section
4.1 places the variant field. A reader can therefore tell a version 4
value from a version 7 value without being told which it holds, which is
what lets one database column hold both.

Version 4 is 122 random bits and nothing else (section 5.4). It is
unpredictable and unordered, and it says nothing about when or where it
was made.

Version 7 is a 48-bit timestamp followed by 74 random bits (section
5.7). The timestamp is the number of milliseconds since 1970, the Unix
epoch, and it leads the value. Two version 7 identifiers therefore sort
in the order they were made. Section 6.11 says version 7 is designed so
that an implementation that has to sort, such as a database index, sorts
the raw bytes without parsing them.

Two values are named by the specification and produced by no version.
The Nil UUID is all 128 bits zero (section 5.9). The Max UUID is all 128
bits one (section 5.10). Their version and variant fields read as the
bits that are there, so neither is mistaken for a minted value.

The canonical text form is 32 hexadecimal digits in groups of 8, 4, 4, 4
and 12, with a hyphen between the groups. The specification says to
produce it in lowercase and to accept either case.

| Quantity | Value |
| --- | --- |
| Bits in a UUID | 128 |
| Bytes in a UUID | 16 |
| Characters in the canonical text form | 36 |
| Hexadecimal digits in it | 32 |
| Hyphens in it | 4 |
| Bits in the version field | 4 |
| Bits in the variant field | 2 |
| Random bits in a version 4 value | 122 |
| Timestamp bits in a version 7 value | 48 |
| Random bits in a version 7 value | 74 |
| The version 7 timestamp field runs out in | the year 10889 |

## Install

```
novo pkg add uuid-nv
```

## Example

```novo
use uuid

fn main() [io, fs, rand, time]
    // A version 4 value, whose 122 free bits come from the operating
    // system's generator.
    let id = uuid.v4()!
    println("${id.version()} ${id.variant()}")   // 4 2

    // A version 7 value, whose first 48 bits are the millisecond it
    // was made in.
    let key = uuid.v7()!
    println("${key.version()} ${key.unix_ms() > 0}")   // 7 true

    // The canonical text form, and the same value read back from it.
    let text = key.to_str()
    println("${uuid.parse(text) == Some(key)}")   // true
```

Build and test with `novo pkg build` and `novo test tests/uuid_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `uuid` | The `Uuid` type. The two constants, `nil` and `max`. The four ways to mint, `v4`, `v7`, `v4_from` and `v7_from`. The two ways to read one somebody else made, `parse` and `from_bytes`. The methods on the value: `to_str`, `to_bytes`, `version`, `variant` and `unix_ms`. |

The API reference is on
[the package's page](https://novo-lang.org/packages/uuid-nv). `novo doc`
generates it from these sources: every `pub` declaration with its
signature, its effect row and the comment block written above it. Each
entry carries a worked example that is compiled and run as a test.

## How to choose an entry point

**`uuid.v4` is the identifier that reveals nothing.** It is 122 bits
drawn from the operating system's generator. Reach for it for a token, a
public identifier, and anything an observer must not be able to order or
to date.

**`uuid.v7` is the identifier a database indexes.** Its first 48 bits
are the millisecond it was made in, so consecutive inserts land next to
each other in an index rather than scattering across it. The price is
that the value tells its reader when it was created.

**`uuid.v4_from` and `uuid.v7_from` take the randomness from the
caller.** The generator is an argument, and for `v7_from` the timestamp
is one too. This is the pair for a test that must mint the same
identifier twice, for a backfill that mints identifiers sorting where
the records they stand for belong, and for a target that has no
operating-system generator to read.

**`uuid.parse` and `uuid.from_bytes` read a value somebody else made.**
`parse` takes the text form and `from_bytes` takes sixteen bytes. Each
answers `None` rather than a partial value, and `version` and `variant`
then say what arrived.

## The rules a user needs

1. **This package mints version 4 and version 7, and it parses every
   version.** `version` and `variant` report the bits that are in the
   value rather than the bits this package would have written. A version
   1 or a version 6 value from another system reads as 1 or 6.
2. **The variant this package writes is 2.** That is the layout RFC 9562
   specifies, the two most significant bits of octet 8 set to `10`
   (section 4.1). `variant` also answers `0` or `1` for the older
   Apollo NCS layout, whose variant is a single zero bit, and `3` for
   Microsoft's layout and the range reserved for the future. A value whose variant is not 2 has a version field that means
   nothing.
3. **`v4` and `v7` are as unpredictable as the operating system's
   generator.** Both read `/dev/urandom` through
   [rand-nv](https://novo-lang.org/packages/rand-nv), which is a
   cryptographically secure pseudorandom number generator. RFC 9562
   section 6.9 is the specification's guidance on unguessability.
4. **`v4_from` and `v7_from` are as unpredictable as the generator
   handed to them, and no more.** A generator from `rng.seeded` is
   xoshiro256\*\*, which is fast and is not cryptographically secure. An
   observer who has seen a handful of its outputs can compute the rest,
   so an identifier minted from one belongs in a fixture rather than in
   a session token.
5. **A target with no operating-system generator gets `None`.** `v4` and
   `v7` answer `None` rather than falling back to something guessable.
   Seed an `Rng` from whatever entropy the target does have and use the
   `_from` pair there.
6. **`==` compares two values and there is no ordering operator.** Two
   version 7 values compare in the order they were minted when their
   bytes or their canonical text are compared, because the timestamp
   leads and hexadecimal preserves order. A version 4 value carries no
   order at all.
7. **Ordering is by the millisecond and no finer.** Two version 7 values
   minted in the same millisecond are ordered arbitrarily with respect
   to each other. RFC 9562 section 6.2 describes counter methods for a
   caller who needs finer ordering, and they need state this package
   does not keep.
8. **`parse` accepts two spellings and nothing else.** The canonical
   `8-4-4-4-12` form in either case, and the same 32 digits with the
   hyphens left out, which is what a text column usually holds. A hyphen
   anywhere but the four canonical positions is refused, so that two
   texts never name one value. `to_str` always produces the lowercase
   form, so a round trip through this package normalises an uppercase
   input.
9. **`from_bytes` takes exactly sixteen bytes, big-endian.** A length
   that is nearly right is refused rather than padded, because a UUID
   built from fifteen bytes and a zero is a different identifier and
   nothing downstream would notice. `to_bytes` writes the order RFC 9562
   lays a UUID out in, which is the order a wire field or a `BINARY(16)`
   column expects.
10. **`unix_ms` is meaningful on a version 7 value and on nothing
    else.** On a version 4 value those bits are random, so ask `version`
    first. The reading is the wall clock of the machine that minted the
    value, which is what makes two machines' version 7 identifiers
    comparable and also what limits what the number is worth.
11. **A millisecond above the 48-bit field is truncated to it.** The
    version nibble beside the field is left alone. The bound is the year
    10889.
12. **Two calls allocate and the rest do not.** `to_str` allocates the
    36-character string and `to_bytes` the sixteen-byte buffer. A `Uuid`
    is two `Int`s, so one in a struct or a list costs sixteen bytes with
    no indirection.

## What is not included

- **Versions 1, 2, 3, 5, 6 and 8 are not minted.** Version 1 and version
  6 need a MAC address and a stateful clock sequence. Version 3 and
  version 5 are name-based hashes over a namespace. Version 8 is by
  definition whatever its author decides. Each is a different package
  rather than an argument here. Reading any of them still works, because
  `parse`, `version` and `variant` are about the layout rather than
  about who wrote it.
- **A monotonic counter within one millisecond.** This package draws
  fresh randomness for every version 7 value rather than keeping a
  counter, so ordering stops at the millisecond. RFC 9562 section 6.2
  describes the counter methods, and they need state across calls.
- **Braces, a `urn:uuid:` prefix and surrounding whitespace.** `parse`
  refuses all three. A parser that guesses at the shape of its input
  accepts strings its writer never meant.
- **A comparison that orders two values.** `==` answers whether two
  values are the same. A caller that needs an order compares `to_bytes`
  or `to_str`.
- **A build for a microcontroller.** `v4` and `v7` read a file on the
  host, so the package declares `layer = "host"` and ships no device
  build. The `_from` pair is arithmetic over bits the caller supplies.

## Related packages

- [rand-nv](https://novo-lang.org/packages/rand-nv) is where the random
  bits come from. It holds the operating system's generator and the
  seedable xoshiro256\*\* that the `_from` pair takes.
- [ulid-nv](https://novo-lang.org/packages/ulid-nv) is the other
  128-bit sortable identifier. A ULID is a 48-bit millisecond timestamp
  followed by 80 random bits, written as 26 characters of Crockford
  base32 rather than as 36 characters of hexadecimal.

## Tests

```
novo test tests/uuid_tests.nv
```

The vectors are the example values RFC 9562 prints in Appendix A. The
version 4 example is `919108f7-52d1-4320-9bac-f847db4148a8` and the
version 7 example is `017F22E2-79B0-7CC3-98C4-DC0C0C07398F`, whose
timestamp field names 2022-02-22T19:22:22Z. Each is parsed, read field
by field and printed back. Testing against the specification's own
values rather than against this implementation's output is what makes
the suite evidence about the format.

Beside the vectors the suite asserts the two named constants, both
spellings and both cases of the text form, five inputs that are nearly a
UUID and are refused, the round trip through sixteen bytes, and the
fields each version promises over 64 draws from a seeded generator. It
asserts that version 7 values sort in the order they were minted, that
two minted in one millisecond still differ, and that a millisecond above
the 48-bit field is truncated rather than allowed to reach the version
nibble. Every example in a documentation comment is compiled by `novo
doc` and run by `novo test`, so an example that stopped being true is a
failing test.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
