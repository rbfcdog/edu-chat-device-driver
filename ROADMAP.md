# Further work: robustness and usability

The owner genuinely intends to spend **at least 20 additional hours** improving this personal learning project, especially concurrent access, shared-buffer/event semantics, regression coverage and the C client. This is a direction for real further work, not a claim that the work is complete, an allocation of hours to individual items, or a promise that every item fits into that time.

The current design remains one fixed, bounded shared byte buffer. It is not a durable messaging service or a private multi-user chat. The priorities below strengthen that learning exercise rather than add unrelated kernel features.

## 1. Define concurrent access and cursor/event semantics

**Current limit:** operations use a shared mutex, but each open descriptor has its own read position. Clear and limit-shrink change shared length without resetting every reader. Clear-on-read clears globally when a reader reaches the end. `poll()` and blocking reads depend on that combination of cursor, length and global flags. Having a lock does not by itself establish the intended behavior across those transitions.

**Work:** specify what existing and newly opened readers should see when writers append or replace, another reader consumes the buffer, the buffer is cleared, the limit shrinks, or flags change. Decide whether readers beyond the new end should wait, reset, or receive another documented outcome, then make event delivery consistent with that decision.

**Acceptance:** deterministic two-reader/two-writer scenarios assert the chosen byte visibility, cursor recovery and readiness behavior. Waiting readers can be interrupted; clear/shrink/flag transitions do not leave them stuck contrary to the documented contract. Bounded stress runs check for hangs and kernel diagnostics. No concurrency correctness is claimed before those checks exist and pass.

## 2. Specify failed-write and error semantics

**Current limit:** oversized writes return `EMSGSIZE`, and appends without enough remaining space return `ENOSPC`. The overwrite path clears contents before `copy_from_user()`, so failure-preserves-old-data is not an existing guarantee. A partial user-memory copy, zero-length I/O and interrupted operations need explicit contracts. The `failed_writes` counter currently covers size/space rejections, not every error path.

**Work:** decide the intended buffer state after failed copies and zero-length writes, document retry/atomicity expectations, and align statistics with their defined meaning. Review read-copy failures and readiness versus errors as part of the same contract, without assuming `POLLOUT` means any requested write will fit.

**Acceptance:** targeted invalid-pointer, partial-copy, zero-length and interrupted-I/O cases assert return values, contents, length, cursor and counter effects. Oversize and full-buffer failures are distinguished. Tests report the actual behavior rather than silently accepting data loss or relying only on printed diagnostics.

## 3. Add boundary and transition regression coverage

**Current limit:** the existing selftest mostly prints content, info, readiness and statistics; its explicit empty-read `EAGAIN` check and syscall/write checks are not a comprehensive regression suite.

**Work:** add small, runnable assertions for exact-fit writes, one-byte overflow, append exhaustion, partial reads, empty buffers, minimum/maximum capacity and logical limits, invalid flags/ioctls, clear-on-read, shrink/grow, and descriptor reopen behavior. Cover both sysfs and ioctl configuration where they should agree. Reuse the semantic decisions above rather than freeze accidental behavior into tests.

**Acceptance:** each case fails with a nonzero status and useful context when bytes, return codes, length, flags, positions or readiness differ from the contract. Blocking cases have deadlines and clean up their own processes. A documented disposable-VM command runs the suite; a successful observational selftest alone is not presented as proof.

## 4. Make the terminal client resilient

**Current limit:** the client enables global append-only mode, skips old bytes on entry and exits on write failure. It does not queue/retry under backpressure or explicitly handle all poll hangup/error events. `snprintf()` returns the length that would have been produced, but that value is passed directly to `write()` without first checking it against the fixed output buffer size. Long nicknames or input therefore require more than a cosmetic UI improvement.

**Work:** bound formatting before writes; define long-line behavior; handle EOF, `POLLHUP`, `POLLERR`, `EINTR` and `EAGAIN` without spinning or losing pending input. Define what the client should do when append storage is full and when the shared buffer is cleared underneath it. Make the fact that its configuration affects other users visible.

**Acceptance:** short interactive exchanges still work, while long nicknames/lines cannot cause an out-of-bounds read or silent truncation. Piped input and EOF terminate cleanly. Full-buffer and transient-readiness scenarios have a documented, exercised outcome without a busy loop, silent message loss, or an unbounded pending queue. Recovery follows the shared-cursor contract from item 1.

## 5. Make builds and VM checks reproducible

**Current limit:** this repository has separate standalone and in-tree paths; the integration patch adds hooks but does not copy sources. Kernel/API/header compatibility and module-loading privileges remain environment requirements.

**Work:** add minimal build/test automation and packaging instructions that keep artifacts separate, check the selected kernel build tree, distinguish integration from external-module builds, and run destructive/global-state tests only in a disposable guest. Record exact kernel/toolchain versions alongside real results rather than imply broad compatibility.

**Acceptance:** a clean checkout can build the module and both C programs using documented commands. The automated regression entry point refuses unintended host module loading, uses bounded guest tests, cleans up only resources it owns, and records failures honestly. In-tree instructions do not duplicate directories or integration hooks. No host-wide process killing or blind module unloading is introduced.

## Scope and evidence

These items are planned, not observed test failures or completed fixes. Some are visible code-path limitations; others are behavioral questions that need experiments and a documented decision. Future verification should state the exact tested environment and cases, not extrapolate to all kernels or concurrent schedules.

The current `0666` shared device has no confidentiality boundary; robustness work does not turn it into secure production messaging. Any change to that access model needs its own explicit design and scope. Licensing and provenance remain as described in [README.md](README.md#license-and-provenance) and [PROVENANCE.md](PROVENANCE.md); organizer approval for GPL-family licensing is still required before paid sessions.
