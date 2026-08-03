# Linux device driver analysis
## mtk-tpd driver binary
From `lsmod`, we know the `mtk-tpd` driver was compiled into the kernel binary.

Naturally `/proc/kallsyms` comes in handy. And there is a magic trick I learned: `echo "0" > /proc/sys/kernel/kptr_restrict`.

By unpacking the firmware file, you get linux kernel file, which is in zImage format. Unpack it again to get the real kernel binary. Now it's ready to be disassembled, with Ghidra. The binary is in ARM Little Endian format.

We see standard `tpd_init`, `tpd_probe`, and such. And it seems the wheel controller is communicating via I2C (calling `i2c_register_driver`).

Eventually, voila, `touch_event_handler`! ... Followed by an immediate disappointment. All it does is read 6 bytes the I2C peripheral gives (`tpd_read_byte`), and if-then-else to emit key events. The six number log line from the previous section is all the data to which the driver have access to.

## tpd firmware update routine
However, scanning through the kernel functions, we see `tpd_local_init` actually register two I2C drivers: one named `APT32F` and `UPDATE`. In fact `update_probe` calls `spt511x_firmware_update`. Indeed SPT5115S is the controller chip on the wheel ribbon cable!

Thankfully, `spt511x_firmware_update` is straightforward: through I2C, 1) request firmware version, 2) if different, write flash & eeprom, 3)reboot.

And the firmware to send to and write on the SPT5115S chip, which runs 8051, is stored in plain bytes in kernel binary. 
