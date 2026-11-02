# CC-CLI application answers

Copy the answer text under each field. GPL-family licensing requires extra program approval before paid sessions. This document is an application draft, not an approval or legal certification.

## Your GitHub profile URL

https://github.com/rbfcdog

## Project name

edu-chat-device-driver

## Repository link

https://github.com/rbfcdog/edu-chat-device-driver

## Short description of the project

An educational Linux character-device driver that lets local processes exchange demonstration messages through `/dev/edu_chat`. Written in C, it includes a bounded kernel buffer, configurable behavior through ioctl and sysfs, blocking reads, poll notifications, a userspace chat client, and a selftest. I built it for independent personal learning and intend to continue improving its robustness, boundary handling, and usability.

## Which license is the project under?

GPL v2. The project sources specify GPL-2.0-only, with GPL-2.0 WITH Linux-syscall-note for the UAPI header. The root LICENSE contains GPL version 2, and COPYING.Linux-syscall-note preserves the UAPI exception. I understand this requires additional program approval.

## What is your relationship to this project?

I created it and I am the only owner.

This describes ownership of my original contributions. The acknowledged LKCAMP tutorial and Linux-related upstream material retain their existing rights and GPL obligations. I am not claiming sole ownership of third-party code.

## Is the project owned by a company, partnership or other entity, including your own startup?

No.

## Do other people actively work on this repository with you?

No.

## Would you take part in separate sessions and review another participant's work?

Yes.

## Anything else we should know about this project?

This is independent personal learning work, not a university assignment, employer project, or client deliverable. It was not built using their paid time, equipment, funding, or confidential information. I personally own my original contributions, while preserving the existing GPL rights and attribution for tutorial and kernel-related material.

The project demonstrates local kernel/userspace communication using synthetic messages. It does not require a customer database, private business records, financial or health data, AI-model training, cloud credentials, or a remote AI service. The device accepts arbitrary local bytes, so demonstrations will stay in an isolated disposable VM with synthetic messages rather than real conversations. The shared device is not designed to provide confidentiality or per-user isolation.

I genuinely plan at least 20 hours of additional robustness and usability work. The public ROADMAP.md covers concurrent readers and writers, clear/shrink/cursor semantics, buffer boundaries and error-path regression coverage, and userspace-client input/event handling. These are planned improvements, not claims that the project is production-ready or every behavior is already verified.

The existing development history is preserved. I would like the required GPL v2 approval before beginning paid sessions.

## Extra-approval message

I would like to submit https://github.com/rbfcdog/edu-chat-device-driver. It is my independent personal Linux character-device learning project, with no active human collaborators and no company ownership. The driver/client/selftest sources specify GPL-2.0-only, and the UAPI header uses GPL-2.0 WITH Linux-syscall-note. The root license and exception texts are included. May I receive the additional GPL-family approval required by the program? I will wait for approval and onboarding before starting paid sessions.

## Before certification

- Obtain the program's GPL-family approval and selection before beginning paid work.
- Confirm tutorial/upstream attribution and redistribution obligations are satisfied. Existing source identifiers were preserved, not relicensed as MIT.
- Use a disposable VM with matching kernel headers/image. Do not load this educational module on a shared or production host.
- Use synthetic messages only. Do not store real conversations, credentials, or VM logs in Git.
- Treat scan/build/runtime observations in README as bounded evidence, not proof of universal security or eligibility.
- Keep the real commit history and use a GitHub-linked author email for future work.
