# 8051 controller firmware reverse engineering & tweaking
## SPT5115S touch pad controller
From kernel binary we extract 3964 bytes of 8051 bytecode. It's Ghidra time again.

SPT5115S has an 8051 core, and a separate DSP which (probably) measures and converts raw capacitance from eight sensor pads to more convenient values. A large part of the 8051 firmware is for controlling the DSP via `EXTMEM`, but I haven't really looked through the protocol itself, because the sensor values seem reasonable enough.

Other parts of the 8051 firmware detect virtual rotation ticks from the read data, and communicate with the main MCU via I2C. Here I focus on the detection logic.

## Functions
Below are the offsets of some important functions from 8051 binary.

### `0759`: main
initialization and loop

### `0752`: loop routine
Only function that is called in the main loop. If the controller is setup and running correctly, calls `0920` then `077b`. Else re-initialize something (sorry, out of scope).

### `0920`: process input from DSP data
Read/communicate DSP readings via `EXTMEM`, preprocess, and handle button inputs.

1. Read DSP data including touch pads and buttons.
2. Preprocess/extract data into buttons/pad status.
3. Check if touch data is valid, and set up corresponding flag.
4. Emit button events if button status changed. (type=1/2, key=1/2/4/8)

### `0774`: process touch pad
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

The capacitance deviation from the base value for each pad is theoretically linear to the size of the contact (finger) area, so simple linear weighted sum would yield reasonable accuracy. At least I felt no jumble or such. (Haven't looked at the raw sensor readings.)

### `0a9d` ~ `0b1f`: event functions for each condition
10(+2) small functions called when a button/tick event is detected. 8 of them are for buttons and are combinations of up/down and 1/2/4/8. 2 of them are for CW and CCW ticks. 

They write event data bytes to a memory buffer then call `0aa7`. This buffer data is where the kernel driver reads from. You can see that only 3 bytes from the 6 bytes logged are actually changing.
* `INTMEM:7c`: event type - 1 = button down, 2 = button up, 3 = rotation
* `INTMEM:7d`: button - one of 1/2/4/8. valid if type is 1 or 2
* `INTMEM:7e`: direction - 1 = CCW, 3 = CW. valid if type is 3.

### `0aa7`: notify event data ready to send
I don't really looked into I2C communication control, but this function is called after buffer writes so I guess this prepares and notifies the I2C controlling parts of the firmware.

## Tweaks
I'm not at all familiar with 8051 and didn't want to setup build toolchain or such for it. So I relied on Ghidra assembler to hand-tweak some parts of the controller firmware.

### Report absolute position
The position calculation function `0dcb` already returns absolute position values, including not touching `0xFF` value. To report the values to main MCU, function `0774` should be patched. 

The patched part is below:
```
 CODE:077b  8f 40           MOV                wheel_pos,R7
 CODE:077d  ef              MOV                A,R7
 CODE:077e  24 29           ADD                A,#0x29              (41)
 CODE:0780  95 42           SUBB               A,wheel_pos_prev
 CODE:0782  ff              MOV                R7,A
 CODE:0783  94 80           SUBB               A,#0x80              (128)
 CODE:0785  50 45           JNC                LAB_CODE_07cc
 CODE:0787  ef              MOV                A,R7
 CODE:0788  75 f0 28        MOV                B,#0x28              (40)
 CODE:078b  84              DIV                AB
 CODE:078c  e5 f0           MOV                A,B
 CODE:078e  94 03           SUBB               A,#0x3
 CODE:0790  50 3a           JNC                LAB_CODE_07cc        (Emit event)
 CODE:0792  22              RET                                     (No event)
```

`wheel_pos` = `INTMEM:40`, `wheel_pos_prev` = `INTMEM:42`. Code from `07cc` is the wrap-up part of the original `0774` function, which emit the event and update `wheel_pos_prev`. The patched code jumps over about half of the original function, and the patch leaves it as an unreachable section.

In short, emit absoulte value `wheel_pos` if the change is greater or equal to 2, including touch start and end.
