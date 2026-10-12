# Finch Phone

A mobile [Finch](https://github.com/calebtrueman/finch) that runs iOS apps (`.ipa`) and
Android apps (`.apk`) side by side on ordinary Android phone hardware.

Finch's userland (dyld, libSystem, Foundation, CoreGraphics, …) is built for Darwin. Finch
Phone runs that userland **unmodified** on a Linux kernel through the *Darwin
personality*: a Mach-O loader plus a translation layer for Darwin system calls and Mach
traps. The phone keeps the Linux kernel and drivers of an open Android base, which also
runs Android apps natively. Finch therefore doesn't depend on Apple hardware: it runs on
any kernel that has drivers for the device.

Status: early bring-up. See [docs/DESIGN.md](docs/DESIGN.md) and
[docs/ROADMAP.md](docs/ROADMAP.md).

## Target device

Motorola One Hyper (`def`, Snapdragon 675 / sm6150, 4 GB RAM, 1080×2340), with an unlocked
bootloader, currently running crDroid 13 (LineageOS 20 based) on Linux 4.14.

## Layout

| Path | What |
|---|---|
| `runtime/` | The Darwin personality: `finch-run`, a freestanding Linux/arm64 program that loads and runs Darwin Mach-O binaries |
| `tools/` | Host scripts: build, push to the phone, run tests |
| `docs/` | Design and roadmap |

Finch Phone consumes Finch's built root (`finch/build/root`) as its system image. Nothing
from that image, and no Apple binary, is committed here.

## License

MIT OR Apache-2.0, as for Finch.
