> Developed and published from this repository.  `uuid-nv` began under
> `orbit/uuid-nv` in the [novo-lang](https://github.com/novolang) monorepo
> and graduated out of it with its history; issues and pull requests
> belong here.

# uuid-nv

RFC 9562 UUIDs, in the two versions worth minting today. A UUID is 128
bits with four of them naming the **version** — which rule produced the
other 124 — and two naming the **variant**, the layout family the value
belongs to. A reader can therefore tell a v4 from a v7 without being
told, which is what lets one column hold both.

```novo
use uuid

fn main() [io, fs, rand, time]
    let id = uuid.v4()!                   // reveals nothing
    let key = uuid.v7()!                  // sorts by when it was made
    println(key.to_str())                 // 019250a1-7c3f-7b21-8e0c-…
```

```
novo pkg add uuid-nv
```

## Which version to use

| | v4 | v7 |
|---|---|---|
| Contents | 122 random bits | 48-bit Unix millisecond timestamp, then 74 random bits |
| Sorts by | nothing | when it was minted |
| Reveals | nothing | the millisecond it was minted |
| Use it for | a token, a public identifier, anything an observer must not order or date | a primary key, a log record, anything a database indexes |

**v7 is the default choice for a stored key.** Rows keyed by v4 land at
random positions in an index, so every insert dirties a different page;
rows keyed by v7 land at the end, and the index stays dense. The price
is that a v7 tells its reader when it was created, which is why a public
identifier should still be a v4.

## What it gives you

The API is on [the package's page](https://novo-lang.org/packages/uuid-nv),
generated from these sources: every `pub` declaration with its signature,
its effect row and the comment block written above it. A table of names
here would be a second original, and the second original is the one that
goes stale.

A `Uuid` is two `Int`s, so it costs sixteen bytes in a struct with no
indirection, and `==` compares it.

## Minting the same identifier twice

`v4` and `v7` read the operating system's generator, so no two calls
agree and no test can pin one. The `_from` pair takes a generator you
supply instead:

```novo
use rng
use uuid

// The same fixture on every run, on every machine.
fn fixture() -> Uuid
    var r = rng.seeded(7)
    uuid.v4_from(r)
```

The generator decides how unpredictable the result is. A seeded `Rng` is
xoshiro256\*\*, which is fast and not cryptographically secure, so a
UUID minted from one belongs in a fixture and not in a session token.

`v7_from` takes the timestamp as well, which is what makes a
time-ordered identifier testable at all — the same milliseconds in, the
same order out.

## Reading one somebody else made

`parse` accepts the canonical `8-4-4-4-12` form in either case, and the
same 32 digits with the hyphens left out — which is the shape a text
column usually holds. Nothing else: no braces, no `urn:uuid:` prefix, no
surrounding whitespace. A parser that guesses at the shape of its input
accepts strings its writer never meant, and then two texts name one
value.

`to_str` always produces lowercase. That is what the specification says
to write, and it is why a round trip through this package normalises an
uppercase input.

`version` and `variant` are what a reader asks about a value it did not
mint. Both report what is there rather than what this package would have
put there: a v1 or a v6 from another system reads as `1` or `6`, and
`variant` answers `0` or `3` for the older Apollo and Microsoft layouts.

## What it costs

One read of `/dev/urandom` per identifier, or two 64-bit draws from a
seeded generator, then a handful of shifts and masks. `to_str`
allocates the 36-character string and `to_bytes` the sixteen-byte
buffer; nothing else here allocates.

**Host only, for `v4` and `v7`.** Both draw from `rand-nv`'s operating
system source, which is `/dev/urandom`. A target without that file gets
`None` rather than a fallback to something guessable — seed an `Rng`
from a hardware entropy peripheral and use the `_from` pair there.

## What it does not do

No v1, v3, v5, v6 or v8. v1 and v6 need a MAC address and a stateful
clock sequence, v3 and v5 are name-based hashes over a namespace, and v8
is by definition whatever its author decides — each is a different
package rather than a flag here. Reading any of them still works:
`parse`, `version` and `variant` are about the layout, not about who
minted it.

No monotonic counter within a millisecond. Two v7s minted in the same
millisecond are ordered arbitrarily with respect to each other; RFC 9562
§6.2 describes counter methods for callers who need finer ordering, and
they need state this package does not keep.

## Dependencies

It depends on [`rand-nv`](https://novo-lang.org/packages/rand-nv) for the
random bits, and for the seedable generator the `_from` pair takes. The
range is on [the package's page](https://novo-lang.org/packages/uuid-nv),
read from this manifest.

The range is the widest this package supports, because it calls only
what `rand-nv` shipped in its first release. That matters to you rather
than to us: one version of a package is built into a program, so a range
that excluded a release you already hold would be a version conflict you
had to resolve.

The millisecond clock a v7 needs is `Zoned.now`, which is in the
standard library — so adding this package brings one package with it,
and it is pinned in your `novo.lock` alongside.

## Tests

```
novo test tests/uuid_tests.nv
```

The vectors are the example values RFC 9562 prints in its appendix — the
v4 `919108f7-52d1-4320-9bac-f847db4148a8` and the v7
`017F22E2-79B0-7CC3-98C4-DC0C0C07398F`, whose timestamp field is
2022-02-22T19:22:22Z — parsed, read field by field and printed back.
Beyond those: the two named constants, both spellings and both cases of
the text form, five inputs that are nearly a UUID and are refused, and
the layout each version promises over 64 draws from a seeded generator,
including that v7s sort in the order they were minted.
