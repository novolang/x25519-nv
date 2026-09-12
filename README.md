# x25519-nv

X25519 is an elliptic-curve Diffie-Hellman key exchange over Curve25519,
specified in [RFC 7748](https://www.rfc-editor.org/rfc/rfc7748). Two parties who
have never met each other each publish a 32-byte public key and arrive at the
same 32-byte shared secret. This package brings it to novo-lang.

It is the exchange half of Curve25519. The signature half is
[ed25519-nv](https://novo-lang.org/packages/ed25519-nv), which supplies the
field arithmetic both schemes share and is this package's only dependency.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What X25519 is

A Diffie-Hellman key exchange lets two parties derive a shared secret over a
channel anyone can read. Each party picks a secret scalar, multiplies a fixed
base point by it, and sends the result. Each then multiplies the value it
received by its own scalar. Both arrive at the same point, and an eavesdropper
who saw only the two public values cannot compute it.

X25519 performs that multiplication on Curve25519 using the *Montgomery ladder*,
a loop that runs the same 255 steps whatever the scalar is. It works with one
coordinate, called `u`, and never forms the other one.

Everything in X25519 is 32 bytes and everything is little-endian. A public key
*is* a `u` coordinate: there is no format byte, no sign bit and no second
coordinate to get wrong. That is why this is the exchange most new protocols
reach for.

| Value | Size |
| --- | --- |
| Secret key | 32 bytes |
| Public key | 32 bytes |
| Shared secret | 32 bytes |

## Install

```
novo pkg add x25519-nv
```

## Example

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

Build and test with:

```
novo pkg build                # type- and effect-check the package
novo test tests --isolate     # RFC 7748 section 5.2 and section 6.1
```

Today `novo test` fails on purpose: every test reaches a
`not implemented: x25519-nv.x25519.<fn>` panic. The tests are the specification
the implementation will have to satisfy.

## What the package contains

One module.

| Module | Contents |
| --- | --- |
| `x25519` | The whole surface: the three 32-byte size constants; the key types `X25519SecretKey`, `X25519PublicKey` and `X25519KeyPair`; the constructors and accessors; the two multiplications `scalar_mult` and `scalar_mult_base`; the exchange `diffie_hellman`; the small-order test `is_low_order`; and the error type `X25519Error`. |

No function in this package draws randomness, reads a clock, or performs input
or output. The 32 bytes a key pair needs arrive as an argument, which is what
lets the same code run in firmware.

## How to choose an entry point

**Most programs call `key_pair_from_random` and then `diffie_hellman`.** The
first turns 32 bytes of entropy you supply into a clamped secret key and its
public key. The second performs the exchange and refuses an all-zero result.

**Call `scalar_mult` when you want the primitive.** It computes the same ladder
and hands back whatever the arithmetic produced, an all-zero value included. It
is for a caller with a reason, and its return type says so: `[u8]`, not
`Result`.

**Call `is_low_order` when you store a peer's static public key.** A server that
registers a client key should find out at registration that the key is
worthless, not at the first session.

**Call `scalar_mult_base` to derive a public key from a secret scalar on its
own**, without building a key pair.

## The rules a user needs

1. **The shared secret is not a key.** It is a `u` coordinate with structure in
   it. RFC 7748 section 6.1 says to run it through a key derivation function.
   [hkdf-nv](https://novo-lang.org/packages/hkdf-nv)'s `extract` over it, with
   the handshake transcript as the salt, is what protocols that get this right
   do.
2. **Clamping happens inside the constructor.** RFC 7748 section 5's
   `decodeScalar25519` clears the low three bits and bit 255 of a secret scalar
   and sets bit 254. `secret_key_from_bytes` does that, so a value of
   `X25519SecretKey` is always ready for the ladder and no caller has to
   remember a step.
3. **`secret_key_bytes` therefore does not return the bytes that went in.** Up
   to five bits are gone. A program that persists a key and later compares it
   with its own entropy will see a difference. Persist what comes out of
   `secret_key_bytes`: that is the key, and the entropy was raw material.
4. **An all-zero shared secret is refused.** RFC 7748 section 6.1 says a
   protocol MAY check for the all-zero value. This package makes the check, and
   `X25519SharedSecretIsZero` is the refusal. An all-zero output means the peer
   sent one of the low-order points, so the shared secret is a constant the peer
   chose and already knows, and everything derived from it is known to the peer
   too. A caller who wants the unchecked value calls `scalar_mult`.
5. **A secret or public key of any length other than 32 bytes is refused**, with
   `X25519SecretKeyLengthWrong(got, want)` or
   `X25519PublicKeyLengthWrong(got, want)`.

Clamping does two things a caller should know about. Clearing the low three bits
makes every scalar a multiple of the cofactor 8, so a point of small order
contributes nothing. Setting bit 254 fixes the leading bit's position, so the
ladder runs the same 255 steps whatever the secret is. Neither is optional.

## Timing behaviour

The Montgomery ladder runs a fixed 255 iterations for every secret scalar,
because clamping fixes the position of the leading bit. That is the property the
scheme is designed around: the time an exchange takes says nothing about the
secret.

The shared secret is compared by the caller, not here. A program that checks one
against a stored value needs a constant-time comparison, and the one on the
registry is `digest.ct_eq` in
[crypto-nv](https://novo-lang.org/packages/crypto-nv).

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device with no
heap allocator, and `tests/embedded_probe.nv` is that claim as a program. Every
function here is arithmetic over bytes the caller already holds, so the claim is
the right one to make.

**The probe does not build today.** Two toolchain defects stand in the way, the
same two that ed25519-nv's probe records:

1. `Result<T, E>` cannot be written in code compiled for a device with no heap,
   because the `Error` trait is not available there.
2. A `[u8]` cannot be *read* there: `list.get` is missing from that surface,
   while `list.len`, `list.is_empty`, `list.with_capacity` and `list.try_push`
   are present.

The signatures keep both `Result` and `[u8]` rather than working around the
defects. The defects are recorded so that they get fixed.

## What is not included

- **A public-key validation function.** Curve25519 is *twist-secure*: a `u`
  coordinate that is not on the curve lies on the quadratic twist, whose order
  is also large enough that nothing about the secret leaks. The invalid-curve
  check that [p256-nv](https://novo-lang.org/packages/p256-nv) must make — P-256
  scalar multiplication against a point off the curve leaks the private key,
  which is CVE-2018-5383 — has no counterpart here. What remains is the
  low-order points, and the all-zero check covers those without a table.
- **A random number generator.** Entropy is a host concern. Draw the 32 bytes
  yourself, from [rand-nv](https://novo-lang.org/packages/rand-nv), from a
  hardware peripheral, or from `/dev/urandom`, and pass them in.
- **A key derivation function.** See rule 1.
- **Ed25519 signatures.** Those are in ed25519-nv.

## Related packages

- [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) supplies the `fe25519`
  module: the prime field GF(2²⁵⁵ − 19) and the scalar clamping. The two schemes
  share the field and nothing above it. This one runs a Montgomery ladder that
  never forms a `y` coordinate; Ed25519 works in extended Edwards coordinates
  and forms both. The module boundary is exactly the field.
- [p256-nv](https://novo-lang.org/packages/p256-nv) is the NIST curve. Use it
  when a peer or a certificate requires P-256; use this one otherwise.
- [hkdf-nv](https://novo-lang.org/packages/hkdf-nv) turns a shared secret into
  keys.

One cost of the dependency is worth naming. Depending on ed25519-nv means
depending on everything ed25519-nv depends on, and that includes crypto-nv for
the SHA-512 that Ed25519 needs and X25519 never uses. Whether it survives
dead-code elimination is a measurement for the implementation release. If it
does not, the answer is a separate package holding the field alone with both
schemes above it.

## Test vectors

RFC 7748 section 5 is the reference, and its pseudocode is the ladder verbatim.
`x25519-dalek` and `libsodium` are the second and third checks.

`tests/x25519_vectors_tests.nv` carries RFC 7748 section 5.2 and section 6.1:
the two raw scalar multiplications, and the whole two-party exchange with both
sides computing the same secret from different halves. Every expected value was
recomputed before it was written down — the exchange against a reference X25519,
the scalar multiplications against the ladder as section 5 writes it.

`tests/x25519_rules_tests.nv` carries the seven low-order encodings that yield
an all-zero secret against section 6.1's own Alice scalar: 0, 1, the two points
of order 8, and the non-canonical spellings `p − 1`, `p` and `p + 1`. Each one
was run through the ladder before it was written down.

## Implementation status

| Item | Implemented |
| --- | --- |
| `x25519.X25519_SECRET_KEY_BYTES`, `.X25519_PUBLIC_KEY_BYTES`, `.X25519_SHARED_SECRET_BYTES` | yes (they are constants) |
| `x25519.X25519SecretKey`, `.X25519PublicKey`, `.X25519KeyPair`, `.X25519Error` | declared |
| `x25519.secret_key_from_bytes`, `.secret_key_bytes` | no |
| `x25519.public_key_from_bytes`, `.public_key_bytes`, `.public_key_of` | no |
| `x25519.key_pair_from_random` | no |
| `x25519.scalar_mult`, `.scalar_mult_base` | no |
| `x25519.diffie_hellman`, `.is_low_order` | no |
| `x25519.X25519Error.message` | no |
| The microcontroller probe | does not build — the two toolchain defects above |

## Licence

Apache-2.0. See `LICENSE`.
