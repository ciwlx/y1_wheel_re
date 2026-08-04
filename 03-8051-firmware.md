# 8051 controller firmware reverse engineering
## SPT5115S touch pad controller
From kernel binary we extract 3964 bytes of 8051 bytecode. It's Ghidra time again.

SPT5115S has an 8051 core, and a separate DSP which (probably) measures and converts raw capacitance from eight sensor pads to more convenient values. A large part of the 8051 firmware is for controlling the DSP via `EXTMEM`, but I haven't really looked through the protocol itself, because the sensor values seem reasonable enough.

Other parts of the 8051 firmware detect virtual rotation ticks from the read data, and communicate with the main MCU via I2C. Here I focus on the detection logic.

## Functions
Below are the offsets and my own 'pet names' of some important functions from 8051 binary.

### `0759`: main
initialization and loop

### `0752`: process
Only function that is called in the main loop. If the controller is setup and running correctly, calls `0920` then `077b`. Else re-initialize something (sorry, out of scope).

### `0920`: process input from DSP data
Read/communicate DSP readings via `EXTMEM`, preprocess, and handle button inputs.

1. Read DSP data including touch pads and buttons.
2. Preprocess/extract data into buttons/pad status.
3. Check if touch data is valid, and set up corresponding flag.
4. Emit button events if button status changed. (type=1/2, key=1/2/4/8)

### `077b`: process touch pad
If touch sensor data is valid, track touch position change and emit touch event if necessary.

1. Check the flag set by `0920`.
2. Call `0dcb` to calculate touch position.
3. If touching, accumulate touch position change.
4. Emit a rotation tick if the accumulated change surpass a set threshold which is 5 (and -5). (type=3, key=1/3)

### `0dcb`: calculate estimate touch position
Estimate touch position from 8 sensor values. The return value is one of 40 (0~39) estimate position values, each with approximately 9 degress apart along the circular pad.

1. Looping through 8 touch values, subtract baseline value, and find max value index (let's say `max_i`).
2. Invalidate position to `0xFF (-1)`, not touched, if max value is too low (`<5000`).
3. Else, from max value and adjacent two values (values at `max_i-1` and `max_i+1`, lets say `ccw_i` and `cw_i`), linearly estimate relative position along the three pads. The formula is roughly `5 * (max_i + 2*cw_i) / (ccw_i + max_i + cw_i)`, which results in [0,10).
4. Add the absolute offset `5 * (idx-1)` to the relative position value. It's `idx-1` because relative position is [0,10), not [-5,5).

The capacitance deviation from the base value for each pad is theoretically linear to the size of the contact (finger) area, so simple linear weighted sum would yield reasonable accuracy. At least I felt no jumble or such.
