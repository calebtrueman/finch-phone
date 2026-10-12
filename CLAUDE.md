# Finch Phone: notes for Claude

Finch Phone runs Finch's Darwin userland on Android phone hardware through a Darwin
personality on Linux (`runtime/`), next to native Android apps. Read `docs/DESIGN.md` first.

Hard rules:
- Never commit Apple binaries or Finch's built root. The system image comes from
  `../finch/build/root` (the Finch repo) at build/push time.
- Never copy GPL code into the runtime (Darling, the Linux kernel, GPL-only Asahi files).
  Every source file carries `SPDX-License-Identifier: MIT OR Apache-2.0`.
- `finch-run` is freestanding: no libc calls, no TLS. Darwin code owns TPIDR_EL0 and x18.
- Device: Motorola One Hyper (`def`), unlocked, crDroid 13 / Linux 4.14. adb and fastboot
  are at `/opt/homebrew/share/android-commandlinetools/platform-tools`. Don't flash
  or wipe without asking the user.
- Work that changes Finch itself (for example a Finch UIKit framework) goes to the Finch
  repo through its own process, not into this one.
