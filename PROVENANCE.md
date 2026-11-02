# Provenance and licensing

## Owner confirmation

Rodrigo (`rbfcdog`) confirmed that this project was independent personal learning, not university coursework/research or employer/client work, and was not built using those organizations' paid time, equipment, funding, or confidential information. He identifies himself as the sole personal owner of his original contributions. These are owner-provided statements, not an independent legal audit or a claim to exclusive ownership of upstream material.

The original development history is retained. The project is not being re-uploaded to a new repository or presented as newly created for the program.

## Upstream material

`drivers/lkcamp/Kconfig` explicitly attributes the design to the LKCAMP device-driver tutorial. Linux kernel interfaces, the kernel selftest conventions, and tutorial-derived material retain their existing rights and notices. Attribution is not a claim of endorsement by LKCAMP or the Linux project.

- Canonical tutorial reference: [LKCAMP device drivers](https://docs.lkcamp.dev/intro_tutorials/device_drivers/). The historic tutorial path is [unicamp_group/dev_drivers](https://docs.lkcamp.dev/unicamp_group/dev_drivers/).
- Driver, client, selftest, and build/configuration sources specify `GPL-2.0-only`.
- `include/uapi/linux/edu_chat.h` specifies `GPL-2.0 WITH Linux-syscall-note`.
- The root [LICENSE](LICENSE) contains the GPL version 2 text. The UAPI's exception text is in [COPYING.Linux-syscall-note](COPYING.Linux-syscall-note).
- These texts were obtained from the Linux project's [GPL-2.0 license](https://github.com/torvalds/linux/blob/master/LICENSES/preferred/GPL-2.0) and [Linux-syscall-note exception](https://github.com/torvalds/linux/blob/master/LICENSES/exceptions/Linux-syscall-note).

This preparation preserves the existing licensing. It does not relicense tutorial/kernel-derived code as MIT, add an unrestricted dual license, or remove the UAPI exception. Any redistribution must respect the applicable upstream notices and GPL obligations.

The kernel's `MODULE_LICENSE("GPL")` marker declares GPL compatibility to the module loader; the existing per-file SPDX notices and included license text determine the distribution terms. Do not infer permission to relicense from the loader marker.

## Data and execution boundaries

The application exchanges arbitrary bytes between local processes through a shared kernel buffer. Example messages in the C selftest are demonstration strings, not evidence of real user conversations. The repository does not require a customer database, remote inference service, payments, training data, or cloud credentials for its intended driver workflow.

This does not certify the contents of a user's local device buffer or all historical artifacts as free of personal information. Use only synthetic messages in a disposable VM. The world-writable device and shared control flags provide no confidentiality or per-user isolation.

## Program requirements

The supplied CC-CLI rules accept GPL-family projects only with additional approval. Request approval for **GPL-2.0-only**, preserving the UAPI syscall exception, before starting paid sessions. Do not claim the project has been selected or that the license alone establishes eligibility.

The owner confirms an intention to spend at least 20 hours on further robustness and usability work. [ROADMAP.md](ROADMAP.md) records the intended areas rather than completed features or measured hour estimates. Build/runtime evidence and any limitations are recorded in the README.
