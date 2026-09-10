# x25519-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

X25519 — the Diffie-Hellman of RFC 7748 — over Curve25519: a key pair
from 32 random bytes the caller supplies, the Montgomery ladder, and a
shared secret.  Everything is 32 bytes, everything is little-endian,
and a public key **is** a `u` coordinate: no format byte, no sign bit,
no second coordinate to get wrong.  That is why this is the exchange
everyone reaches for.

It is the other half of
[ed25519-nv](https://github.com/novolang/ed25519-nv), and it depends on
it for the field arithmetic the two schemes share.

## Adding it, and checking it

```bash
novo pkg add x25519-nv        # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test tests --isolate     # RFC 7748 §5.2 and §6.1, red until the bodies land
```

`novo test` is red today and that is the point of the release: every
assertion below the constants fails with `not implemented:
x25519-nv.x25519.<fn>`.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use std.list
use x25519

// The 32 bytes come from the host — rand-nv, an nRF52 RNG peripheral,
// /dev/urandom.  This package never asks for them itself.
fn session(entropy: [u8], peer_u: [u8]) -> ?[u8]
    match x25519.key_pair_from_random(entropy)
        Err(_) => None
        Ok(me) =>
            match x25519.public_key_from_bytes(peer_u)
                Err(_)   => None
                Ok(peer) =>
                    match x25519.diffie_hellman(me.secret, peer)
                        Err(_) => None
                        Ok(s)  => Some(s)
```

**The result is not a key.**  It is a `u` coordinate with structure in
it, and RFC 7748 § 6.1 says to run it through a key derivation
function.  hkdf-nv's `extract` over it, with the handshake transcript
as the salt, is what the protocols that get this right do.

## The layer, and why

`core`.  Every function is arithmetic over bytes the caller already
holds: nothing is read, nothing is written, no clock is consulted, and —
the one that decides the layer — **no randomness is drawn**.  A key pair
needs entropy, and entropy is a `[rand]` effect that belongs to the
host; so `key_pair_from_random` takes the bytes as an argument and this
package's whole surface stays `[]`.

`tests/embedded_probe.nv` is that claim as a program.  **It does not
build today**, for the two open toolchain defects ed25519-nv's probe
records as well: `Result<T, E>` cannot be spelled at `@tier(embedded)`,
because the `Error` trait is not in the prelude there; and a `[u8]`
cannot be *read* at that tier, because `list.get` is not on the
embedded surface while `list.len`, `list.is_empty`,
`list.with_capacity` and `list.try_push` are.  The signatures keep both
rather than retreating from either — a package whose refusals carry no
reason would be routing around the defect instead of recording it.

## The two rules

An X25519 implementation makes exactly two decisions that RFC 7748
leaves visible and that a caller **cannot read off the bytes**.  Both
are stated here, and `tests/x25519_rules_tests.nv` is them as cases so
that a reader who suspects otherwise can run them.

### Clamping happens inside the constructor

RFC 7748 § 5's `decodeScalar25519` clears the low three bits and bit
255 of a secret scalar and sets bit 254.  `secret_key_from_bytes` does
that, so a value of `X25519SecretKey` is always ready for the ladder
and no caller has to remember a step.

The visible consequence: **`secret_key_bytes` does not return the bytes
that went in.**  Up to five bits are gone.  A caller that persists a
key and later compares it with its own entropy will see a difference,
and the answer is to persist what comes out of `secret_key_bytes` — it
is the key; the entropy was raw material.

An implementation that clamped on the way *out* instead would agree
with this one on every shared secret and disagree on every stored key,
which is exactly the kind of difference that is invisible until two
systems try to share a database.

Clearing the low three bits also makes every scalar a multiple of the
cofactor 8, so a point of small order contributes nothing; setting bit
254 fixes the leading bit's position, so the ladder runs the same 255
steps whatever the secret.  Both are about the secret not being visible
in the timing, and neither is optional.

### An all-zero shared secret is refused

RFC 7748 § 6.1: "protocols MAY check for the all-zero value".  **This
package makes the check**, and `X25519SharedSecretIsZero` is the
refusal.

- An all-zero output means the peer sent one of the low-order points.
  The "shared" secret is then a constant the peer chose and already
  knows, so everything derived from it is known to the peer too.
- The protocols for which the check is unnecessary are the ones that
  authenticate the transcript afterwards, and they know who they are.
  The ones that do not are the ones written by somebody reaching for a
  Diffie-Hellman in a hurry.  A default that is safe for the second
  group and merely redundant for the first is the right default.
- A caller who genuinely wants the unchecked value has `scalar_mult`,
  one line away, named for what it is.

`is_low_order` asks the same question at the boundary, for a server
storing a client's static key: it should not wait until a session to
find out the key is worthless.  Checking the *output* stays the primary
route, because it catches every case without a table, and a table is a
thing that can be missing an entry.

**There is no `validate` here, and its absence is deliberate.**  P-256
needs one, because scalar multiplication against a point off that curve
leaks the private key — that is CVE-2018-5383 and p256-nv's `ecdh`
refuses rather than computing.  Curve25519 is *twist-secure*: a `u`
that is not on the curve is on the quadratic twist, whose order is also
large enough that nothing leaks.  So the invalid-curve check that
p256-nv must make has nothing to do here, and what remains is the
low-order points, which the zero check covers.

## The load-bearing interface

`scalar_mult` and `diffie_hellman` being two functions.

```novo
pub fn scalar_mult(k: X25519SecretKey, peer: X25519PublicKey) -> [u8] []
pub fn diffie_hellman(k: X25519SecretKey, peer: X25519PublicKey) -> Result<[u8], X25519Error> []
```

They compute the same ladder.  The first is the primitive and answers
whatever the arithmetic produces, zero included; the second is the key
exchange and refuses the zero.  Splitting them is what lets the safe
one be the default without taking the arithmetic away from the caller
who has a reason — and the return types say which is which before the
documentation does.

## Where the field comes from

`ed25519-nv`, whose `fe25519` module is the prime field
GF(2²⁵⁵ − 19) and the scalar clamping.  The two schemes share the field
and nothing above it: this one runs a Montgomery ladder that never
forms a `y` coordinate, Ed25519 works in extended Edwards coordinates
and forms both.  So the module boundary is exactly the field, and the
direction is the one the grid states — this package is split out of
ed25519-nv.

The alternative was a second copy of the field arithmetic, and the
argument against it is the other curve on the grid: p256-nv's reference
has point arithmetic that disagrees with RFC 6090 while every one of
its field-arithmetic checks passes.  There should be one field with one
set of vectors over it, not two that differ.

**The cost, named.**  Depending on ed25519-nv means depending on its
closure, and that includes crypto-nv for the SHA-512 that Ed25519 needs
and X25519 does not use at all.  Whether that survives dead-code
elimination is a measurement for the implementation lane; if it does
not, the answer is a `curve25519-nv` holding the field alone with both
schemes above it — a grid change, and not something this release should
pre-empt.

## The reference implementation

RFC 7748 § 5, whose pseudocode is the ladder verbatim, with
`x25519-dalek` and `libsodium` as second and third opinions.

**The test vectors are RFC 7748 § 5.2 and § 6.1**, in
`tests/x25519_vectors_tests.nv` now rather than later — the two raw
scalar multiplications, and the whole two-party exchange with both
sides computing the same secret from different halves.  Every expected
value was recomputed before it was written down: the exchange against a
reference X25519, the scalar multiplications against the ladder as § 5
writes it.

`tests/x25519_rules_tests.nv` carries the seven low-order encodings
that yield an all-zero secret against § 6.1's own Alice scalar — 0, 1,
the two points of order 8, and the non-canonical spellings `p − 1`, `p`
and `p + 1` — each one run through the ladder before it was written
down.

## Status

| item | implemented |
| --- | --- |
| `x25519.X25519_SECRET_KEY_BYTES`, `.X25519_PUBLIC_KEY_BYTES`, `.X25519_SHARED_SECRET_BYTES` | yes — they are constants |
| `x25519.X25519SecretKey`, `.X25519PublicKey`, `.X25519KeyPair`, `.X25519Error` | declared |
| `x25519.secret_key_from_bytes`, `.secret_key_bytes` | no |
| `x25519.public_key_from_bytes`, `.public_key_bytes`, `.public_key_of` | no |
| `x25519.key_pair_from_random` | no |
| `x25519.scalar_mult`, `.scalar_mult_base` | no |
| `x25519.diffie_hellman`, `.is_low_order` | no |
| `x25519.X25519Error.message` | no |
| the embedded probe | **does not build — two open toolchain defects, above** |

## Licence

Apache-2.0.  See `LICENSE`.
