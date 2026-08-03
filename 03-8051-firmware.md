# 8051 controller firmware reverse engineering
## SPT5115S tpd controller firmware
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
  1. Looping through 8 touch values, subtract baseline value, and find max value index.
  2. Invalidate position to `0xFF` (-1), not touched, if max value is too low (`<5000`).
  3. 

TBC
