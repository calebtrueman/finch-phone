# Finch Phone design

## Goal

One phone runs both:

- **Android apps** (`.apk`), on the open Android base's own runtime (ART), unchanged.
- **iOS apps** (`.ipa`) and Finch programs, as Darwin Mach-O processes, on Finch's own
  open userland (dyld, libSystem, libdispatch, objc4, CoreFoundation, Foundation,
  CoreGraphics, QuartzCore, plus a Finch UIKit).

Both appear as apps on one home screen.

## Why a Darwin personality on Linux, not XNU

XNU has no drivers for a Qualcomm phone: no display, touch, modem, GPU, power or storage
drivers, and none exist anywhere in open form for XNU. The phone's Linux kernel has all of
them. Finch's compatibility promise is at the **userland ABI** (Mach-O binaries calling
libSystem), so Finch Phone keeps Linux below and provides the Darwin kernel interface on
top. The same runtime works on any arm64 Linux device, which makes Finch hardware-independent.

Darling takes a similar approach, but it is GPL-3.0 and uses a Linux kernel module.
Finch Phone shares no code with it and needs no kernel module (see "Syscall capture").

## `finch-run`: the Darwin personality

`finch-run` is a **freestanding** static Linux/arm64 ELF: no bionic, no libc. That matters
because Darwin code owns the thread-pointer registers and x18 in its threads, and the
runtime must be safe to enter from any Darwin instruction.

```
finch-run [--root DIR] PROGRAM [ARGS...]
```

It plays the part of XNU's `exec`:

1. Map the commpage (`0xFFFFFC000`) and the read-only commpage (`0xFFFFF4000`) with
   the CPU count, 16 KB user page shift, memory size and timebase.
2. Load the program's Mach-O and Finch's `/usr/lib/dyld` from the root, copying segments
   into anonymous memory. (Android forbids executable file mappings from app data, and
   code must be rewritten anyway.)
3. Rewrite code for the host CPU (see "Code rewriting").
4. Install the syscall trap (see "Syscall capture").
5. Build the Darwin entry stack `[mach_header*, argc, argv…, 0, envp…, 0, apple…, 0]`
   and jump to dyld's entry point. From there dyld, libSystem and the app run as on Darwin.

### Syscall capture

Darwin code enters the kernel with `svc #0x80`. x16 holds the number: positive for BSD
syscalls, negative for Mach traps, and `0x80000000` for platform calls. Errors set the
carry flag. Linux ignores the `svc` immediate and reads the number from x8, so these
calls must be caught before Linux runs them.

`finch-run` installs a **seccomp-BPF filter keyed on the instruction pointer**. A syscall
whose PC lies inside `finch-run`'s own text is allowed; any other syscall gets
`SECCOMP_RET_TRAP`, which delivers `SIGSYS` without running it. The SIGSYS handler,
which is runtime code and therefore allowed to make syscalls, decodes x16 from the
signal context, emulates the call, writes x0/x1 and the carry flag back, and returns.
This works on the device's 4.14 kernel. Syscall User Dispatch would need 5.11. Threads
inherit the filter.

### Code rewriting

Finch builds its userland as **arm64e**, and on Apple CPUs the kernel owns two registers
that Linux doesn't. On every executable mapping, `finch-run` rewrites:

| Darwin instruction | Problem on Linux / ARMv8.2 | Rewrite |
|---|---|---|
| `mrs xN, TPIDRRO_EL0` (thread TSD base) | EL0 can't set TPIDRRO_EL0 on Linux | `mrs xN, TPIDR_EL0`, which the runtime sets to the TSD base |
| `mrs xN, TPIDR_EL0` (CPU number and cluster) | now holds the TSD base | `movz xN, #0` (CPU 0) |
| `pacia`/`pacda`/`autia`/`autda`/… (non-hint forms), `xpaci`/`xpacd` | undefined on v8.2 | `nop`. Pointers stay unsigned, which is consistent throughout. |
| `pacga xD, xN, xM` | undefined | `movz xD, #0` |
| `braa`/`brab`/`blraa`/`blrab` and their `z` forms, `retaa`/`retab` | undefined | `br`/`blr`/`ret` |
| `ldraa`/`ldrab` (no writeback, offset ≥ 0) | undefined | `ldr` |

Hint-space PAC instructions (`paciasp`, `autibsp`, `xpaclri`, …) are already no-ops on
CPUs without PAC. A `SIGILL` handler emulates anything left over, such as `ldraa` with
writeback.

### Kernel surface to emulate (in order of need)

- **BSD**: file I/O with Darwin `stat`/`dirent` layouts; `mmap` with Darwin flags, 16 KB
  alignment and executable mappings copied and rewritten; `getentropy`, `sysctl`/
  `sysctlbyname` (`hw.*`, `kern.*`), `proc_info`, `csops`, `issetugid`, `getpid`, `exit`, …
- **Mach traps**: `mach_reply_port`, `task_self_trap`, `thread_self_trap`,
  `host_self_trap`, `mach_msg2_trap`, `_kernelrpc_mach_vm_*`, `_kernelrpc_mach_port_*`,
  `mach_timebase_info`, `semaphore_*`, `thread_switch`, `swtch_pri`.
- **Mach IPC**: an in-process port namespace. Messages to kernel objects (task, host,
  thread, semaphore) are answered by MIG servers in `finch-run`. Port-to-port messages
  between Darwin threads queue in the runtime. Cross-process IPC comes later, through a
  broker.
- **pthread / workqueue**: `bsdthread_register`, `bsdthread_create`,
  `workq_open`/`workq_kernreturn` (libdispatch's thread pool), `__ulock_wait`/`__ulock_wake`
  (futex), `thread_set_tsd_base`.
- **kevent**: `kevent_qos`/`kevent_id` on epoll, timerfd and eventfd.
- **Signals**: Darwin `sigaction` with its `_sigtramp`, and Linux signal frames translated to
  Darwin `ucontext`/`mcontext`.

## Display and input (iOS apps)

An iOS app links UIKit, which Finch doesn't have yet. Finch Phone adds a **minimal
UIKit** in the same style as Finch's AppKit: an ABI-compatible implementation of the
symbols that apps import, drawing through Finch's CoreGraphics/QuartzCore.

On the Android side, a small host app (**Finch Shell**) owns an Android `Surface` for each
Darwin app. It starts `finch-run` as a child process, from the APK's native library
directory so that exec is allowed, and shares a pixel buffer and an input channel with it.
The Darwin side presents frames into the buffer; touches come back as UIKit events. The
Darwin side never runs inside an ART process.

## `.ipa` install

An `.ipa` is a zip holding `Payload/Name.app`. Finch Shell unpacks it into Finch's
`/Applications`, reads `Info.plist` (name, icon, executable) and adds a launcher entry.
App Store `.ipa` files are FairPlay-encrypted, so the targets are unencrypted builds:
self-built, open-source, or decrypted by their owner.

## The phone image

Phase 1 runs as an app on the existing ROM (crDroid 13) and is developed over `adb`. The
MVP's final step is a Finch-branded image: an open Android base (LineageOS for `def`)
with Finch Shell as the home screen and Finch's root in `/system/finch`.

## Rules carried over from Finch

- No Apple binaries are committed. The system image is Finch's own built root.
- No GPL code is copied into the runtime (Darling, Linux). Interface facts from headers,
  documentation and observed behaviour are fine.
- Every source file carries `SPDX-License-Identifier: MIT OR Apache-2.0`.
