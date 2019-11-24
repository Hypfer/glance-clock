# GlanceClock BLE Protocol
![The Clock](https://i.imgur.com/fAZ4hvm.png)

## Overview
GlanceClock is a 22.8cm diameter wall clock with BLE connectivity.
It features a 32x8 LED Matrix Display as well as four 48-LED RGB Rings for text and other visualisations.

Its purpose is to act as an output device for notifications, weather information, appointments etc.

The device requires both a cloud account as well as an internet-connected smartphone to relay commands to the clock.

This repository aims to gather all information needed to build a cloudless bridge device which enables use of all Clock features using MQTT/REST/etc.

## Caveats
This is currently WIP. Not everything is fully understood yet. Help is much appreciated.

Also, there are two main issues:

### Pairing
The Clock doesn't advertise itself as a regular pairable bluetooth device so getting it to pair with anything but the original app
 (e.g. a regular linux host) is a pain.

### Time Synchronization
For Time Synchronization, the Mobile App opens up a BLE `Current Time Service` on the Phone which is polled by the clock on connect. (And periodically?)

To force the Clock to poll said Time Service, the Mobile App reads Characteristic `5075fc78-1e0e-11e7-93ae-92361f002671` of 
service `5075f606-1e0e-11e7-93ae-92361f002671` and then immediately disconnects from the clock, which then fetches the time.
Timing is of the essence here, since the Clock will only poll the time if the disconnect happens swiftly. (~100-200ms?)

## General
Communication is straightforward. The only requirement is to successfully pair (using PIN) with the Clock.

Then, all communication is done by reading and writing to characteristic `5075fb2e-1e0e-11e7-93ae-92361f002671` (`0x2901`) of service `5075f606-1e0e-11e7-93ae-92361f002671`.

There are other characteristics and services which are used for e.g. firmware updates. These haven't been documented yet.

Data serialization is done using protocol buffers. (See Glance.proto)

Reading `0x2901` returns a Settings message with the current settings.
Writing `0x2901` with a command + serialized payload executes said command.

### Example
`gatttool --sec-level=high -b CLOCK_MAC -t random --char-write-req -a 0x001f -n 023000002203120141` sends a `Notify` command with a serialized `Notice` message.
It will display the letter `A`, play the default notice sound `General_alert_1` as well as the default animation `Pulse` in the default color `Lime`

### Pairing with a Linux host
Using a fresh raspbian Buster image with Bluez 5.50, pairing is possible but requires good timing.

You will need two terminals. One with `bluetoothctl`, the other with `gatttool`
First, prepare everything in bluetoothctl:
```
power off
power on
agent on
default-agent
```
Then, open `gatttool` in interactive mode in the second terminal: `gatttool -b CLOCK_MAC -t random -l low -I`
In there, connect to the clock using `connect`.
Now, prepare the first terminal with the following statement but don't press enter yet. `pair CLOCK_MAC`

Switch to the second terminal, type `disconnect`, immediately switch back to the first terminal and press enter.
This way, you will not get `Device not available` which is caused by Caveat 1.

You _MUST_ get a pin prompt. If you didn't get that, the pairing process wasn't successful and you have to start again.
Type `remove CLOCK_MAC` and begin again from the start. It might take multiple attemts :(

### Known commands
_These are all in decimal_

| Command                   | Sequence   | PayloadMessage    | Description                                                                                     |
|---------------------------|------------|-------------------|-------------------------------------------------------------------------------------------------|
| CustomScene               | 0,0,i,j    | CustomScene       | i = Display mode 0 exclusive; 8 watchface; 24 ring & text j = scene slot                                                                                   |
| Notify                    | 2,i,0,j    | Notice            | i = scene_priority Known j values  0 => default? ifttt 6 => Weather                             |
| Timer                     | 3,0,0,0    | Timer             |                                                                                                 |
| Alarm                     | 4,47,0,0   | ??                |                                                                                                 |
| Settings                  | 5,0,0,0    | Settings          |                                                                                                 |
| CallScene                 | 6,83,0,103 | Notice            | With modificator 129?? On call end, ScenesDelete with SceneId 103 gets executed                 |
| SaveForecastScene         | 7,i,j,k    | ForecastScene     | i:? prio maybe? j: 8 => only ring, 16 => only text, 24 => ring & text k: scene slot             |
| SaveAppointmentsScene     | 8,0,i,7    | AppointmentsScene | i can be either 0 or 8. 0 is only alerts, 8 is alerts + watchface. 7 seems to be the scene slot |
|                           |            |                   |                                                                                                 |
| TimerStop                 | 10         | -                 |                                                                                                 |
|                           |            |                   |                                                                                                 |
| AlarmStop                 | 20         | -                 |                                                                                                 |
| AlarmClear                | 21         | -                 |                                                                                                 |
|                           |            |                   |                                                                                                 |
| ScenesStop                | 30         | -                 | Go to previous scene slot. Screen turns black for a moment                                      |
| ScenesStart               | 31         | -                 | Go to next scene slot                                                                           |
| ScenesClear               | 32         | -                 | Clear all scenes. Also hides Digital Clockface until new scenes arrive or settings are updated  |
| ScenesDelete              | 33,0,0,i   | -                 | i = scene slot                                                                                  |
| ScenesDeleteMany          | 34,0,i,j   | -                 | i = To Slot, j = From Slot                                                                      |
| UpdateAndRefresh          | 35         | -                 | Displays cloud update animation and refreshes the screen                                        |
|                           |            |                   |                                                                                                 |
| EnableAutomaticNightMode  | 40         | -                 |                                                                                                 |
| DisableAutomaticNightMode | 41         | -                 |                                                                                                 |
|                           |            |                   |                                                                                                 |
| ???                       | 41         | -                 | Seems to hide the watchface                                                                     |
| BondsClear                | 42         | -                 | Remove all paired devices                                                                       |
| StartCalibration          | 43         | -                 | Starts the calibration                                                                          |
| ConfirmCalibration        | 44         | -                 | Confirm the calibration                                                                         |
| ???                       | 45         | -                 | Some kind of alarm with notes on screen                                                         |
|                           |            |                   |                                                                                                 |
| ClearUserInfo             | 50,0,0,0,0 | -                 | Removes all paired devices and shuts off the clock.                                             |
| BrightnessSceneStop       | 60         | -                 |                                                                                                 |
| BrightnessSceneStart      | 61         | -                 |                                                                                                 |
| DSP_STATE_SHOW            | 70         | -                 | Displays a long number. Maybe serial?                                                           |

It seems to be possible to omit zeroes. At least sending 20 instead of 20,0,0,0,0 also works

### Constants

#### Types

| Name                        | ID |
|-----------------------------|----|
| CUSTOM_SCENE_TYPE           | 0  |
| CALENDAR_TYPE               | 1  |
| NOTIFY_TYPE                 | 2  |
| TIMER_TYPE                  | 3  |
| ALARM_TYPE                  | 4  |
| SETTINGS_TYPE               | 5  |
| CALLS_TYPE                  | 6  |
| TIMERS_CLEAR_TYPE           | 10 |
| ALARMS_STOP_TYPE            | 20 |
| ALARMS_CLEAR_TYPE           | 21 |
| SCENES_STOP_TYPE            | 30 |
| SCENES_START_TYPE           | 31 |
| SCENES_CLEAR_TYPE           | 32 |
| SCENES_DELETE_TYPE          | 33 |
| ANCS_START_TYPE             | 40 |
| ANCS_STOP_TYPE              | 41 |
| BONDS_CLEAR_TYPE            | 42 |
| HOMING_START_TYPE           | 43 |
| HOMING_CONFIRM_TYPE         | 44 |
| USER_INFO_CLEAR             | 50 |
| BRIGHTNESS_SCENE_STOP_TYPE  | 60 |
| BRIGHTNESS_SCENE_START_TYPE | 61 |
| DSP_STATE_SHOW_TYPE         | 70 |

#### Task Priorities

| Name                    | ID  |
|-------------------------|-----|
| TASK_PRIORITY_TIMER     | 46  |
| TASK_PRIORITY_ALARM     | 47  |
| TASK_PRIORITY_TIMER_END | 80  |
| TASK_PRIORITY_PWR_STATE | 81  |
| TASK_PRIORITY_CALL      | 83  |
| TASK_PRIORITY_PIN_CODE  | 90  |
| TASK_PRIORITY_GREATINGS | 255 |

#### Scene Priorities

| Name                         | ID |
|------------------------------|----|
| SCENE_PRIORITY_BAND_LOW      | 1  |
| SCENE_PRIORITY_SYSTEM_IDLE   | 1  |
| SCENE_PRIORITY_EMUN_MAX      | 15 |
| SCENE_PRIORITY_BAND_MEDIUM   | 16 |
| SCENE_PRIORITY_BAND_SYSTEM   | 32 |
| SCENE_PRIORITY_SYSTEM_MSG    | 33 |
| SCENE_PRIORITY_BAND_HIGH     | 48 |
| SCENE_PRIORITY_BAND_HIGHEST  | 64 |
| SCENE_PRIORITY_BAND_CRITICAL | 80 |

### Icons
There are icon characters available which you can use in any TextData you like

| Charcode 	| Icon                                	|
|----------	|-------------------------------------	|
| 128      	| House                               	|
| 129      	| Phone                               	|
| 130      	| Clock                               	|
| 131      	| Plug                                	|
| 132      	| Smartphone                          	|
| 133      	| Star/Sun/Bright                     	|
| 134      	| Smiley                              	|
| 135      	| A huge nearly-all pixel white block 	|
| 136      	| Nothing                             	|
| 137      	| 0/3 Battery                         	|
| 138      	| 1/3 Battery                         	|
| 139      	| 2/3 Battery                         	|
| 140      	| 3/3 Battery                         	|
| 141      	| Bell with diagonal line             	|
| 142      	| Speaker off                         	|
| 143      	| Thermometer                         	|
| 144      	| Droplet                             	|
| 145      	| Heart                               	|
| 146      	| Barometer (?)                       	|
| 147      	| Pipe with arrow left                	|
| 148      	| Arrow right with pipe               	|
| 149      	| Umbrella                            	|
| 150      	| Bell                                	|
| 151      	| Speaker                             	|
| 152      	| Wind speed                          	|
| 153      	| Cloud                               	|
| 154      	| Missing Character                   	|
| 176      	| °                                   	|

## Factory reset
If something goes horribly wrong, push and hold the reset button + Power button.
Let go of the reset button and keep holding the Power button until the LED blinking pattern changes.
Then release it as well.

Please note that this will also reset the firmware back to the factory version.

## Misc
You can find firmware images, the official website documentation and more [here.](https://github.com/Hypfer/glance-clock-assets)

It is possible and recommended to flash Firmware ZIP files using the `nRF Toolbox App`.

### Night Mode
Taken from the offical Website + App Version 2.0.1:
```
Night mode activates a digital clock that you can see at night when the hands are not visible.
In Night mode all integrations that set up As a clock Face (ACF) are disappeared, the sound is muted, but notifications will still be displayed.

When it's dark to see the clock hands, the digital time will be shown on the clock face automatically. Uses internal ambient light sensor built into the clock 
```

### Other random pieces of information
`When you perform calibration hands don't reach 12:00 when there is a green segment, but stop at 11:59. Please take off the minute hand and place it back to make hands straight at 12:00. Probably the hands were mechanically misaligned a bit. `

Concerning what is most likely command `35`:
`Yes, we have added this icon to show that Clock is doing something and responding to changes. Otherwise, if you change an As Clock Face you may not see anything for 10-15 seconds before the face is changed.`

## Credits
The structure of this readme was plagiarized from [https://github.com/aprosvetova/xiaomi-kettle](https://github.com/aprosvetova/xiaomi-kettle)

The weather watchface messages were analyzed using [https://github.com/jmendeth/protobuf-inspector/](https://github.com/jmendeth/protobuf-inspector/)