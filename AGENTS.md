# AGENTS.md — PFC horcrux fork

*For the edification of 01node.com — and anyone else running threshold
signers through a Kubernetes operator and learning these lessons the
expensive way.*

## What this repo is

A fork of [strangelove-ventures/horcrux](https://github.com/strangelove-ventures/horcrux)
(see [amygdala-labs/horcrux](https://github.com/amygdala-labs/horcrux) for the
post-rename upstream), kept as a **sync-fork**: `main` tracks upstream releases
verbatim; our changes are dependency sweeps and build fixes only, tagged
`vX.Y.Z-pfc.N`. Never diverge functionally from upstream here — if a behavior
change is needed, it belongs upstream or in a deliberate, documented patch.

## Hard rules (each one paid for)

1. **CGO is mandatory.** Build with `CGO_ENABLED=1`. The cosigners encrypt
   share traffic with ECIES (go-ethereum secp256k1), which panics at sign
   time without cgo: *"ScalarMult is not available when secp256k1 is built
   without cgo"*. A CGO_ENABLED=0 build looks perfect until a 3-member raft
   cluster signs its first block.
2. **Do not bump go-ethereum casually.** It is deliberately held at the
   upstream pin. ECIES is the most sensitive path in the codebase; changes
   there get a dedicated soak, never a sweep rider.
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
