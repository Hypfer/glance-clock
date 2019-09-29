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
The Clock doesn't advertise itself as a regular pairable bluetooth device so getting it to pair with anything but the original app (e.g. a regular linux host) is a pain

### Time Synchronization
**HELP WANTED**
 
For Time Synchronization, the Mobile App opens up a BLE `Current Time Service` on the Phone which is periodically (?) polled by the clock.

While sending messages from a linux host was possible using `gatttool`, recreating this feature turns out much harder than expected.
Especially since there is basically no documentation on bluez whatsoever.

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

| Command                 	| Sequence   	| PayloadMessage 	| Description                                                                                                                	|
|-------------------------	|------------	|----------------	|----------------------------------------------------------------------------------------------------------------------------	|
| Settings                	| 5,0,0,0    	| Settings       	|                                                                                                                            	|
| Timer                   	| 3,0,0,0    	| Timer          	|                                                                                                                            	|
| TimerStop               	| 10,0,0,0,0 	| -              	|                                                                                                                            	|
| Alarm                   	| 4,47,0,0   	| ??             	|                                                                                                                            	|
| InfoUser                	| 50,0,0,0,0 	| -              	|                                                                                                                            	|
| AlarmClear              	| 21,0,0,0,0 	| -              	|                                                                                                                            	|
| AlarmStop               	| 20,0,0,0,0 	| -              	|                                                                                                                            	|
| CallScene               	| 6,83,0,103 	| Notice         	|  With modificator 129?? On call end, ScenesDelete with SceneId 103 gets executed                                           	|
| ScenesStop              	| 30,0,0,0,0 	| -              	|                                                                                                                            	|
| ScenesDelete            	| 33,0,0,i,0 	| -              	| i = sceneId?                                                                                                               	|
|                         	|            	|                	|                                                                                                                            	|
| OpenBrightnessConfView  	| 61         	| -              	|                                                                                                                            	|
| CloseBrightnessConfView 	| 60         	| -              	| Or general homescreen?                                                                                                     	|
| WeatherWatchfaceFoo     	| 35         	| -              	| ??                                                                                                                         	|
|                         	|            	|                	|                                                                                                                            	|
| Notify                  	| 2,i,0,j    	| Notice         	| Known i values  0 => delayed 48 => instant, ifttt 16 => instant, weather  Known j values  0 => default? ifttt 6 => Weather 	|

It seems to be possible to omit zeroes. At least sending 20 instead of 20,0,0,0,0 also works

#### Some constants
They definitely have some meaning. Some of them are the first byte of command sequences

| CUSTOM_SCENE_TYPE            | 0   |
|------------------------------|-----|
| CALENDAR_TYPE                | 1   |
| SCENE_PRIORITY_BAND_LOW      | 1   |
| SCENE_PRIORITY_SYSTEM_IDLE   | 1   |
| NOTIFY_TYPE                  | 2   |
| TIMER_TYPE                   | 3   |
| ALARM_TYPE                   | 4   |
| SETTINGS_TYPE                | 5   |
| CALLS_TYPE                   | 6   |
| TIMERS_CLEAR_TYPE            | 10  |
| SCENE_PRIORITY_EMUN_MAX      | 15  |
| SCENE_PRIORITY_BAND_MEDIUM   | 16  |
| ALARMS_STOP_TYPE             | 20  |
| ALARMS_CLEAR_TYPE            | 21  |
| SCENES_STOP_TYPE             | 30  |
| SCENES_START_TYPE            | 31  |
| SCENES_CLEAR_TYPE            | 32  |
| SCENE_PRIORITY_BAND_SYSTEM   | 32  |
| SCENES_DELETE_TYPE           | 33  |
| SCENE_PRIORITY_SYSTEM_MSG    | 33  |
| ANCS_START_TYPE              | 40  |
| ANCS_STOP_TYPE               | 41  |
| BONDS_CLEAR_TYPE             | 42  |
| HOMING_START_TYPE            | 43  |
| HOMING_CONFIRM_TYPE          | 44  |
| TASK_PRIORITY_TIMER          | 46  |
| TASK_PRIORITY_ALARM          | 47  |
| SCENE_PRIORITY_BAND_HIGH     | 48  |
| USER_INFO_CLEAR              | 50  |
| BRIGHTNESS_SCENE_STOP_TYPE   | 60  |
| BRIGHTNESS_SCENE_START_TYPE  | 61  |
| SCENE_PRIORITY_BAND_HIGHEST  | 64  |
| DSP_STATE_SHOW_TYPE          | 70  |
| SCENE_PRIORITY_BAND_CRITICAL | 80  |
| TASK_PRIORITY_TIMER_END      | 80  |
| TASK_PRIORITY_PWR_STATE      | 81  |
| TASK_PRIORITY_CALL           | 83  |
| TASK_PRIORITY_PIN_CODE       | 90  |
| TASK_PRIORITY_GREATINGS      | 255 |

### Icons
There also are some icon characters available which you can use in any TextData you like
Their range starts at charcode 128 (duh)


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

## Credits
The structure of this readme was plagiarized from [https://github.com/aprosvetova/xiaomi-kettle](https://github.com/aprosvetova/xiaomi-kettle)
