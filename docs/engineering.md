# CYDER iFONE — iPhone 11 boot diagnostics

**Test setup** · iPhone 11 A2111 (N104AP, A13/T8030), iOS 26.5, Linux Mint 22.3, Waveshare RP2350-USB-A.

[Hardware and Cascadia build](build.md) · [Firmware and payload tests](firmware.md)

## Cascadia comparison

Project Cascadia documented a Linux boot on the same iPhone 11 board model using an Apple-silicon Mac, a serial debug cable, and boot inputs from iOS 16.4 (20E247). I also used 20E247 inputs in the early attempts, with Linux Mint as the host and iOS 26.5 installed on the phone. The tests didn’t isolate the host or the installed iOS version as the cause of the failed handoff. [Cascadia bring-up notes](https://github.com/rzxvx/Project-Cascadia/blob/5e40bfb537327f02aa298e2c2e00e116aecbec1d/devices/a13/docs/research/bringup-status.md)

## Boot attempt log

Each entry records the starting state, the test, and the last observable result. Attempts 1–4 are reconstructed from lab notes. Attempts 5 and 6 also have timestamped terminal snapshots. Commands below are historical excerpts.

### 1 · Standard Cascadia boot

**Setup.** The RP2350 established exploited DFU. The Cascadia A13 build had a patched 20E247 iBSS, a packaged m1n1/Linux image, and an Alpine RAM disk. The expected result was a phone-side Linux network interface.

**Run.** Ran `sudo -E ./cascadia a13 boot`. Six transfers reached 100% and the script reported `bootx submitted`. The screen was backlit but black. The host saw Apple Recovery (`05ac:1281`), no new phone serial or network interface, and no reply to a ping at `172.16.42.1`. Recovery was the last confirmed stage. [Attempt 1 lab note](../diagnostics/earlier-attempts/01-lab-note.txt)

### 2 · Repeat after verifying DFU

**Setup.** The phone initially enumerated as `05ac:1227` in ordinary DFU. The boot tool returned `RuntimeError: this is not Pwned DFU device`. Power-cycled the RP2350, reapplied USBLiter8, and checked `irecovery -q` for `PWND: usbliter8` before retrying the same image.

**Run.** Ran `sudo -E ./cascadia a13 boot`. Patched iBSS reached Recovery, all six transfers completed, and the host submitted `bootx`. The phone stayed at `05ac:1281`. Correcting the DFU setup cleared the first error. The later handoff still produced no Linux interface. [Attempt 2 lab note](../diagnostics/earlier-attempts/02-lab-note.txt)

### 3 · White-screen diagnostic

**Setup.** Built a 104-byte ARM64 display stub and packaged it with m1n1, boot arguments, and the N104 device tree. Starting from iOS, entered DFU, ran USBLiter8 on the board, reconnected the phone directly to the laptop, and loaded the patched 20E247 iBSS. Verified Recovery before sending the device tree, trust cache, GPU firmware, and power-management firmware. [Payload build and checks](firmware.md)

**Run.** Uploaded the framebuffer test image with `sudo irecovery -f` and issued `sudo irecovery -c bootx`. The upload finished. The phone showed the Apple logo. The missing white screen left both early payload execution and framebuffer setup unresolved. [Attempt 3 lab note](../diagnostics/earlier-attempts/03-lab-note.txt)

### 4 · Standalone m1n1 proxy

**Setup.** Entered DFU from iOS, applied USBLiter8 on the RP2350, and reconnected the phone to the laptop. Loaded `iBSS.n104.20E247.sigpatched.raw` and verified Recovery with `irecovery -q`. Selected `m1n1-proxy.rkrn.img4` to test the loader with Linux and Alpine removed.

**Run.** Ran `sudo ./boot-from-recovery.sh proxy`. The script reported `bootx submitted`. The phone returned to iOS without exposing the expected m1n1 proxy. This reproduced a failed custom boot with a smaller payload. [Attempt 4 lab note](../diagnostics/earlier-attempts/04-lab-note.txt)

### 5 · Patched iOS 26.5 bootloaders

**Setup.** Prepared N104 iBSS and iBEC from build 23F77. From exploited DFU, used `usbliter8ctl boot` to send `patched/ibss-patched.raw`. Queried Recovery, then wrapped the patched iBEC as `patched/ibec-265-test.img4`. [Patch and container checks](firmware.md)

**Run.** Uploaded the iBEC with `irecovery -f` and sent `go`. On November 6, the 06:07 snapshot showed Apple Recovery after iBSS, the 06:13 snapshot showed Recovery after the iBEC transfer, and the 06:30 snapshot still listed `05ac:1281`. The observations establish that the newer iBSS worked far enough to reach Recovery. They leave iBEC execution unresolved. [Attempt 5 terminal excerpts](../diagnostics/earlier-attempts/05-recovery.txt)

### 6 · Unsigned m1n1 image

**Setup.** Continued in the patched Recovery session from attempt 5. Prepared `boot6/m1n1-unsigned.rkrn.img4`. Sent the N104 DeviceTree and empty trust cache with their Recovery commands (`devicetree` and `firmware`), then uploaded the unsigned image.

**Run.** Issued `irecovery -c bootx`. The 06:40 terminal snapshot on November 6 records all three transfers at 100%. The phone remained at `05ac:1281`. The serial device on the laptop belonged to the RP2350. No m1n1 interface appeared. [Attempt 6 terminal excerpts](../diagnostics/earlier-attempts/06-transfers.txt)

### 7–8 · l41ka / PongoOS

**Setup.** Changed from the Cascadia Recovery route to the l41ka controller route. Prepared l41ka firmware 0.1.0 and Linux controller app 0.0.1 for the Waveshare board. Set up PyUSB and pyserial in the debug checkout and used `scripts/laikadbg.py --cdc` to collect board output. The intended marker was an interactive PongoOS console.

**Run.** The saved controller log shows a DFU target open, the start of the A13 exploit, and a serial read failure. Separately, the project ledger reports a device-transition timeout and `LIBUSB_ERROR_NO_DEVICE`. Neither record identifies a phone-side exception. The surviving records don’t establish which setup and error belong to 7 versus 8, so these runs share an entry. [Controller log](../diagnostics/earlier-attempts/07-08-controller.log) · [Ledger excerpt](../diagnostics/earlier-attempts/07-08-ledger.txt)

<a id="attempt-9"></a>

## Attempt 9 — l41ka handoff

**Setup.** Connected the DFU phone to the RP2350 running l41ka. Started the board’s CDC serial log on the laptop, loaded usbmon, and captured full-length packets on `usbmon0` using dumpcap with `-s 0`.

**Run.** Used the l41ka debug client to probe the target, run the exploit, probe again, send `setup-iboot`, then check the connection and controller errors. The expected result was the phone reappearing as `laikadfu`. The exploit and setup command logs were empty. The board’s serial log and the USB capture provide the sequence below.

| Evidence | Observation |
| --- | --- |
| Controller log | The final entry reads `target opened: pwned-dfu`. [Saved log](../diagnostics/attempt9-20261107-091051/controller.log) |
| USB frames 23844–23847 | The controller acknowledged `setup-iboot` (opcode `0x03000003`) with status 0. |
| USB frames 23848–23851 | The next device poll returned `00000000ffffffff`. The controller reported no open target. |

The laptop captured its USB connection to the RP2350 using usbmon. The board handled the separate USB connection to the phone. In the reviewed l41ka source, the controller sends status 0 **before** it writes the payload or triggers the handoff with DFU abort. It also closes the target connection after a successful abort. The recorded poll therefore cannot distinguish a normal close from a failed handoff. [Controller source](https://github.com/0cyn/l41ka/blob/167c828a086f7d391a0cf90077e819d442658a3e/src/control/TargetWorker.cpp)

The archived payload passed the patcher’s size and marker checks. The exact image flashed to the board is unverified. The records contain no result for the payload writes or abort, and no subsequent `laikadfu` response. The failing operation is still unknown. [A13 patcher source](https://github.com/0cyn/l41ka/blob/167c828a086f7d391a0cf90077e819d442658a3e/src/rom-patcher/iBootPatcherSetup.cpp)

**Capture reference.** The run record lists 23,867 packets on `usbmon0`. The frame numbers come from the saved decode of the capture, archived locally as `attempt-9.pcapng`. File permissions prevented a fresh read of the full capture. To inspect the exchange in Wireshark, use `frame.number >= 23844 && frame.number <= 23851`. [Protocol decoder](https://github.com/0cyn/l41ka/blob/167c828a086f7d391a0cf90077e819d442658a3e/scripts/laikadbg.py)

## Next diagnostic

Record the firmware build. Log each payload write, the boot return address update (`SetBootLR`), and DFU abort, with a timestamp and return code for each operation. Then record whether the phone reappears as `laikadfu`. If all controller calls complete and the interface remains absent, the next check is a phone-side execution marker.

## Sources and tools

| Where it fits | Projects and tools |
| --- | --- |
| Capture and interpretation | [Linux usbmon](https://docs.kernel.org/usb/usbmon.html), [Wireshark USB fields](https://www.wireshark.org/docs/dfref/u/usb.html), [TShark](https://www.wireshark.org/docs/man-pages/tshark.html) |
| Hardware and l41ka route | [Waveshare RP2350-USB-A](https://www.waveshare.com/wiki/RP2350-USB-A), [l41ka firmware and debug tools](https://github.com/0cyn/l41ka), [l41ka controller app](https://github.com/0cyn/l41ka-app), [PongoOS](https://github.com/checkra1n/PongoOS) |
| Cascadia boot route | [Project Cascadia checkout](https://github.com/rzxvx/Project-Cascadia/tree/5e40bfb537327f02aa298e2c2e00e116aecbec1d), [Hoolock m1n1 build revision](https://github.com/HoolockLinux/m1n1/tree/d5a10ac52a6468484854419a6c5130f1d62073eb), [T8030 Linux source](https://github.com/yhavry/Hoolock-linux/tree/t8030), [Hoolock A13 notes](https://github.com/yhavry/Hoolock-Docs-A13) |
| Firmware inspection and Recovery | [Trussh USBLiter8 patchfinder checkout](https://github.com/Trussh/usbliter8/tree/09d87ddd5e445a08c262829883e96bfb1f90631f), [ipsw](https://github.com/blacktop/ipsw), [Capstone](https://www.capstone-engine.org/), [irecovery](https://github.com/libimobiledevice/libirecovery) |
| Earlier build environment | [Docker](https://docs.docker.com/), [QEMU](https://www.qemu.org/), [LLVM and Clang](https://llvm.org/), [Alpine Linux](https://alpinelinux.org/) |
| Comparison and background reading | [USBLiter8 analysis](https://www.amcrypto.jp/security/usbliter8_Analysis.html), [Liter8](https://github.com/Xplo8E/Liter8), [Project Sandcastle](https://github.com/corellium/projectsandcastle), [Parts Hacker’s A13 report](https://www.patreon.com/partshacker/posts/linux-boots-on-172091834), [Cyber-Scholar USBLiter8](https://github.com/Cyber-Scholar/usbliter8), [USBLiter8 Arctic](https://github.com/kaffeindecaf/usbliter8-arctic), [surrealra1n](https://github.com/pwnerblu/surrealra1n) |
