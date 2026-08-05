# 8051 controller firmware reverse engineering & tweaking
## SPT5115S touch pad controller
From kernel binary we extract 3964 bytes of 8051 bytecode. It's Ghidra time again.

SPT5115S has an 8051 core, and a separate DSP which (probably) measures and converts raw capacitance from eight sensor pads to more convenient values. A large part of the 8051 firmware is for controlling the DSP via `EXTMEM`, but I haven't really looked through the protocol itself, because the sensor values seem reasonable enough.

Other parts of the 8051 firmware detect virtual rotation ticks from the read data, and communicate with the main MCU via I2C. Here I focus on the detection logic.

## Shared memory interface with main MCU: `INTMEM:7b ~ 7f`
These 7 bytes of 8051 internal memory are directly accessible from the main MCU with `tpd_read_byte` and `tpd_write_byte` function. Their interface simple, for read, an index. For write, an index and the byte data.

Below is the usage of each byte:
| Offset | Index | Set By |Value | Details |
| ---- | ---- | ---- | ---- | ---- |
| 7b | 0 | 8051? | ?? | Read once by `tpd_probe`. Seems not accessed by 8051 fw. |
| 7c | 1 | 8051? | 0x55 | probably Magic |
| 7d | 2 | 8051 | 1/2/3 | Type: 1 = button down, 2 = button up, 3 = rotation tick |
| 7e | 3 | 8051 | 1/2/4/8/0 | Button: 1 = down, 2 = right, 4 = up, 8 = left, 0 = not button event  |
| 7f | 4 | 8051 | 1/3 | Direction: 1 = CCW, 3 = CW. Strangely button events still sets this byte as 1. |
| 80 | 5 | MCU | 0/1 | Suspend: 0 = resume, 1 =suspend. Set by `tpd_suspend` and `tpd_resume`. `0920` (below) reads this and do suspend (guess). |
| 81 | 6 | MCU | 0/0x55 | Set by `touch_event_handler` following battery charger presence? Seems not used by 8051 fw. |

## Functions
Below are the offsets of some important functions from 8051 binary.

### `0759`: main
initialization and loop

### `0752`: loop routine
Only function that is called in the main loop. If the controller is setup and running correctly, calls `0920` then `077b`. Else re-initialize something (sorry, out of scope).

### `0920`: process input from DSP data
Read/communicate DSP readings via `EXTMEM`, preprocess, and handle button inputs. Also enter suspend by I2C command or no input for timeout.

1. Read DSP status data, including touch status and buttons.
2. Enter suspend if received I2C command or timeout.
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

1. Looping through 8 touch values, read sensor value, subtract baseline value, and find max value index (let's say `max_i`).
2. Invalidate position to `0xFF (-1)`, not touched, if max value is too low (`<5000`).
3. Else, from max value and adjacent two values (values at `max_i-1` and `max_i+1`, lets say `ccw_i` and `cw_i`), linearly estimate relative position along the three pads. The formula is roughly `5 * (max_i + 2*cw_i) / (ccw_i + max_i + cw_i)`, which results in [0,10).
4. Add the absolute offset `5 * (idx-1)` to the relative position value. It's `idx-1` because relative position is [0,10), not [-5,5).

The capacitance deviation from the base value for each pad is theoretically linear to the size of the contact (finger) area, so simple linear weighted sum would yield reasonable accuracy. At least I felt no jumble or such. (Haven't looked at the raw sensor readings.)

### `0a9d` ~ `0b1f`: event functions for each condition
10(+2) small functions called when a button/tick event is detected. 8 of them are for buttons and are combinations of up/down and 1/2/4/8. 2 of them are for CW and CCW ticks. 

They write event data bytes at offset 2, 3, 4, then call `0aa7`.

### `0aa7`: notify event data ready to send
I don't really looked into I2C communication control, but this function is called after buffer writes so I guess this prepares and notifies the I2C controlling parts of the firmware.

## Tweaks
I'm not at all familiar with 8051 and didn't want to setup build toolchain or such for it. So I relied on Ghidra assembler to hand-tweak some parts of the controller firmware.

TBC
