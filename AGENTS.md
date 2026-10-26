# AGENTS.md — PFC horcrux fork

*For the edification of 01node.com — and anyone else running threshold
signers through a Kubernetes operator and learning these lessons the
expensive way. With thanks to the Gno team, whose production-hardened
fork became this repo's base (2026-10-26).*

## What this repo is

**Base: [gnolang/horcrux](https://github.com/gnolang/horcrux)** (adopted
2026-10-26; v3.3.2-gno lineage). The Gno team maintains the
production-hardened line — nonce robustness, vote-extension edge cases, the
ECIES fixed-width key encoding fix, the grpc NewClient migration — and this
fork tracks it plus small pin bumps (cometbft 0.38.26, x/net 0.60 as of the
adoption). The original upstream ([strangelove-ventures], renamed
[amygdala-labs]) has been inert since 2025; history predating the adoption
lives on the pre-migration tags. Tag scheme: `vX.Y.Z-pfc.N`.

## Hard rules (each one paid for)

1. **CGO is mandatory.** Build with `CGO_ENABLED=1`. The cosigners encrypt
   share traffic with ECIES (go-ethereum secp256k1), which panics at sign
   time without cgo: *"ScalarMult is not available when secp256k1 is built
   without cgo"*. A CGO_ENABLED=0 build looks perfect until a 3-member raft
   cluster signs its first block.
2. **ECIES changes ride the gnolang base, never a sweep rider.** The
   go-ethereum/ECIES path is the most sensitive in the codebase; its one
   known encoding bug is fixed in the base (fixed-width coordinates +
   malformed-key rejection, with tests). Any further ECIES change gets a
   dedicated soak.
3. **`go.work` must track the toolchain line.** A stale `go 1.21` declaration
   silently breaks every workspace-aware build the moment module
   requirements move. Standalone builds: `GOWORK=off`.
4. **Single-signer canaries prove nothing about raft clusters.** The ECIES
   path only executes between raft members. Test evidence must include a
   multi-member election, and production rolls go singles-first, the 3-of-3
   cluster LAST.
5. **Verify binaries, not vibes.** `grep -c "built without cgo" /bin/horcrux`
   returning 0 proves the panic path is compiled out. Panic-string greps are
   definitive offline verification for compiled-out code paths.

## Testing quirks

- `TestThresholdValidatorLeaderElection2of3` is flaky under `-count>1` on
  macOS — it fails identically on clean upstream pins. Not a regression
  signal; CI on Linux is the gate.
- Full-tree local runs can panic on an atomic-rename under package-parallel
  load; use `-p 2` if reproducing.

## Release flow

fork tag `vX.Y.Z-pfc.N` → bump the clone ref in
[PFC-Validator/horcrux-container](https://github.com/PFC-Validator/horcrux-container)
→ CI builds the image (amd64, alpine, CGO) → verify the binary → roll
signers via Flux, singles first, provenance last.
