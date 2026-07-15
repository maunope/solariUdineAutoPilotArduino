# Legacy
I'm stopping development on this board, I'll build a new, ESP32 based one with the following goals:
- Perfoboard friendly, minimal board build effort, ideally just the ESP32, L293D and DC-DC converter.
- WiFi connected, to allow NTP alignment and web UI configuration


<p align="center"><img src="cifra12.png" width="320px" alt="Solari Udine Cifra 12"/></p>


# Disclaimer
This is still a prototype. The circuit design was built and tested on a perfboard; the KiCad board design was never tested. Use at your own risk and please **don't come whining if you damage your flip clock or set your house on fire.**


# Solari Udine Auto Pilot


Arduino-based Solari Udine clock controller, suitable for alternating-pulse Solari Udine clock motors (i.e. Cifra 12). Controls hours and minutes; no calendar functions are provided. The current board design accepts a single Vcc in, either 12V or 24V, from an external power source. 


**Main features:**   
- Self adjusts after a power loss
- Handles DST (Central Europe rules)
- Allows manual adjustment
 

An LED provides visual feedback; an optional LCD1602 display can be plugged in to show the status.


The software tries to mitigate strain on the flip clock by limiting the number and frequency of roller turns:
- Turns are limited to 1 every 3 seconds (manual step adjustments bypass this).
- If the clock is more than 120 minutes behind, movement is paused until the next day (a halted clock displays the right time once a day after all **:-)**).
- If the EEPROM or RTC module fails, no pulses are sent to the motor.
- If the RTC module crashes, attempts are made to restore the last known consistent timestamp.
- A hardware circuit limits pulse duration and frequency to 1 second.
- The circuit board has hardware protection to avoid prolonged motor pulses, limited to 1 second. Once a pulse is sent, following ones are delayed until the hardware protection has completed its cycle. Note that it is totally possible to build the board without the 555 timer; just hardwire the L293D enable pins to HIGH, and the software will work just fine.

The following features improve durability and operation:
- The system self-restarts once per week.
- EEPROM writes are spread over a 256kbit EEPROM. This should guarantee at least 20 years of operation before writes start to fail (more likely 50-70 years).
- If the EEPROM stored time is unavailable, the clock halts until it is adjusted.
- If the RTC module is unavailable, the clock blinks for 30 seconds and then resets.
- The Arduino will stay powered down approximately 95% of the time during regular operation, saving considerable power. As a side effect, minutes will flip within +/- 2 seconds of when the RTC time ticks a minute.  


## Dependencies


- [LiquidCrystal I2C](https://github.com/johnrickman/LiquidCrystal_I2C)
- [RTClib](https://github.com/adafruit/RTClib)
- [SparkFun External EEPROM Arduino Library](https://github.com/sparkfun/SparkFun_External_EEPROM_Arduino_Library)
- [Rocket Scream Low Power](https://github.com/rocketscream/Low-Power)


## Circuit Board


**WORK IN PROGRESS:** Check the schematic in this repo. I've built and tested it on perfboard; the board design might not be 100% correct, and the layout could definitely 
use some improvement.

<img src="controllersolari.png" width="640px" alt="Solari Udine Autopilot Circuit Board">

The **"Simplified"** board version uses some 30% fewer components by removing the following features:

- Hardware limitation of pulses duration
- Paralleled L293D outputs

Considering how unlikely it is for the sketch to remain stuck with motor output set high, you might want to save yourself some hassle, especially if you're building on perfboard.

I've built the board using these cheap DS3231 modules bought off Amazon. The design uses a male connector matching their Vcc-signal-signal-NC-Ground pinout; you might want to use something different and change the board design accordingly.
One good thing about these is that they're small enough and can be easily swapped out for testing and debugging.

<img src="rtcmodule.png" width="240px" alt="RTC Module">


**How to read led signals**


Either for good or bad news, the green LED will flash. The red LED will stay on when the general enable switch is on.


<img src="feedbackledpulses.png" width="640px" alt="Feedback led pulses interpretation card">


## How to install and adjust


**First things first: let's not fry stuff.**
- If you built the board yourself, either run the Arduino sketch with the `SET_COMPILE_TIME_TO_RTC` flag turned on, or use serial commands to prime the RTC module.
- Check your clock's coil working voltage. Most accept 24V plus 12V or 48V with different wiring. **The current board design was tested with a 12V external power source; 24V should work fine albeit with the voltage regulator running hotter; 48V is beyond the regulator specs.**
- Check your coil's max current draw. One L293D with paralleled channels can drive up to 1200mA; no Solari Udine clock should draw more than that.
- Unplug the clock, wire your power supply and the clock motor coil
- Turn the enable switch *OFF*
- Power the board.
 **No smoke? Good.**
- On the first start, a manual time adjustment is required. A "Time Adjustment Required" pattern will flash on the feedback LED.
- Manually adjust the clock flip rolls to match the current time; the optional LCD displays it on the first line.
- Dial at least a one-minute advance with the button; this will prime the pulse direction.
- When the clock time matches the RTC time, *press the push button for 3 seconds until the "Manual Time Adjustment Saved" pattern flashes.*
- Turn the enable switch *ON*


The controller will now keep the flip clock display aligned with the internal RTC clock.


## Enabling motor movement

A switch allows pausing motor movement for maintenance and dialing in commands; the clock will catch up automatically once enabled again.
An LED will remain **ON** while movement is paused.
The switch also disables deep sleep time and prepares the Arduino for accepting commands: wait until the feedback LED blinks once after pausing movement, then use the push button or serial commands.

## Push button commands

*Important:* Only use push button commands with the motor enable switch turned off. 

- Brief press: Advances the clock 1 minute, the LED blinks briefly.
- Three-second press: Aligns EEPROM time with RTC time for manual adjustment, and the LED blinks in a pattern.

## Serial port commands

*Important:* Only issue serial commands with the motor enable switch turned off.

Commands format: **(<<|>>)[A-Z]{1,10}[a-z,A-Z,0-9]{1,20}**

- **<<** to read info, **>>** to set
- command name, in caps
- (where applicable) parameter

One command per line, max 32 chars long. The parser is pretty crude, please stick to tested commands ;-) 

**Available commands**:

- **<<BOOTTIMESTAMP** prints the current boot timestamp.
- **>>COMPILEDATETIME** aligns the RTC date and time to the sketch compilation timestamp.
- **<<COMPILEDATETIME** prints the sketch build timestamp.
- **>>DAILYSECONDSOFFSET[+-][0-9]{3}$** (i.e., >>DAILYSECONDSOFFSET+010, >>DAILYSECONDSOFFSET+000, >>DAILYSECONDSOFFSET-020) stores the desired number of daily RTC error compensation seconds to the EEPROM.
- **<<DAILYSECONDSOFFSET** prints the daily seconds of RTC error correction setting stored in the EEPROM.
- **>>RTCDATETIMEyyyymmddhhmmss** (i.e., >>RTCDATETIME20241119235959) sets the RTC date and time. It takes standard time, *not DST*.
- **<<RTCDATETIME** prints the RTC date and time.
- **<<EEPROMDATA** prints EEPROM date, time, and clock status information.




## Build flags

- `DEBUG_MODE` enables serial port and LCD debug messages, disables sleep time, and forces reinitialization of the EEPROM and RTC module on each read.
- `SET_COMPILE_TIME_TO_RTC` forces writing the sketch build time to the RTC on each boot. Only use this to program new RTC modules.


## Troubleshooting
**Q:** Instead of catching up, the clock is standing still and the "Paused until next day" pattern flashes.


**A:** When more than 120 minutes need to be caught up, the automatic adjustment will pause and wait for the clock time to be correct again the next day.


**Q:** The Arduino keeps resetting every half a minute or so, the feedback led flashes


**A:**  Either the EEPROM or RTC module isn't responding; the board needs fixing.


**Q:**  I've hooked up a display, now what?


**A:** Note that the display only works when DEBUG_MODE flag is raised


<img src="displaycodes.png" width="640px" alt="Optional display fields">


**Q:**  How can I tell if my eeprom is corrupted or simply requires initialization?

**A:**  "Adj. time" on the display or "No valid time set in eeprom, please adjust clock" on the Serial are good news.


**Q:** My RTC module lost its time configuration!

**A:** >>DATETIME[0-9]+ command should fix it, however, if it happens again, the module batttery might be drained

**Q:** I can't tell what's going on with eeprom data

**A:** The <<EEPROMDATA command extracts all the info saved on the EEPROM.

**Q:** LCD doesn't display anything, and/or serial messages are missing

**A:** Set the DEBUG_MODE build flag and upload the firmware again.

**Q:** **(serial|button) commands** are not working!

**A:** They only work reliably when motor movement is paused (solid red LED). During regular operation, the Arduino is awake and responding to commands for only a few seconds per minute. Good luck catching it awake! :-) 

**Q:** My clock drifts X seconds ahead/behind every day

**A:** Buy a proper RTC module next time! :-) There's a handy feature to correct this if the drift is the same every day; see serial commands.

**Q:** My clock drifts X *minutes* ahead/behind every day

**A:** Yikes! That's not OK! Try a different RTC module, then check your board for errors on the I2C bus pullup resistors and +5V power. I've tested the board design by powering I2C devices straight from a voltage regulator, bypassing the Arduino's internal one. Either do the same, or dig into the sleep configurations to figure out how to keep the RTC powered during sleep. If you figure something out, open a PR here :-)

**Q:** I got an RTC failure message/blink and then the board rebooted

**A:** Your RTC went nuts! It happens every now and then with cheap *ss modules; my MTBF is around 4 months. The sketch tries its best to recover to the last timestamp that made sense if it detects the RTC is acting up.

**Q:** No matter what I do, the clock keeps flipping!

**A:** The RTC module is returning wrong timestamps, and the checks I've put in place are not working. :-( 
Compile the sketch in debug mode, use serial commands to check what timestamp is recorded in the RTC module, and fix it. 
Realign the EEPROM time with the RTC time, reboot, and check after a few minutes that the RTC module is behaving. If you wish, drop me an email about the problem including the timestamp you had in the RTC module and the one in your EEPROM (or, even better, figure out what's wrong and open a PR :-) ).


**Q:** That PCB design is **lame**!

**A:** It is. Any help is much appreciated! ;-)

## Todo
- When disabling motion while sleep is skipped (i.e. 2 secs after a pulse), the feedback LED doesn't blink.
- Add support for Bluetooth communication.
- Improve KiCad design.
- Figure out some predictable behavior for when the RTC is known to be f*cked up, i.e., start from 1970 and stop calculating DST.


