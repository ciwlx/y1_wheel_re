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

## Analysis
This section is roughly ordered along the time.

* [Linux-level input events](01-linux-input.md)
* [Kernel & driver binary reverse engineering](02-kernel-driver.md)
* [Wheel controller 8051 firmware reverse engineering](03-8051-firmware.md)

