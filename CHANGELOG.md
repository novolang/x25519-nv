# Changelog

All notable changes to x25519-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-12

### Changed — The README is rewritten in plain technical-writer prose; no signature changed.

## 0.0.1 — 2026-09-10

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `x25519` — a key pair from 32 caller-supplied random bytes, the
  Montgomery ladder in both its forms, and the Diffie-Hellman.
- **Clamping happens inside `secret_key_from_bytes`**, so a value of
  `X25519SecretKey` is always ready for the ladder. The visible
  consequence is documented where it shows: `secret_key_bytes` does
  not return the bytes that went in, and the key is what to persist.
- **An all-zero shared secret is refused.** RFC 7748 § 6.1 makes the
  check optional; this package makes it, because an all-zero output is
  a secret the peer chose and already knows. A caller who wants the
  unchecked value has `scalar_mult`, which is the reason it and
  `diffie_hellman` are two functions rather than one with a flag.
- `is_low_order` — the same question asked when a peer's key arrives
  rather than when it is used.
- No `validate`, and the README says why: Curve25519 is twist-secure,
  so the invalid-curve check p256-nv must make has nothing to do here.
- `tests/x25519_vectors_tests.nv` — RFC 7748 § 5.2's two scalar
  multiplications and § 6.1's whole two-party exchange, every value
  recomputed before it was written down.
- `tests/x25519_rules_tests.nv` — the clamping, in three cases, and
  the seven low-order encodings that yield an all-zero secret.
- `tests/embedded_probe.nv` — the device claim as a program.

### Known

- **The embedded probe does not build**, for the two open toolchain
  defects ed25519-nv's probe records: `Result<T, E>` cannot be spelled
  at `@tier(embedded)`, and a `[u8]` cannot be read there because
  `list.get` is not on the embedded surface.
- **The dependency on ed25519-nv carries crypto-nv with it**, for a
  SHA-512 X25519 never calls. Whether dead-code elimination removes it
  is a measurement for the implementation lane; if it does not, the
  answer is a package holding the field alone, which is a grid change.
- The manifest names ed25519-nv by path while the two are developed
  together. A published package depends on published packages by range
  and `novo pkg publish` refuses the path form outright, so this
  becomes `^0.0.1` at publish.
