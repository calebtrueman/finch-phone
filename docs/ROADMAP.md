# Finch Phone roadmap

MVP: one `.ipa` and one `.apk` both launch on the phone from a Finch home screen.

## Stage 1: Darwin personality bring-up (over adb)
- [ ] `finch-run` loads a freestanding arm64e Mach-O, traps its `svc #0x80`, prints, exits
- [ ] Code rewriting: PAC, TPIDRRO_EL0, TPIDR_EL0
- [ ] Commpage
- [ ] dyld + libSystem: a C `hello` from Finch's root prints through `printf`
- [ ] Threads, ulock, workqueue, kevent: libdispatch runs `dispatch_async` and timers
- [ ] objc4 + CoreFoundation + Foundation: an `NSLog` / `NSRunLoop` program

## Stage 2: drawing
- [ ] CoreGraphics renders into a bitmap context and writes a PNG on the phone
- [ ] QuartzCore layers commit into a shared buffer

## Stage 3: Finch Shell + UIKit
- [ ] Android host app: Surface, shared buffer, touch channel, starts `finch-run`
- [ ] Minimal UIKit: `UIApplicationMain`, `UIWindow`, `UIView`, `UIViewController`,
      `UILabel`, `UIButton`, touches
- [ ] `.ipa` install: unzip, Info.plist, launcher entry
- [ ] A test `.ipa` (Finch-built, unencrypted) runs on the phone

## Stage 4: Finch image
- [ ] Finch Shell as the home screen, listing Android and Finch apps together
- [ ] LineageOS-based image for `def` with Finch's root in `/system/finch`; flash and boot
- [ ] MVP: `.ipa` and `.apk` both launch from the home screen
