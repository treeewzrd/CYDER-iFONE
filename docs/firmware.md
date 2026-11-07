# Firmware and diagnostic payloads

## Isolating the Linux payload

After the Cascadia boot produced a dark screen and no network response, I tried two smaller tests. The standalone m1n1 proxy removed the Linux kernel and Alpine from the boot path. It still produced no proxy connection.

For a visible test, I built a 104-byte ARM64 framebuffer program with Clang and LLD in the Cascadia container. I padded it to 4,096 bytes, compressed it to 103 bytes, and packaged it with m1n1, boot arguments, and the device tree. I checked the ARM64 image magic and component boundaries in the 1,014,183-byte payload. The boot arguments started at offset `0xF4000`, immediately after m1n1. The device tree began at `0xF40AC`. I rebuilt the repaired IMG4 with `ipsw` 3.1.730 and compared it with the 3.1.714 output using `cmp`. The files matched byte for byte. After transfer and `bootx`, the phone showed the Apple logo. The program’s expected white screen never appeared.

## Testing newer bootloader inputs

The original Cascadia inputs came from iOS 16.4. To test the newer boot chain, I extracted the N104 iBSS and iBEC bootloader images from the iOS 26.5 (23F77) firmware. The two unpatched RELEASE images compared identical at 2,443,008 bytes each. The first validation pass expected the string `iBoot-`. These images contained `iBoot for n104`. Correcting the identifier check allowed patching to proceed.

I ran the USBLiter8 patchfinder and checked the output with Capstone disassembly. Both files retained their original size. [Patchfinder revision](https://github.com/Trussh/usbliter8/tree/09d87ddd5e445a08c262829883e96bfb1f90631f)

| Bootloader | Patcher report | Binary comparison |
| --- | --- | --- |
| iBSS | 5 patches, 18 functions found | 17 changed bytes |
| iBEC | 14 patches, 19 functions found | 64 changed bytes |

Disassembly confirmed edits at file offset `0xE78F0` and the CTRR-related offsets `0x273A4`, `0x273B0`, and `0x273C4`. The patchfinder reported the validation function at `0x8700204D0`. The modified instruction was at `0x8700E78F0`. Whether that edit changed the intended check remains open.

Patched iBSS brought the phone into Recovery. I wrapped the patched iBEC as an IMG4. Inspection showed an `ibec` payload of 2,443,008 bytes and no IM4M manifest. I uploaded that container and issued `go`. The phone stayed in Recovery. An unsigned m1n1 image also transferred completely without producing a proxy interface. [Boot results and USB evidence](engineering.md)

## Project files

| Component | Contents |
| --- | --- |
| Project Cascadia | Build environment, boot inputs, compiled images, packaged RAM disk, and framebuffer stub. |
| USBLiter8 patchfinder | iBoot patchfinder and patching scripts. |
| iOS 26.5 firmware | Extracted firmware, patched images, test iBEC container, and the attempt 6 unsigned image. |
| l41ka release | Extracted firmware and Linux controller. |
| l41ka debug checkout | Controller, exploit, patcher, and payload source used to interpret attempt 9. |
| CYDER iFONE | Build documentation, boot attempt log, and diagnostic evidence. |
