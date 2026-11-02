# edu_chat: an educational Linux character device

`edu_chat` is a personal learning project for developers exploring Linux character-device drivers and the user/kernel boundary. It exposes `/dev/edu_chat`: local processes write bytes into a shared kernel buffer and read them back through ordinary file operations. A small C terminal client presents those bytes as a chat demonstration.

**This is a bounded shared-buffer experiment, not production messaging.** It has no accounts, private rooms, durable storage, message framing, or delivery guarantees. It is intended for disposable Linux virtual machines, not a host or a multi-user service.

- [Build and run in a disposable VM](#standalone-build-and-run-in-a-disposable-vm)
- [Optional in-tree kernel integration](#optional-in-tree-kernel-integration)
- [Further work and acceptance goals](ROADMAP.md)
- [Licensing](#license-and-provenance), [provenance](PROVENANCE.md), and [CC-CLI submission context](CC_CLI_PROJECT_SUBMISSION.md)

## Current behavior

The implementation includes:

| Interface | Current behavior |
| --- | --- |
| Character device | Dynamically registered `cdev`, user/kernel copies, and per-open read positions. |
| Buffer | One allocation at module load: default 4,096 bytes, maximum 65,536 bytes. A separate logical limit can be changed without reallocating. |
| Writes | Replace the shared contents by default; global append mode accumulates bytes until the logical limit is reached. This is not a ring buffer. |
| Reads | Read from the descriptor's position. At the current end, return zero by default; with the global blocking-read flag, wait, or return `EAGAIN` for an `O_NONBLOCK` descriptor. |
| Clear-on-read | Optionally clear the entire shared buffer when a read reaches its end; this is not a separate queue for each reader. |
| Events | A wait queue supports blocking reads and `poll()`/`select()` readiness. Write readiness reports some available space, not that an arbitrary message will fit. |
| Control | `ioctl()` commands get information/statistics, clear contents, and set the global limit/flags. Sysfs exposes `limit`, `flags`, `length`, `stats`, and `clear`. |
| Programs | An observational selftest and an interactive C client using `poll()`. The client's prompts are currently in Portuguese. |

Flags start at zero and affect **all** open descriptors: append is `0x1`, clear-on-read is `0x2`, and blocking-read is `0x4`. The logical limit starts at the allocated capacity. A write larger than the limit returns `EMSGSIZE`; an append that exceeds remaining space returns `ENOSPC`. Appending does not reset the writer's read position. Clearing or shrinking the shared buffer does not reset every other descriptor's position.

The shared mutex protects buffer operations and metadata, but that is not proof of correct multi-process event or cursor semantics. Concurrent readers/writers, clear/shrink transitions, failed user-memory copies, and cursor recovery remain areas for investigation and regression coverage; see [ROADMAP.md](ROADMAP.md).

### Security and data handling

The driver requests device-node mode **`0666`**. Unless the guest's device-management policy overrides it, any local user can read or overwrite the shared bytes, clear them, and change global behavior through `ioctl()`; these operations do not require administrative privileges. Sysfs configuration normally requires root, but it is not an authorization boundary for the equivalent ioctls.

**There is no confidentiality or isolation between local users.** Use only synthetic, non-sensitive input in a disposable VM with no untrusted users. Do not type credentials or personal conversations. The device accepts arbitrary local bytes; absence of committed chat logs would not certify that real data has never been used. Kernel code can compromise or crash its guest, and clearing this buffer is not a secure-erasure guarantee.

## Source map

| File | Responsibility |
| --- | --- |
| [`drivers/lkcamp/edu_chat_core.c`](drivers/lkcamp/edu_chat_core.c) | Allocation, registration, `0666` node policy, initialization and teardown. |
| [`drivers/lkcamp/edu_chat_fops.c`](drivers/lkcamp/edu_chat_fops.c) | Read/write, per-open positions, readiness and ioctls. |
| [`drivers/lkcamp/edu_chat_sysfs.c`](drivers/lkcamp/edu_chat_sysfs.c) | Sysfs controls and statistics. |
| [`drivers/lkcamp/edu_chat_internal.h`](drivers/lkcamp/edu_chat_internal.h) | Internal shared declarations. |
| [`include/uapi/linux/edu_chat.h`](include/uapi/linux/edu_chat.h) | Public flags, limits, ioctl numbers and payload layouts. |
| [`tools/testing/selftests/edu_chat/`](tools/testing/selftests/edu_chat/) | C selftest, terminal client and userspace Makefile. |
| [`kernel-integration.patch`](kernel-integration.patch) | Hooks for an existing Linux source tree; not a complete source patch. |

## Standalone build and run in a disposable VM

### Prerequisites

Create a disposable Linux VM/snapshot first. **Run every command below inside that guest**, including module loading. Do not load this module into your workstation's kernel. Avoid shared folders containing sensitive host data.

The guest needs:

- Git, GNU Make, a C compiler and libc development headers for the programs.
- A kernel-module build toolchain and matching, prepared headers/build tree for the **guest's running kernel**, normally at `/lib/modules/$(uname -r)/build`. Distribution packages may also require tools such as binutils or libelf development files.
- Kernel module loading support, `kmod` (`insmod`, `modinfo`), and administrator access within the guest. Guest Secure Boot/module-signing policy must permit the module; do not disable host protections for this demo.
- Device management such as devtmpfs/udev to create `/dev/edu_chat`, mounted sysfs, and `timeout` from GNU coreutils for the bounded selftest invocation below.

Kernel APIs vary. This source uses the one-argument `class_create()` API; compatibility with every kernel release is not promised. Use headers compatible with the source and matching the guest kernel, or use the in-tree workflow to build and boot a suitable guest kernel.

### Clone and build

```sh
git clone https://github.com/rbfcdog/edu-chat-device-driver.git
cd edu-chat-device-driver

make -C /lib/modules/"$(uname -r)"/build \
  M="$PWD/drivers/lkcamp" \
  CONFIG_EDU_CHAT=m \
  KCFLAGS="-I$PWD/include" \
  modules

make -C tools/testing/selftests/edu_chat
```

The module Makefile uses `CONFIG_EDU_CHAT` to select the composite module, so the external build must pass `CONFIG_EDU_CHAT=m`. The include flag supplies the repository's `<uapi/linux/edu_chat.h>` to the kernel build. The userspace Makefile supplies `include/uapi` itself and, outside a full kernel tree, builds both `edu_chat_test` and `edu_chat_client` with its standalone fallback. This workflow does **not** need `kernel-integration.patch` or installation into system directories.

Expected output paths are `drivers/lkcamp/edu_chat.ko` and the two programs under `tools/testing/selftests/edu_chat/`. These are build targets, not a claim that a particular environment has been verified.

### Load and inspect — guest only

Start from a fresh VM boot with no existing `edu_chat` instance. Inspect the guest identity and module metadata before loading:

```sh
uname -r
modinfo drivers/lkcamp/edu_chat.ko
sudo insmod drivers/lkcamp/edu_chat.ko capacity=4096
ls -l /dev/edu_chat
cat /sys/class/edu_chat/edu_chat/limit
cat /sys/class/edu_chat/edu_chat/flags
```

The initial limit should be `4096` and flags `0x0` for this load. If the module is already loaded, do not blindly remove it: use a fresh guest boot or explicitly identify its owner and open handles. `Invalid module format` usually calls for checking the guest kernel/build-tree match and module metadata; inspect the guest kernel log for the actual reason. A missing node calls for checking the guest's device-management setup and module initialization log, not guessing a major number.

### Minimal byte-buffer demonstration

With the freshly loaded default flags and no other clients:

```sh
printf 'alice: hello from the guest\n' > /dev/edu_chat
cat /dev/edu_chat
cat /sys/class/edu_chat/edu_chat/length
cat /sys/class/edu_chat/edu_chat/stats
```

In default overwrite mode, a new open reads the stored bytes and then reaches the current end. Reading does not normally delete them. If another process enables blocking-read mode, `cat` can instead wait at the end; stop that specific foreground command with Ctrl+C.

### Existing selftest

Close the clients first: this program changes global flags/limit and clears all contents. Run it only against this disposable guest instance:

```sh
timeout 10s ./tools/testing/selftests/edu_chat/edu_chat_test /dev/edu_chat
```

The selftest exercises writes, reads, ioctls, `poll()`, clear-on-read, limit changes and statistics. It checks syscall failures and full writes, and explicitly checks for `EAGAIN` on an empty nonblocking read with blocking-read mode enabled. Most content, readiness and statistics checks **print observations rather than assert expected values**. A zero exit status is not comprehensive regression or concurrency evidence. A timeout is a failure to investigate, not a successful test. See the bounded execution evidence below.

### Two-terminal client demonstration

After the selftest and before opening either client, reset the demo's shared state:

```sh
printf '4096\n' | sudo tee /sys/class/edu_chat/edu_chat/limit >/dev/null
printf '0x1\n' | sudo tee /sys/class/edu_chat/edu_chat/flags >/dev/null
printf 'clear\n' | sudo tee /sys/class/edu_chat/edu_chat/clear >/dev/null
```

In two terminals **inside the same guest**, from the repository root:

```sh
# Terminal A
./tools/testing/selftests/edu_chat/edu_chat_client /dev/edu_chat alice
```

```sh
# Terminal B
./tools/testing/selftests/edu_chat/edu_chat_client /dev/edu_chat bob
```

Each client opens `O_RDWR | O_NONBLOCK`, sets the global flags to append-only, drains existing bytes from its own cursor without displaying them, and polls the keyboard and device. Open both clients before sending short synthetic lines; use short nicknames too. Exit each foreground client with Ctrl+C.

This is not unlimited chat history: reads in append-only mode do not free space, and the client exits on a failed write when the buffer fills. It does not implement queued/retried writes or robust formatting-length handling. Clearing or shrinking underneath active readers needs further cursor/event work; stop both clients before resetting the demonstration. EOF/hangup handling and backpressure are also backlog items, not verified features.

### End the session safely

Close each foreground client or reader explicitly. Do not kill processes by broad name/pattern and do not force module removal. The simplest cleanup for this workflow is to shut down and discard/revert the disposable VM. If investigating a module-in-use error, identify the particular processes holding `/dev/edu_chat` and close only those you started; a fresh guest avoids relying on blind unload/reload commands.

## Optional in-tree kernel integration

This is a **different workflow** for someone already comfortable building and booting a Linux kernel in a VM. It requires a separate, compatible, configured Linux source tree and its normal kernel-build dependencies. Run the following from that tree's root, not from this repository:

```sh
REPO=/absolute/path/to/edu-chat-device-driver

# Check the hook patch against this kernel revision before copying files.
git apply --check "$REPO/kernel-integration.patch"

# Use a clean tree with neither destination directory already present.
test ! -e drivers/lkcamp && test ! -e tools/testing/selftests/edu_chat && \
  test ! -e include/uapi/linux/edu_chat.h && \
  cp -R "$REPO/drivers/lkcamp" drivers/ && \
  cp "$REPO/include/uapi/linux/edu_chat.h" include/uapi/linux/ && \
  cp -R "$REPO/tools/testing/selftests/edu_chat" tools/testing/selftests/ && \
  git apply "$REPO/kernel-integration.patch"
```

Stop on a failed check or copy. The patch only adds hooks to `drivers/Kconfig`, `drivers/Makefile`, and `tools/testing/selftests/Makefile`; it does **not** contain the copied driver, UAPI or program files. Its context is kernel-revision-specific. Do not apply it twice, add duplicate hooks manually, or repeat `cp -R` into existing destination directories: that can create nested copies such as `drivers/lkcamp/lkcamp`. Inspect and reconcile an existing integration instead.

With the copy and hooks successfully integrated into an already configured kernel tree:

```sh
./scripts/config -m EDU_CHAT
make olddefconfig
make -j"$(nproc)" drivers/lkcamp/edu_chat.ko
make -C tools/testing/selftests/edu_chat
```

Inside a full kernel tree, the program Makefile uses `tools/testing/selftests/lib.mk` rather than its standalone fallback; follow that kernel tree's exported-header preparation requirements. Building a module for a new kernel does not make it loadable into the guest's old running kernel. Build/install and boot the corresponding kernel **in the disposable VM** using that tree's normal procedure before following the guest-only runtime steps above. Neither workflow here installs or modifies the host kernel.

## License and provenance

The driver, C programs and build files retain their **GPL-2.0-only** notices; see [LICENSE](LICENSE). The exported header retains its distinct SPDX identifier, **`GPL-2.0 WITH Linux-syscall-note`**; see [COPYING.Linux-syscall-note](COPYING.Linux-syscall-note). That UAPI exception is not a blanket relicensing of the driver or client.

See [PROVENANCE.md](PROVENANCE.md) for tutorial/upstream origins and contribution boundaries, and [CC_CLI_PROJECT_SUBMISSION.md](CC_CLI_PROJECT_SUBMISSION.md) for the personal-learning submission context. GPL-family licensing requires organizer approval **before paid CC-CLI sessions**; inclusion of these documents does not constitute approval.

## Verification for this submission preparation

The standalone commands above built `edu_chat.ko`, `edu_chat_test`, and
`edu_chat_client` against the `7.0.0-38-generic` prepared headers using GCC 13
for the module and the host C compiler for the programs. The build warned about
the compiler executable's name, unavailable `pahole`, and skipped BTF generation
because no `vmlinux` was supplied; the module still linked successfully.

That exact module and both programs were exercised in a disposable QEMU/KVM
guest booting the matching kernel image, with a minimal initramfs and networking
disabled. No host module was loaded. The guest observed:

- an exact synthetic byte-message roundtrip through `/dev/edu_chat`;
- successful completion of the existing selftest, including its `EAGAIN` check;
- a blocking reader receiving a synthetic message written by another process;
- the actual terminal client formatting and displaying a short synthetic message;
- successful module unload and removal of its sysfs class.

This is a smoke run, not concurrency stress, long-input safety, broad kernel
compatibility, in-tree patch verification, or proof of all roadmap behavior.
The full two-terminal interactive demonstration was not automated in this run.

Gitleaks 8.30.1, checked against its published archive SHA-256, reported zero
credential matches in the existing Git history (21 commits scanned by the tool;
22 commits in the repository). A source/history review found demonstration
messages, not customer records or conversation logs. This is scoped evidence,
not a guarantee that no secret or real data could ever be introduced. Author
metadata and a historic local build path are not represented as credentials.

## Changes in this submission preparation

Added canonical GPLv2 and UAPI-exception texts, source/provenance notes, the genuine robustness/usability backlog, a build-artifact ignore file, and application answers. Replaced the Portuguese overview with English reviewer-facing build, VM-safety, interface, security and test-scope documentation. No driver/client behavior or source licensing identifiers were changed; planned improvements remain future work.
