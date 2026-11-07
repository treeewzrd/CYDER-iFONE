# CYDER iFONE

I set out to build and boot a RAM-only Linux system on my iPhone 11. I configured Docker and ARM64 emulation, prepared the phone’s boot inputs, and used Project Cascadia to build m1n1, the kernel, and an Alpine filesystem. I set up a Waveshare RP2350 board to get the phone into exploited DFU, then loaded a patched bootloader and reached Recovery.

The phone accepted the boot images, but I couldn’t confirm that it ran Linux. I built a small ARM64 program that should turn the screen white if it ran. The phone showed the Apple logo. I also tried the m1n1 loader on its own, with Linux and Alpine removed. Its USB interface never appeared. That narrowed the investigation to a boot path I could test without the kernel.

I still want to use this phone as a mini Linux computer with Wi-Fi and a working camera. For now, I’m using the failed boots to learn where the handoff stops. On attempt 9, the controller reported success. I compared the USB capture with the controller source and found that the code sends that response before it writes the payload. The next thing I need to log is whether that write finishes.

| Project record | Contents |
| --- | --- |
| [Hardware, setup, and Cascadia build](docs/build.md) | Board photo and connections, host configuration, missing manifest, compiled images, and packaging. |
| [Firmware and diagnostic payloads](docs/firmware.md) | Framebuffer test, proxy image, iOS 26.5 bootloader patches, and checks against the built files. |
| [Boot diagnostics](docs/engineering.md) | Results by attempt, Wireshark frame numbers, controller source, and the remaining gap. |
| [Projects and tools](docs/engineering.md#sources-and-tools) | Upstream code, equipment, and references. |
