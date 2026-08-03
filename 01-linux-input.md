# Linux-level input handling
## Linux input events
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

## mtk-tpd driver logs
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
