Central to the ability of controlling the K40 laser and making it burn appropriately is an understanding of the M2-Nano control protocol, generally known as Lhmicro-GL.

This wiki page gives an overview of the Lhmicro-GL and associated protocols (like USB) - more information can be found on the [Edutech.ch wiki](https://edutechwiki.unige.ch/en/Lhystudios_M2_Nano).

### Contents
* [Lhmicro-GL](#lhmicro-gl)
* [USB transmissions](#usb-transmissions)

## Lhmicro-GL

### Fundamentals
Unlike command rich formats like gCode, the Lhymicro-GL was developed specifically for lasers and is maximally basic for that, typically using single letters to denote an operation.
 
All commands in the Lhmicro-GL protocol do one of only 12 things:

1. Turn the laser ON or OFF
1. Changes the direction of the x-axis stepper motor
1. Changes the direction of the y-axis stepper motor
1. Changes whether ticks are suppressed on the x-axis
1. Changes whether ticks are suppressed on the y-axis
1. Appends values to a distance magnitude register (see below for an explanation)
1. Sets the tick rate to the stepper motor
1. Executes the current command state
1. Changes whether the programmed tick rate is sent to the stepper motor chips or not
1. Resets the board
1. Pauses the device
1. Homes the laser head

The commands used can be completed in a single command routine, which on a 22 MHz 8051 chip (see [Hardware](https://github.com/meerk40t/meerk40t/wiki/Tech:-Lhymicro-M2-Nano-Hardware)) is 12 ticks long. 

Converting this to the speed at which you can switch the laser on and off, this equates to being fast enough to raster alternate pixels at 400mm/s, and amazingly enough it can achieve this on a 40 year old processor.

### Protocol Modes
There are primarily two different modes for the M2 board: `Default` and `Compact` (可压缩). Whilst these two modes do share some of the same commands and states, they primarily work in some fundamentally different ways. One of the main differences is Compact mode changes the stepper motor ticks to the set tick rate, which means the steppers move at a set speed. It also changes when the command state is executed: in Default mode you need a `N` command to commit the distance magnitude to the get ticked out, whilst in Compact mode the distance is executed immediately.

The encoding uses uppercase values as commands and lowercase values as units of magnitude. The magnitude values are always summed together and are positive values in a flagged direction. The direction flags are `T`, `B`, `R`, and `L` these are likely top, bottom, right, and left for their namesakes, however they correspond to Left (T), Right (B), Bottom (R), and Top (L). This is weird, and getting the command meaning right in your head means rotating by 90 degrees and flipping horizontally.

#### Default Mode
In default, we can set the speed, laser operation, and direction flags. The things we need for compact mode, while not in compact mode. We cannot unset the speed without a calling `I` in default mode or `@` in compact mode. Calling `I` will kill any processes, or states, and reset the chip. All commands and states execute at the end of a block, this in default mode is, usually the command `N`. The final flagged states and magnitudes are executed when entering compact mode. The X magnitude and Y magnitude are independent of each other. Magnitudes in either direction are added together.

Note that in default mode, the last flagged direction gets the magnitude so `RzzzzL` assigns the 1021 mils (4 * 255) of distance to the `L` command (Top / -Y direction). Setting the laser `D` in default can leave the laser on without the head moving. Sending `IDS1P` will simply turn the laser on. Sending `IUS1P` will turn the laser off. It is best to execute anything remaining in a command states before switching modes, doing something else might get weird results if some left over magnitude is somewhere, since it will get assigned to some direction. In default mode different directions also combine and trigger as a block, so `RzzTzzN` is a diagonal move, but `RzzNTzzN` since it's two blocks, is bottom (+y) move followed by a left (-x). If the magnitudes are different it performs diagonal then linear moves so RzTzzN is a diagonal move for 255 mils of distance and an exclusively left move for the remaining 255. All moves in default-mode are performed at full default speed.

#### Compact Mode
Switching to compact mode uses `S1E` or `S0E`. Compact mode will always utilize the speed value and step values set with `C`, `V` and `G`. Within compact, the permitted commands are `D`,`U`,`L`,`R`,`T`,`B`,`M`,`F`,`@` as well as the lowercase magnitude values, anything else will likely cause the mode to stop and often can result in non-understood behaviors. When exiting compact mode with `SE` we usually end with a mode exit command (`F` or `@`) and that command will be relevant only after doing command `SE` in default mode. The `N` command exit compact mode and `SE` triggers the end. If we ended compact with a `F` command, then the state should be considered locked until the queue finishes and querying the device returns a status with PEMP (empty) signal. If we send an `@` command, it resets the device as part of the command stream. We reset our speeds and other modes and can set our desired modes again. These changes only apply after calling `SE`. Since these are taken together we usually end a mode with `@NSE` or `FNSE`. Simply exiting compact mode requires an `N` command. But, this will not clear the G, V, or C values. If 'C' is set twice it put you in 1/12th speed mode. (The 8051 takes 12 clock cycles to process a command so there's a internal clock for that). So if we are expecting to use a different speed, or are unsure, we should just reset (@) or finish (F).

Do note, however, that if we reset, the machine may still be doing things and we cannot know whether or not these are finished. The `@` command resets as part of the stream. But, commands like `I` and `P` commands execute out of stream. Calling `I` causes the system to immediately reset the board, losing the current buffer and state. Calling `P` causes it to pause. A packet of `PN` causes the system to halt until resumed with another `PN` packet.

Within compact mode, every command is executed as soon as it's sent. So `D` turns the laser on. `Rzz` moves +Y zz (512 mils) immediately. To perform a diagonal, the `M` command is added (in default_mode `M` does the same as no command, causing the momentum to be assigned to the default `R` (+Y) and executed). In compact, diagonal M is the in the x-direction set on the stepper chip and y-direction set on the other stepper chip. So `RRLTB` is LB and goes (+x,-y).

It is customary to set these directions just before `S1E` when initializing the compact made. So usually we enter compact mode with something `NRBS1E` this sets the initial states to `RB` when M is executed. For raster-mode this sets the `R` state on the stepper chip but the direct direction mode of `B` (right) for any values. So `NRBS1EDzzU` will set the `R` direction then the `B` direction. Setting the `R` there means if we're in harmonic mode, when we go from `B` to `T` the step will be made in the `R` direction. Here we set the initial compact direction to be `B` last direction command set. So the laser will be turned on `D` and the zz (510 mils of distance) will execute their state at the `U`, going in the last set direction `B`. 

The commands in default mode apply any magnitude assigned at the next command. So the command state will always be executed before processing any command.

Let's look at another example:
`IV2241553G003RcNRBS1EiDzzzzzz111TmD...`

We reset the device with `I` command and apply the `V` with the value `2241553`. The setting for G is set to 003. Since there isn't a second G command this applies to both the switch from B->T and T->B. We set the R flag (bottom, +y) and assign c (3) magnitude. This is triggered by `N` command because it's still in default mode. Then we assign flags `RB`, the `B` flag being assigned last sets that as the ordinal direction and enter into compact mode `S1E`. A magnitude of `i` (9) is assigned and executes at the `D` going `B` direction (since it was the last direction set). The `D` makes the laser down. And `zzzzzz111` (6 * 255 + 111) units of additional `B` direction with the laser down are triggered at the `T` flag.
However, since G is set, the switch from B to T triggers the step. The laser to turn off, and a step in the 'R' (Bottom) direction is 3 units (assigned by G) and the direction reverses. In the `T` direction with the laser off `m` (13) units are travelled in the `T` direction (Left), executed at the `D` command which then turns the laser again.

With this in mind, the chinese software generally rasters with code like `UcDcUiDcUiDiUcDiUcDiUc` while flickering the laser.

Compare this example with the original to the same for x-step rastering:
```
Y-Step: IV2241553G003RcNRBS1EiDzzzzzz111TmD...
X-Step: IV2221554G003BcNBRS1EiDzzzzzz111LmD...
```

Here we start the same except the speedcode acceleration/deceleration value is 4 instead of 3. Likely owing to the fact that vertical rastering is moving more stuff for and need more time to decelerate it. But, the other notable difference is that we use `RB` rather than `BR` This makes `R` (Bottom +y) the last flag set and thus the direction we travel for the `i` units assigned by the D command (which turns the laser on). Here though, we trigger the shift to L which steps in the x direction rather than the y direction since we switched from going from Bottom to going Top. This step is done in the last flagged x-direction which is `B` (right, +x) as set between the `N` command and `S1E`

### Harmonic Steps
While we can set directional flags within the compact mode block, this is problematic because the `G` mode steps on each reversal. The harmonic motion `G` trigger on direction change within compact mode. So, if we change modes within the compact block, it can trigger a harmonic step. So these mode changes are done outside compact blocks initially.

The `G` settings operates only within compact mode and triggers a step between 0-63 mils at the transition. Two G codes can be concatenated to implement unidirectional rastering by stepping by units at one end and no units on the other end `G000G002` means step down two units on a right-to-left transition and zero units on a left-to-right transition. The `C` setting causes `G` to become void and there's no way to unset `C` without calling reset `@` or `I`. It's not strictly clear what it does. The `G` parameter triggers when we switch directional modes. So if we're going `B` (+X, Right) and trigger a `T` command (-X, Left). The step is triggered. If we gave a magnitude during this step, sometimes it gets weirdly applied, to the step, and is performed diagonally like an M command with that particular distance. If we wish to just make the given step set by the `G` we should always change direction flags without a distance. During the raster step the laser state is always set to off. It has an implicit `U` command.

The step is the same in each flagged direction, so if we do an `L` or `R` commands to change the vertical we will invoke the step operations (moving in the X direction, and turning off the laser) when we change the Y-direction currently set in `G` mode. This means we should be careful to set all the desired modes before entering the `S1E` because in compact mode as all direction state changes trigger steps. If `C` is set, the value of `G` is ignored and the direction state changes. The direction changes might also get ignored if we enter compact mode with S0E.

### Distance Magnitude
The distance magnitude can be a 3 digit number 255 and below, a lowercase letter, or a few different symbol which kinda qualify as letters and aren't found in the original Chinese software. The letter `z` is a special case being equal to +255 unless it is directly preceded by `|`. So all `|` commands are worth 25 which is equal to the value of `y`. However, when `|` is just before a `z` the `z` is 26 units rather than 255. Which combined with the `|` gives a distance of 51. The only other way to hit this number would be something like `~t` however that doesn't appear in the original software. The original software use `|` for all values from 26 to 51.

|Letter|Distance||Letter|Distance||Letter|Distance||Letter|Distance|
|-|-|-|-|-|-|-|-|-|-|-|
|a|1||j|10||s|19||\|z|51
|b|2||k|11||t|20||[000-255]|value
|c|3||l|12||u|21||`|0
|d|4||m|13||v|22||{|28
|e|5||n|14||w|23||}|30
|f|6||o|15||x|24||~|31
|g|7||p|16||y|25|
|h|8||q|17||z|255|
|i|9||r|18||\||25|


Note, these values are all added up, so you can slap anything together and it'll just be added to the magnitude, usually this is something like "zzzzzzzzzzzz" and a remainder like "|g". The only difference that matters is `|z` is 51 since the `|` causes the 255 power of `z` to be 26. But this only applies for the character immediately after the `|` for the next character, so `|az` is 25+1+255 and `|za` is 25+26+1. So if you send the word 'laser' it adds up to 55. And will go a distance of 55 mils.

### Commands
Calling `S1P` triggers instant execution ignoring any commands that occur after that. This maybe because the P command causes the system to pause. This kind of suffix is usually used for one-off commands like move right `IR<distance>S1P` After the `P`, rest of commands in the packet don't matter unless there is another P. Whenever there are two P commands in the same packet the device rehomes. Unless the second `P` command was flushed out with an `I` command. `IPP` will rehome the device, but `PIP` will not. If the `P` commands are in different packets they won't cause a rehome. `S2P` works like `S1P` but unlocks the rail so that it move more freely. This can be done as a single trigger like `IRzzzNS2P` it will release the rail and cause it to jump back with the tension losing registration. Doing `IRzzzS2P` will break the `Rzzz` command and fail to execute.

* `I`: Initialize. Set the machine to initial state. Deletes the buffer. Any commands currently in the stack are deleted. This includes all commands preceding the I within the same packet. This is executed out of sequence. This does not necessarily reset modes set on other chips. Turning the laser on with IDS1P then sending any additional I commands does not turn the laser off.
* `P`: Pause. Triggers a machine pause. If in compact mode and doing stuff a `PN` packet will pause the machine, a second such packet will resume the machine.
* `R`,`L`: +Y, -Y direction flags. Set the direction flags for execution of the directional magnitude. This is set on the Y stepper motor chip. Suppresses X stepper chip.
* `B`,`T`: +X, -X direction flags. Set the direction flags for execution of the directional magnitude. This is set on the X stepper motor chip. Suppresses Y stepper chip.
* `M`: In compact mode, performs a 45° move in the direction of the last set direction flags. (Does nothing in default, R (+Y) gets the magnitude, doing `L<distance>T<distance>N` in default will do an angle in that mode)
* `D`,`U`: Laser On and Laser Off. Can be done in default or compact. Leaving or entering compact mode turns the laser off. When a G-raster step is invoked within compact, the laser is also disabled.
* `C`: Cut. Can be set in default mode (in any order at any point in default), but C overrides the G value, a  C value right after the V code causes the speed to be cut to 1/12th the typical value. Likely by using the command ticks rather than the ticks from the crystal. Without C set, all orthogonal movements are diagonal movements and the laser cannot be turned on.
* `G`: Raster_Step. Can be set in default mode. A single set G value sets the step amount for both directions. Two set G values eg, `G000G003` sets different step values for the other transitions.
* `V`: Speedcode. Differs by controller board, making some EGV files and commands not compatible since they differ with the board used. See Speedcode breakdown for specific. This is a long bunch of numbers that set several different things.
* `@`: Resets modes. Set all the set modes to the default values. Behaves strangely in default mode. Usually this is invoked while in compact mode, calling `@` which resets, then `N` which exits compact mode, then `SE` which sends the resets. The reset is issued in sequence so more commands can easily follow, setting new values and returning to compact mode. Since there's no way to tell where the system is in execution, you cannot follow this up with any `I` commands since that could destroy a expected set of commands. For this reason the final exit is usually finish which does eventually signal if the queue is empty with a PEMP flag.
* `F`: Finishes. In compact, requires we exit `N` then call `SE` to take effect. Then the device waits until all tasks are complete then signals a status with PEMP flag set, meaning the queue is empty this is usually 236.
* `N`: Executes in default mode, in compact mode, causes the mode to end. We can then issue rapid moves and return to compact mode with `S1E`. Without a reset, we could exit compact mode but without clearing the speed `V` or `G` or `C` values, thing could go weird.
* `S1`: Triggers Compact Mode.
* `S0P`: Homes then executes the commands. `IBzzzzzzzzRzzzzzzzS0P$` would home the machine then move to that location.
* `S1P`: Executes command, ignores rest of packet. Locks rail.
* `S2P`: Works like S1P but does not lock the rail.
* `PP`: If within the same packet, causes the device to rehome and all states are defaulted.
* `S2E`: Goes weird, but sometimes returns the device just to the left (might be rehomed device with an unlocked rail).
* `E`: Triggers F ability of compact mode. Without an E the F will not trigger. Without a second E after the F the mode will not engage. The F is held until the second E command.
* `S0`: Unknown. Seen in chinese software retrace feature: `IV2282554G000G001R|nS0B|nEaD|kUrDrU070DrU` (400mm/s). Usually homes the device when attempted manually.

Note: `IBTBTBTBTBTBTBTBTBTBTBTBTBTBBaS1P$` actually trigger the B and T for each of these events. This is determined by checking the signal processing on the board itself.

![nano](https://user-images.githubusercontent.com/3302478/75086288-62346180-54e7-11ea-92df-6642bcb3e2df.png)

### Speed Codes
The units in the Lhmicro-GL speed code have particular acceleration values that tend to give slightly different equations. Doing real-world physical timing the device speed finds the stated code values are 8.9% slower than the requested code. However, as is, these equations can be used to convert between values and speeds.

In suffix C notation the values are scaled down by a factor of 1/12th and sometimes the initial value changes. The typical formatting of the speed codes is `CV<speedcode>1C` where the appended C may or may not be present. Stepped down speeds only use acceleration value of 1.

The K40 Laser is controlled with a pair of stepper motors. Stepper motors work by moving 1 unit per tick. In this case the unit is 1/1000th of an inch. The speed a stepper motor travels is the result of the time between ticks. The smaller the time between ticks, the faster it moves. We are dealing with a 1000 dpi stepper motor, so, for example, to travel at 1 inch a second requires that the device tick at 1 kHz. This speed requires a delay 1 ms between ticks. The delay between ticks is controlled with a counter that counts the number of smaller ticks from the 21.1184 MHz processor until it reaches a threshold and sends the tick. This is the fundamental unit of the speedcode.

In the value for the speed code the value is subtracted from the maximum ticks of 65536. Since the processor treats negative number the same as positive numbers in binary. This is likely just a negative 16 bit number. For the diagonal correction value the encoded value is multiplied by the stepping and added to the count threshold. When going diagonal we requires some number of additional ticks, since we are technically travelling a longer distance.

#### Encoding
* For M1, M2, B1 & B2 boards, speed codes are in the form `CV<SpeedCode><Accel><Stepping><Diagonal Correction>C?`.
* for M, A & B boards, speed codes are in the form `CV<SpeedCode><Accel>` (as these later boards do not require diagonal corrections).

The *speed code* value is encoded as a 16 bit number which is broken up into two 3-digit ascii strings between 0-255. For example, a speed value of `36176` is `0x8D50` in hexadecimal - `0x8D` is `141` and `0x50` is `80`, so we encode this as `141` for the high bits and `80` for the low bits - so the resulting speed code is `141080`. It may seem weird to convey a 5 digit number encoded as 6 ASCII bytes, but keep in mind the processor does no work itself - it is flipping switches and with this encoding it doesn't need to do any time consuming mathematical calculations. 

The *stepping* value is a 3-digit ascii number between 0 and 128 which is used as a factor for the diagonal corrections.

The *diagonal correction* value is encoded in the same way as the speed code (two 3-digit ascii numbers, denoting a 16 bit value) and denotes an added delay amount that is appended to the orthogonal delay amount, when the device moves diagonally. 

#### Speed Equations
The gearing equations for calculating the millisecond delay between ticks (T) depends upon the version of the control board (hence needing to set this in the preferences): 

Once encoded they are then sent as `65536 - V`. 

|Board||Formula|
|-|-|-|
|A, B, B1|1|784 + 2000T
||2|784 + 2000T
||3|896 + 2000T
||4|1024 + 2000T
|B2|C|784 + 2020T
||1|784 + 24240T
||2|784 + 24240T
||3|896 + 24240T
||4|1024 + 24240T
|M, M1|1|5120 + 12120T
||2|5120 + 12120T
||3|5632 + 12120T
||4|6144 + 12120T
|M2|C|8 + 1010T
||1|5120 + 12120T
||2|5120 + 12120T
||3|5632 + 12120T
||4|6144 + 12120T

The initial values fall into concrete patterns:
* 2 * 8 * 7 * 7
* 2 * 8 * 8 * 7
* 2 * 8 * 8 * 8

and 

* 2 * 10 * 256
* 2 * 11 * 256
* 2 * 12 * 256

#### Speeds for acceleration

The typical breakdown of which acceleration/deceleration braking is used when depends on the speed requested. Some equations cannot logically denote some speeds. In some cases the value created will be negative. The Chinese software has a bug where it bitshift the value anyway and put it in the higher-bit value as 1677???? as a negative 24 bit number. These are not actually accepted by the boards as correct values. Using some gearing at some speeds cause the device to be louder. The Chinese software used the following speeds to determine the correct braking to use. However, it is possible to use a different braking for many other valid speeds.

* Non-Raster:
    * 1: [0 - 25.4]
    * 2: (25.4 - 60]
    * 3: (60 - 127)
    * 4: [127 - max)
* Raster:
    * 1: [0 - 25.4]
    * 2: (25.4 - 127]
    * 3: [127 - 320)
    * 4: [320 - max)

It is known however, that the chinese software uses a different brake value for x-stepping rasters than for y-stepping rasters. A speed of 128 in normal rastering gets a value of 3, but in x-step rastering gets a value of 4.

The optional suffix-C notation is used when values fall below a given minimum, this minimum value changes between board utilized.
* B2: [0 - 7)
* M2: [0 - 7)

The B2 board does not have valid values between [7 - 9.50853] as using the first equation there causes negative values.

The M and M1 boards do not have suffix-C notation and simply go negative below a certain point. In both cases it should be 5.0955mm/s.
* M: [0 - 6)
* M1: [0 - 6) or with raster [0 - 7)

The values for A, B, and B1 have a slope of 2000 and that can properly denote low values correctly and does not appear to use the C notation. The minimum value permitted with those values is 30Hz which is stated in an FAQ by the Chinese manufacturer to be the minimum allowed speed about 0.76 mm/s. The relationship here is still unknown.

#### Speedcode Values
The speedcode value is converted into two 3-digit ascii strings, but the value itself is derived from using a T value in terms of millisecond delay, in the correct gearing for the correct board. See the `Examples` section for examples.

#### Step Values
The step value only given when there are diagonal corrections used by the speedcode is between 0 and 128 and usually it is the rounded-up value of the speed as expressed in mm/s so if we are going 1 inch a second, that's 25.4 mm/s which would give us a step value of 26. Which would be encoded as `026`. The step value is used as a factor which divides the diagonal correction.

#### Diagonal Corrections
Since there is a horizontal and vertical stepper motor, they will, if given the same step-ticks, go at diagonal with the same delay as an orthogonal move. However this is a problem, a step of 1 mil horizontal and 1 mil vertical is actually a step of not 1 mil, but `sqrt(2)` mils. If both steps are made we have travelled 1.4142... mils rather than 1 mil. This means since laser's cut depth is a product of the time spend firing a lasering a particular point, diagonals will travel further over the same time and thus natively cut less deeply.

The speed codes add in a factor to correct for this in the M1, M2, B1, and B2 boards. This value adds in an additional delay when moving in a diagonal move. This value is added on top of the orthogonal delay value denoted by the speedcode.

This diagonal correction is calculated with the equation `diagonal_correction = R * -m * period_in_ms / S` where `R` is the ratio constant used and `m` is the slope in the gearing equation and `period_in_ms` is the amount of delay in the orthogonal value and `S` is the step value.

#### Speed Examples

If we want a speed of 0.5 inches a second, on an M2 board. That's is 12.7 mm/s which inside the first gear. So we are going to use the first gearing equation. We need 500 ticks a second, since we are moving 500 mils / second, so 0.5kHz gives us 2ms delay between ticks. Our use a T value is 2 ms so we plug that into the first gearing equation `5120 + 12120T` so `5120 + 12120 * 2`, `29360` which is subtracted from the max ticks of 65536 `65536 - 29360` meaning we should encode the value `36176`.

In hex, `36174` equals 0x8D50 and 0x8D is 141 and 0x50 is 80 So the resulting speed code is `141080` we used accel of `1` so we'd encode `1` for that value. With 1 set, it will not bother with accel/decel.

Since M2 boards use horizontal corrections, we have to calculate that.

We're going at 12.7 ms/s that'll give us a step value of 13 encoded as '013' since it's rounded up.

We need to calculate the value of the diagonal correction so use the equation `diagonal_correction = R * -m * period_in_ms / S` which is `0.4142 * 12120 * 2 / 13` or `772.32369...` which we'll call `772` which is 0x0304 in hex, 0x03 is 3 and 0x04 is 4 so this gets encoded as 003004 giving us a final speed code of:
`CV1410801013003004`. 

The value I used as the constant to calculate the horizontal correction here is `sqrt(2) - 1` which is how much longer a diagonal is from a orthogonal. The chinese software uses an automatic amount in the chinese software uses a ratio of `0.261199033289`, this minimizes the error for series of different angles being cut and thus different mixtures of diagonal and orthogonal cuts.

#### Known Errors
There are several known errors from the Chinese software which are duplicated and explained. The gaps in speed codes can be corrected by calling `validate_speed(mm_per_second, board, uses_raster_step=False)` with a proposed speed. This will return the nearest valid speed in mm_per_second for that particular Nano board, also since the negative values are what causes the error, a simple change is to make those values simply equal 0 rather than any negative value.

* Some speedcode values produced by Chinese software are negative. The boards cannot and does not validly permit negative values. The negative values produced by the chinese software are 24 bit numbers in the upper bits of the ascii string. These values will be reversed into speeds if converted that way. However the control boards do not accept these values.

* The value calculated for B2 at a speed of 9mm/s uses the equation `768 + 24240T`. `9mm/s` is `0.35433 in/s` (mm/25.4), thus we get a T value `(1/speed)` of `2.8222...` which gives us `768 + 68410.67` which is beyond the max 65536 delay. The calculated value in the Chinese software for a B2 board at `9mm/s` is invalid. This produces a speedcode `CV167772011821010006250` which is wrong. However a value of `2mm/s` for a B2 board produces a valid value with suffix-C notation. The B2 numbers become invalid after the end of suffix-C at `7mm/s` until the first gear equation becomes valid at `9.50853mm/s`. Extending the C notation above that limit may fix that problem.

* M and M1 boards do not have suffix-C notation (or no examples are produced by the chinese software) and the value zeroes out at 5.0955mm/s. So speed below this are not permitted, without becoming negative (which aren't allowed).

* The codes produced when using `G` harmonic movement at very low speeds where suffix-C notation would be required are not permitted. As such, if the software needs to use the suffix-C notation, the chinese software would try to do this without the suffix, the code is in error. This will result in significantly faster speeds than are actually than are requested. Since it doesn't apply the suffix-C notion but gives values as if it had.

### MeerK40t specific codes

Some codes are included in MeerK40t but are not part of Lhymicro-GL

These are:
* `\n`: carriage return means to pad the packet when you send it. Since it only processes complete packets.
* `-`: carryout wait sequence where you wait for the P_EMP flag from the system to say the codeblock is empty.
* `*`: abort. Everything that sees this should flush their buffers, and do not process anything already located in their buffers.
* `&`: resume the controller.
* `!`: pause the controller.

Several of these are used with regard to realtime commands which get promptly applied to the front of the buffer rather than the end. The reason for these is a paused controller can't resume if it doesn't process the data. Even an unpause command to the controller `PN` can't really work unless we are told that we should resume to read that code. So the resume code ends up being `PN&` and pause code is `PN!` which sends the PN to the controller but processes the & and ! as realtime commands to change its current state. The same is true for `*` which is used with `I` for emergency stop so `I*` sends the I to the controller but the * is processed to mean all the data already queued here should also be purged. 

### MeerK40t's EGV file format.
The EGV file format contains Lhymicro-GL preceded by some metadata as follows:

```
Document type : LHYMICRO-GL file
File version: 1.0.01`
Copyright: <copyright name>`
Creator-Software: <software creating file>`
 
%<x>%<y>%<width>%<height>%
<LHYMICRO-GL CODE>
```

The line starting with `%` provides the origin point and dimensions of the burn. Because the Lhmicro-GL code is relative to the start point, the X and Y in this line defines the start point, so you would issue a move command to that position and then start the code from there.

## USB Transmissions
### Basic USB Transmissions
USB data is written data typically in 2 packets of 34 bytes of data. An 0xA6 packet send command followed by 0x00 and the data, which is ended with 0x00 and the CRC code.

The CRC is onewire CRC coded data across the payload, bytes [2-32].

A packet may also consist of just 0xA0 (`HELLO` or mCh341_PARA_CMD_STS). When this happens the read will return six bytes of data. The second byte (Byte #1) contains the EPP pins of the data. 

* 206: `OK`. Everything is fine, we're ready.
* 238: `BUSY`. Cannot accept a packet just now.
* 207: `CRC_ERROR`. The CRC of that packet was wrong, and it was not processed.
* 236: `TASK_COMPLETE`. The 'F' command was sent and the stuff the K40 was doing has finished.
* 239: `POWER`. The board is just booting up or shutting off and doesn't have enough power.
* 204: `UNKNOWN_ERROR`. This can be produced by sending `FNSENSE` to the machine.

### LibUSB timeouts
Some experience with a lossy channel has lead to some revisiting of 207 as a CRC_ERROR the proper value is merely error. This happened while running the stock CH341 driver. So there are codes for 0,0 which may actually be LibUSB timeouts.

```
b'\x00iDaUhDaUhDaUiDaUhDaUiDaUhDaUiD\xb9'
[0, 0, 0, 0, 0, 0]
[255, 238, 0, 0, 0, 0]
[255, 238, 0, 0, 0, 0]
[255, 238, 0, 0, 0, 0]
[255, 238, 0, 0, 0, 0]
[255, 207, 0, 0, 0, 0]
```

The interesting thing here is that 207 happens after the 238 clears. So it's busy for a while, then it responds that there was an error. This ordering matters for some critical operations.

With proper resending on 207 there were a number of errors that cropped up within a noisy channel. 

Without resend a series of three rasters experienced:

First Raster:
```
b'\x00UdD|kUdDdU|kDdUdD072UhDdUhDdUd\x9c'
[0, 0, 0, 0, 0, 0]
[255, 207, 0, 0, 0, 0]
[255, 207, 0, 0, 0, 0]
b'\x00DdUdDdUdD|wUdD064U|cDdUdDtUdDp\xbe'
```

Second Raster:
```
b'\x00DdUdDdUdDxUhDdU|cDdU072DdUhDdUQ'
[0, 0, 0, 0, 0, 0]
[255, 207, 0, 0, 0, 0]
[255, 207, 0, 0, 0, 0]
b'\x00hDdUdDdUhDdUdDdUhDdUdDdUhDdUhDE'
```

Third Raster
```
b'\x00UdDhUdDlUdDtUdD|kUdDdUdDtUdDxU*'
[255, 207, 0, 0, 0, 0]
[255, 207, 0, 0, 0, 0]
b'\x00hDhUdDxUlD|cUdDlUdDhUdDtUdDhUd\xe9'
...
b'\x00080DdUdD|oUdDdUdD096U|wDdUdDtU\\'
[255, 207, 0, 0, 0, 0]
[255, 207, 0, 0, 0, 0]
b'\x00dD|kUdDhUdDlUdDxUdDdUdDtUdDxUd\x81'
```

And the first raster came out without issue, the second raster came out without issue, and the third raster had notable deficits. Shifting registration to the left two times. The error struck while going in the B direction (both times). This is means the 207 packet in those final two cases *should* have been resent. But The first packet appears to have lost about 312mils of an inch and second missed packet lost 375mils. The first packet's distances are `d h d l d t d y k d d d t d x` = 152. And the second packets distances are `80 96 d d y o d d d y w d d t` = 312. These seem accurate but do not quite match the correct values. Maybe something I am not considering with the packet being extracted from the recognized values.

However it is clear that dropping the packets on any 207 was a problem, never dropping the packet is also a problem. There is the notable difference that the status in the should be dropped packets is 0 (for the CH341 driver. But, pure 207 without any intermediate packets which may indicate that 207 means ERROR and in this case the error was the 0 packet. Whereas right after the send 207 means CRC error and the packet should be resent.

#### 204 error - Single Axis Sequence.
The 204 error is also seen in single axis testing.
e.g. IV2240983G000G001BS0RERzzzvLzzzvRzzzvLzzzvFNSE-$
* Will home
* Run a quick back and forth single axis test.
* Stop.
* But, the exit code is 204 for ending the wait.
* If you keep waiting this becomes 236
* If you send IPP$ this will respond with a 207 state.

### Challenge Code

Dimsum labs did some analysis, of their packet sniffed logs there are two exchanges which were not previously seen.

https://github.com/dimsumlabs/lasercutter/tree/master/protocol
```
0000   80 fd c6 22 04 88 ff ff 53 03 02 07 01 00 2d 00   ..."....S.....-.
0010   2a c8 89 58 00 00 00 00 f5 c2 0a 00 8d ff ff ff   *..X............
0020   22 00 00 00 22 00 00 00 00 00 00 00 00 00 00 00   "..."...........
0030   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0040   a6 00 41 a4 86 67 f9 8a 78 f9 f1 79 7d 44 f6 34   ..A..g..x..y}D.4
0050   13 0e a1 46 46 46 46 46 46 46 46 46 46 46 46 46   ...FFFFFFFFFFFFF
0060   a6 25                                             .%
```

The A6 at 0x40 means that this is sent via the EPP to the LHYchip. It still takes the zero, and send A followed by non-ascii. Filling the packet with F then the CRC. But, this isn't lhymicro-gl, since it clearly isn't ascii. It's clearly bulk-out to the device though. This likely makes the thing it said 'A' a4 86 67 f9 8a 78 f9 f1 79 7d 44 f6 34 13 0e a1 which after the A is 16 bytes.

And again:
```
0000   c0 00 ee 26 04 88 ff ff 53 03 02 07 01 00 2d 00   ...&....S.....-.
0010   29 c8 89 58 00 00 00 00 f9 6e 0d 00 8d ff ff ff   )..X.....n......
0020   22 00 00 00 22 00 00 00 00 00 00 00 00 00 00 00   "..."...........
0030   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00   ................
0040   a6 00 41 4b 30 46 46 46 46 46 46 46 46 46 46 46   ..AK0FFFFFFFFFFF
0050   46 46 46 46 46 46 46 46 46 46 46 46 46 46 46 46   FFFFFFFFFFFFFFFF
0060   a6 a4                                             ..
```

Sent the "AK0" like it was a command the device would know. Here K0 might be a command. Or it could be a different A command being executed on 4b 30.

### Challenge

The code on the board and the code send to the board are different.

```
52 84 04 7F 4F FB 4E 04 82 4A 2F D1 D1 F0 CD 62 --- pure zeros.
97 EF 50 CB 90 49 96 32 B3 50 2C E9 A7 52 1B 93 --- zeros ends with 1
```

This is the serial number as a challenge code. The serial number is written on the card so if your serial number is E8CF835E06EB918F then your hex challenge code is "c36be245d0a1c09ce6be2454ffb005b8" which is sent in pure byte binary after an A command.

This is the MD5 hash of the actual string in uppercase letters no spaces.

The M2 Nano has a serial number but it doesn't read it out, but rather checks whether that serial number is valid.

```
0000   a6 00 41 b0 7f 3b a1 75 2c df 58 83 1f 04 4d cb   ..A..;.u,.X...M.
0010   7c 64 65 46 46 46 46 46 46 46 46 46 46 46 46 46   |deFFFFFFFFFFFFF
0020   a6 15                                             ..
```

We get a response code 0xCC (204) if the code matches.

The software needs to send home "IPP" because there's no other method of getting out of 204.

## Raster Direction

If you raster horizontally putting a T or B value in the second directionality. Then any directionality that is not T or B when going into the turn around value either by switching T to B or by applying an NSE cycle will cause the X/Y directional change in the middle of accel/decel switch.

Rastering when done with a logical analyzer looks like this. Where 5 is the Y-axis and 0 is the directionality and 12 is the X axis and 10 the X-directionality.
![correct-direction-rastering](https://user-images.githubusercontent.com/3302478/146706162-2fa8dd96-ffe4-4383-b558-c2d2f67a2818.png)

Here the rastering is switched in the wrong directionality which causes half the twitch to apply to the Y and the other half to apply to the X. This causes the X axis to instantly be clunked up to speed and apply directionality wrongly which causes the position of the laser to be lost. As it cannot return to the same point correctly.
![bad-direction-rastering](https://user-images.githubusercontent.com/3302478/146706169-a26e182a-78c8-4cbd-8258-143e39f8bb9e.png)

## Acceleration Tick Count for Cut and Engrave.

Acceleration tick counts in cut/engrave: Acceleration 1 Step=512, 2=512, 3=1024, 4=1536 ... d=0.768"
Acceleration tick counts in raster: Acceleration 1 Step=512, 2=1024, 3=1536, 4=1792 ... d=0.896"

Four step ticks will move 1 mil distance. Acceleration and deceleration (braking) happen in the same amount of distance, so Accel 1-2 would be 512 steps / 4 = 128 * 2 (for both acceleration and braking) for a total of 256 mil distance for ramping up to speed and slowing down after.

Default speed is 380µs=18.2mm/s.

Raster step period to speed conversions (acceleration time):
70µs=100mm/s (57.333ms)
35µs=200mm/s (57.661ms)
24µs=300mm/s (57.325ms)
18µs=400mm/s (45.559ms)
14µs=500mm/s (45.593ms)
12µs=600mm/s (48.682ms)
11µs=625mm/s (47.848ms)
11µs=650mm/s (47.064ms)
10µs=675mm/s (46.441ms)
10µs=700mm/s

---

egv IBzzzzRzzzzS1P$ Period 91-92µs, Frequency 10.87-10.99kHz (for each stepper) since it’s diagonal.

691µs/1.45kHz=10mm/s
345µs/2.90kHz=20mm/s
230µs/4.35kHz=30mm/s
173µs/5.78kHz=40mm/s
138µs/7.25kHz=50mm/s
115µs/8.70kHz=60mm/s
99µs/10.10kHz=70mm/s
86µs/11.63kHz=80mm/s
77µs/12.99kHz=90mm/s
69µs/14.49kHz=100mm/s
63µs/15.87kHz=110mm/s
57µs/17.54kHz=120mm/s
53µs/18.87kHz=130mm/s
50µs/20kHz=140mm/s
46µs/21.74kHz=150mm/s

---

45.6µs 21.93kHz for the rapid. Which charts to 150 but the chart is in unfixed speeds which are off by 8% or so and should be multiplied by 0.919 to get the real speed. This means our 150 is 150 * 0.919 = 137.85 which is the same as the rapid speed going by the frequency of the ticks. 4 ticks per rotation.