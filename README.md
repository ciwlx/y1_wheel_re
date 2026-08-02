# y1_wheel_re
## What
Reverse engineering touch wheel control software of Innioasis Y1 music player device.

## Why
The touch wheel works in the same principle as the "click wheel" in the apple ipod:
* Touch and move to rotate the virtual "wheel", implemented with a touchpad
* "Click" one of the four underlying buttons
* Click center button (though not strcitly a part of the click wheel hardware).

Though what (should) interests us is the differences.
1. A touch alone won't light up the screen from automatic(idle) screen off. Any one of the buttons should be pressed.
2. Wheel "rotation" is not sensitive enough (subjectively).
3. (Though not obvious) The wheel rotation is "relative" only.

My goal, as a hobbyist, was to find where and how exactly 1 and 2 come from, and to improve the device as far as I can. I think I'm satisfied with the result.

## Knowledges
This section is roughly ordered along the time.

As a disclaimer, I believe what I have is a more common "Type A" Y1.

### Top-level user experience
Here are the differences again:
1. A touch alone won't light up the screen from automatic(idle) screen off. Any one of the buttons should be pressed.
2. Wheel "rotation" is not sensitive enough (subjective).
3. (Though not obvious) The wheel rotation is "relative" only.

### Linux input events
First thing I did was to enable ADB on Y1. And did `getevent -l`. (Let's skip over the Android key event handling layer.) 

Two input devices are relevant:
1. `/dev/input/event0 - name: "mtk-kpd"`
   - This one emits a `EV_KEY` (232, `KEY_REPLY`), follwed by `EV_SYN`, when the center button is pressed or released.
   - Trivial. For the rest of this article, let's just forget this one because it's handled separately, and reasonably.
2. `/dev/input/event2 - name: "mtk-tpd"`
   - This handles virtual wheel rotations, and button clicks.
   - For button clicks,
     - Like the center button, it emits an `EV_KEY` event for each press and release of any button.
     - The keycodes are 158/164/163/165 (back/nextsong/playpause/prevsong) for U/D/L/R. It follows the functions, rather than the placements of the buttons.
   - For virtual wheel rotations,
     - The circle is divided into eight sections with 45 degs each. That makes eight section borders, at 22.5, 67.5, 122.5, ... , 337.5.
     - A rotation/scroll tick is detected when the touching finger moves across a section border; ie) leaves a section and enters another section.
     - Only the relative movement direction (CW or CCW) across the section border is reported.
     - At a tick, a quick series of four events are emitted: `EV_KEY` for down, `EV_SYN`, `EV_KEY` for up, `EV_SYN`. That is, one rotation tick is regarded/emulated as same as a quick click (press and release) of a button.
     - The keycodes are 105/106 (`KEY_LEFT`/`KEY_RIGHT`) for CCW/CW repectively. Once again, the keycodes follow the functions (scrolling through list).

In short, the driver module (mtk-tpd) is shy to give out the touch details.

### mtk-tpd driver logs
Let's see if `cat /proc/kmsg` gives anything. Luckily `mtk-tpd` prints just enough data to the logs.

```
<4>[  247.754940] (0)[62:mtk-tpd]mtk-tpd: *********** ning  touch panel
<4>[  247.757252] (0)[62:mtk-tpd]mtk-tpd: [apt32] 0x55 2 8 1 0 3
<4>[  247.757289] (0)[62:mtk-tpd]mtk-tpd: *****apt32 ctrl report Linux keypos = 8******
```

After some experiments, the second line seems:
```
[apt32] 0x55 <type> <key> <direction> 0 3
```
`type` is:
* 1 when a button is pressed
* 2 when a button is released
* 3 for a rotation tick

`key` is valid when the type is 1 or 2. The value is 4/1/8/2 for U/D/L/R. Or, 1/2/4/8 in CCW starting from the bottom button.

`direction` is valid only when the type is 3. The value is 3/1 for CW/CCW. 

0x55, 0, and 3 were always same throughout my tinkering.

So, it is certain `mtk-tpd` driver is doing its job handling wheel clicks and rotations. Then we should look at how it does.

### mtk-tpd driver binary (ARM)
From `lsmod`, we know the `mtk-tpd` driver was compiled into the kernel binary.

Naturally `/proc/kallsyms` comes in handy. And there is a magic trick I learned: `echo "0" > /proc/sys/kernel/kptr_restrict`.

By unpacking the firmware file, you get linux kernel file, which is in zImage format. Unpack it again to get the real kernel binary. Now it's ready to be disassembled, with Ghidra. The binary is in ARM Little Endian format.

We see standard `tpd_init`, `tpd_probe`, and such. And it seems the wheel controller is communicating via I2C (calling `i2c_register_driver`).

Eventually, voila, `touch_event_handler`! ... Followed by an immediate disappointment. All it does is read 6 bytes the I2C peripheral gives (`tpd_read_byte`), and if-then-else to emit key events. The six number log line from the previous section is all the data to which the driver have access to.

### tpd firmware update routine
However, scanning through the kernel functions, we see `tpd_local_init` actually register two I2C drivers: one named `APT32F` and `UPDATE`. In fact `update_probe` calls `spt511x_firmware_update`. In fact SPT5115S IS the controller chip on the wheel ribbon cable!

Thankfully, `spt511x_firmware_update` is straightforward: through I2C, 1) request firmware version, 2) if different, write flash & eeprom, 3)reboot.

And the firmware to send to and write on the SPT5115S chip, which runs 8051, is stored in plain bytes in kernel binary. 

### SPT5115S tpd controller firmware (8051)
Now we have 3964 bytes of 8051 bytecode. I have to confess I'm in love with Ghidra by now.

SPT5115S has a separate DSP which probably measure and convert raw capacitance from eight sensor pads to more convenient values. A large part of the 8051 code is for controlling the DSP through `EXTMEM`, but I haven't really looked through the protocol itself, because the sensor values seem reasonable enough.

Other parts of the 8051 firmware do detect virtual rotation ticks from the sensor data, and I2C communication with the main MCU. I focused on the detection logic.

Below are some important functions.

* `0759`
  - Main - initialization and loop
* `0752`
  - The function that called in main loop. Calls `0920` then `077b`.
* `0920`
  1. Read DSP data including touch pads and buttons.
  2. Preprocess/extract data into buttons/pad status. 
  3. Set up flag to process touch if neccessary.
  4. Emit button events if button status changed. (type=1/2, key=1/2/4/8)
* `077b` 
  1. Call `0dcb` to calculate touch position.
  2. Emit a rotation tick if touch position changed. (type=3, key=1/3)
* `0dcb`
  1. Looping through 8 touch values, find max value index, and calculate sum(value) and sum(value*index).
  2. Invalidate position to `0xFF` (-1), not touched, if max value is too low (`<5000`).
  3. 



