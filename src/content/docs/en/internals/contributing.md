---
title: Contributing
description: The verification gates, coding conventions and commit format every change must satisfy.
---

## Verification gates

Every change must pass all of these, in order, with **zero warnings**:

```bash
cargo check --workspace --all-targets
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --all -- --check
./scripts/test-linux-mock.sh                 # dbus-run-session + python3-dbusmock
cargo test -p uda-ffi                        # C-ABI regression
cargo check -p uda-platform-windows --target x86_64-pc-windows-gnu --all-targets
```

The last one matters more than it looks: `crates/uda-platform-windows/src/lib.rs` is `#![cfg(windows)]`, so its code compiles to **nothing** on a Linux host. Without the cross-compile check, a Windows-only compile error reaches contributors invisibly.

## The three principles

1. **Capability-driven architecture (never panic).** No `.unwrap()` or `.expect()` on a system call, a D-Bus invocation or an environment variable. Every feature exposes a `SupportLevel` check — `Full`, `Partial(reason)` or `None` — and degrades gracefully before returning an error.
2. **The cascading fallback engine.** Tier 1 XDG portal → Tier 2 native DE IPC → Tier 3 CLI tools → Tier 4 a typed `UdaError::NotSupported`.
3. **Zero-bloat.** No Qt, no GTK, no bundled toolkit. Pure-Rust `zbus` on Linux, `windows-rs` on Windows.

## Conventions

| Concern | Rule |
|---|---|
| Errors | `thiserror` for library-internal errors; every error carries a diagnosis |
| Async | `tokio` where async is required; synchronous wrappers where practical |
| Logging | `log::debug!` / `log::warn!`, never `println!` in a library crate |
| Locks | Recover from poisoning rather than propagating it |
| Tests | One behaviour per test, named as a sentence describing the contract |

## Test placement

Anything that must be verifiable on the CI host belongs in **`uda-core`**, not in a platform backend. The platform crates keep their tests for the parts that genuinely need the OS, run under the D-Bus mock harness.

The Win2 Linux symmetry is deliberate: the pure logic in `uda-core` is what lets a Linux runner test the Windows contract.

## Commit format

Conventional Commits, `<type>(<scope>): <short description>`:

```
feat(tray): add checkbox rows to the cross-platform menu model
fix(notification): normalise a Windows icon path into a file:// URI
refactor(core): replace the absolute-path helper with a host-independent check
docs: rewrite the README as a project card pointing at the documentation site
chore: drop background narration from the crate-level comments
```

## Documentation set

A change touches, as applicable:

| File | When |
|---|---|
| `CHANGELOG.md` | every user-visible change, in the bilingual line-by-line format |
| `README.md` / `README_CN.md` | a new high-level capability, or a shift in the architecture |
| `AGENTS.md` | a phase boundary, or a change to the engineering rules |
| `docs/internals/*.md` | a new protocol, or a correction to an existing mapping |
| `plans/<phase>_plan.md` | at the start of a phase; updated as scope lands |
| `wiki/` | a new guide, reference or protocol page — and the English page when the Chinese one lands |

## Adding a feature

1. Write the spec first: `docs/internals/<topic>_specs.md`, with the protocol, the capability matrix and the degradation chain.
2. Put the shared decision logic in `uda-core`, unit-tested.
3. Implement the two backends against that core.
4. Add the C-ABI surface if a non-Rust host needs it, with the ownership rules documented.
5. Wire the SDKs and one example per capability.
6. Run the full gate set.
7. Update the documentation set above.

## See also

- [Architecture](/en/internals/architecture/) — the layering and what belongs where
- [Fallback engine](/en/internals/fallback-engine/) — the tier chain in detail
