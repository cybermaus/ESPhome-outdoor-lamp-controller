# Outdoor lamp controller for ESPHome

A small ESPHome controller for outdoor lamps: one relay, one wall input, and a daily cycle that follows the sun, a schedule, a light sensor, or any mix of them.

## Usage

### What it does

**It switches outdoor lamps on a sun based schedule.** Simple enough. 

<img width="695" height="134" alt="image" src="https://github.com/user-attachments/assets/c761939d-1417-4cd1-a9ae-41f5cd895fe9" />

It works **without central control** or even somewhat without stable network, for example when the lamp is placed far from the WiFi. It only needs to sync time once.
Each lamp calculates its own cycle. For the rest there is no reliance on connectivity, internet service, or any central controller like Home Assistant.
If WiFi, Home Assistant or the router goes away, the lamps keep switching on their own clock.

### Yet despite not relying on centralized control, multiple devices still work as one.
Lamps that share a group name configure and switch together,
and show up in Home Assistant. Change a setting on one lamp and the whole group is updated. Whichever lamp reaches a switching moment first drives
the group, and the others follow virtually instantly so they visually appear as a single wired lamp chain. A lamp reconnecting to bad WiFi asks
the group what is going on in case it missed any configuration updates. 

It also works far up north, where sunset is sometimes past midnight, and evening twilight merges into morning twilight. It should even work
above the polar circle, but that has only been tested in lab conditions. 

### Main controlling modes

| Switch | The lamp is driven by |
|---|---|
| Use schedule | One pair of on / off times |
| Use sun | Sun up and sun down, calculated locally |
| Connect daylight sensor | An external light sensor on the wall switch input |
| Connect motion sensor | An external motion sensor on the wall switch input |
| Sensor / Motion | Can also be set from Home Assistant or by any lamp in the group |

<img width="699" height="81" alt="image" src="https://github.com/user-attachments/assets/38df30f5-a241-4da8-931e-a1b01266cc29" />

<img width="698" height="85" alt="image" src="https://github.com/user-attachments/assets/af7da3b0-70c2-4d0e-8538-a03edb9d868b" />

They combine in the way you would expect:

- **Schedule only**: on from the on time to the off time.
- **Sun only**: on from sundown to sunup, the whole night.
- **Sun + schedule**: on from sundown until the off time (say 23:00), off during the night, and on again from the on time (say 05:30)
until sunup. A short, efficient lamp cycle that still lights the dark hours you care about.
- **Sensor on top of any of these**: a dark, overcast sky can turn the lamp on early or keep it on later.
- **Sensor only**: the lamp simply follows the sensor, and shares that with the group, only one need a sensor, and they all switch as one.
- **None**: the lamp only follows the wall switch (or Home Assistant) and drives the group to all switch as one.
- **Motion**: when motion is detected and it is dark, the lamp is on, even in the dead of night.

**Twilight level.** The Twilight level can be set: Normal (the official sunrise
and sunset), Half Civil, or Civil twilight (the lamp stays on longer).

**Optional wall switch.** Can be a wall switch, a daylight sensor, or a motion sensor. As wall switch, input acts a hotel or hallway
switch setup: every flip of any switch toggles the lamp and the whole group follows. Multiple locations can hold a wall switch.
Any manually switched state stays until the next sun or schedule cycle update. 

**Feedback info** Two text sensors show the next sun up and sun down, and the active lamp switch schedule.

### All settings

<img width="696" height="791" alt="image" src="https://github.com/user-attachments/assets/a595d86d-c08c-4e0d-aad9-479defa5204a" />

### Quick start

1. Copy `outdoor-lamp.yaml` and `common/diagnostics.yaml` and ensure you have a valid `secrets.yaml`.
2. Also copy the correct `common/board...` file or make one yourself for your specific hardware device.
3. Compile and Flash the device.
4. Optionally, adopt it in Home Assistant, and set the Device label and rename HA Friendly name to the same.<br>
   Even if you adopt in HA, it may be easier to configure the rest using the device Web UI
5. Set your location, timezone, and choose your cycle drivers.
6. For several lamps: give them the same Device Group. They find each other and update each other

## Detailed help

### Hardware

For now, only ESP8266, but soon will add ESP32, as I need those myself.
Select the correct `common/board...` file, or create one based on the example of the others. You need uncomment or add your
board to the main `outdoor-lamp.yaml`  in the `packages:` component

At the very least you need a device with a relay, so pretty much any ESP based smart switch will work. If you have no input
switch or LED, just assign any unused GPIO.

### Entities to set

There are no entities that you *must* set, other then your Location. Even the schedule has 'reasonable' time of days.
But advised is you set or consider at least these:

| Entity | What it does |
|---|---|
| Device Label | A short name for you, for example `Chalet14` or `Lamp03` |
| Device Group | Lamps with the same group name act as one |
| Use sun | Cycle follows the sun |
| Use schedule | Cycle follows the schedule times |
| Schedule on / Schedule off | A single pair of times of the schedule |
| Location Latitude / Longitude | Used for the sun calculation |
| Location Timezone | POSIX timezone text, see below |

### Entities that have the working state

These hold state information for HA, and can also be driven from HA or the user directly.
Any user change is also sent to the group

| Entity | What it does |
|---|---|
| Lamp | The Lamp state |
| Motion | Motion is detected (normally disabled) |
| Sensor | Daylight sensor active (it's dark) |

### Entities with some optional user settings or information

| Entity | What it does |
|---|---|
| Twilight level | Normal, Half Civil or Civil |
| Twilight Info | The next sun up and sun down times|
| Lamp Schedule | The resulting lamp on and off times |
| Connect daylight sensor | Switch input is a light sensor instead of a wall switch |
| Connect motion sensor | Switch input is a motion sensor instead of a wall switch |

### Entities to completely ignore

Unless you want to geek out.

| Entity | What it does or shows |
|---|---|
| _Debug Log | Set logging to DEBUG |
| _LED Brightness | Makes the LED, if any, less bright |
| _Group has peers | Device is not alone. A device that is alone is slightly hesitant before sending configuration out <br>To reset, change the group to a temp name and back |
| _Group synced | Config settings were exchanged with others, or if alone, assumed to be correct <br>To reset, change the group to a temp name and back |
| _<Diagnostics> | Lots of additional diagnostics, not sent to HA, so no network burden |

### Setting up location and time

Latitude and Longitude are with one decimal position, good enough for sun calculations.<br>
The timezone is a POSIX string, not a name. Examples:

| Place | Timezone text |
|---|---|
| Central Europe<br>(Amsterdam, Stockholm, Berlin) | `CET-1CEST,M3.5.0,M10.5.0/3` |
| Western Europe<br>(London, Dublin, Lisbon) | `GMT0BST,M3.5.0/1,M10.5.0` |
| Eastern Europe<br>(Helsinki, Bucharest) | `EET-2EEST,M3.5.0/3,M10.5.0/4` |
| UTC | `UTC0` |
| `blanks` | timezone and location are reset with example |

The time comes from Home Assistant or from NTP. If the clock is stale (without recent updates) the NTP rotates through the DHCP gateway and internet pool servers.
Without a valid clock the lamp cycle does nothing but the wall switch and any hardware input sensors keep working

**Sun levels.**

| Twilight level | Sun elevation |
|---|---|
| Normal | -0.833 degrees, the official sunrise and sunset |
| Half Civil | -3 degrees, roughly about 15 minutes extra daylight|
| Civil | -6 degrees, roughly about 30 minutes extra|

Check the actual info text for actual additional daylight, as the above "about 15 minutes" is in moderate conditions. Far up north near
midsummer or midwinter can add very large amounts of time, but should go to the same twilight threshold level. 

### Wall switch and sensors

The wall switch input can be used to connect a regular switch, a daylight sensor, or a motion sensor.<br>
*Sidenote: a daylight sensor is actually lack-of-daylight sensor, on when light is missing.*

Anyway, the motion sensor turns on the lamp, even in the dead of night when normally a lamp pause might exists.<br>
If there is no valid clock, motion sensor simply always turns on, even during the day. 

The daylight sensor however, only turns on the light a little earlier in the evening, or keeps it on little later in the morning.<br>
If there is no valid clock, it just works as a regular daylight sensor. 

No sensor input means the input is just a wall switch, it works as a hotel or hallway switch. Any switch action on any connected switch will toggle the light 
and the light state will stay until the next switch action, or the next calculated sun, schedule or motion action.

### Groups

Every lamp has a Device Group. Lamps in the same group keep their configuration in sync.

How it behaves:

- A configuration or state change on one lamp is sent to the entire group.
- The first lamp to reach a switching moment moves the entire group.
- An empty group name means no sending is even attempted. A stand-alone lamp. But a named group of only one lamp is also allowed. 
- Various mechanisms are implemented to avoid configuration storms or stale config to reappear.

### Home Assistant

The ESPHome device name itself is `outdoor-lamp-<mac>` so it is best to rename the device in Home Assistant to the Device Label, so you know which lamp is which.

Most configuration entities are disabled by default in Home Assistant. Enable the ones you need, however it
may be easier to leave them disabled and perform configuration on the Device Web UI. 
All parameters are used and shared among the group, regardless if they are enabled in HA or not. 

The lamps do not need HA, but they do report into HA for easy monitoring, but also so that you can drive HA automation from the lamps or from any lamp connected sensors.

Or in the other direction, HA automation can drive the group. So you can have a motion or daylight sensor not hardware connected to this group, and make HA set the relevant switch on
only one device so they will all respond. 

### Troubleshooting

| Problem | Check |
|---|---|
| The lamp never follows the cycle | Lamp Schedule says "No valid clock": check WiFi, NTP and Home Assistant time |
| Lamp Schedule says "Manual switch only" | Use sun and Use schedule are both off |
| Switching at the wrong time of day | Timezone text, latitude and longitude |
| Lamps do not follow each other | Same group name, same UDP port, network allows broadcast, router client isolation |
| New lamp does not get the group settings | Under some conditions, like after a suspected power failure, it waits 20 to 40 minutes to allow the rest of the network to come back. You can force it by changing the group name to a temporary name and back. |
| Lamp is on when the cycle says off | A manual switch or sensor input may last until the cycle next changes state |
| Wall switch does nothing | Connect daylight sensor or Connect motion sensor is on, so the input is read as one of those |
| Device lost from the network | See reboot/recover section below |

### Reboot and recovery

Because the clock is so important, unlike normal ESPHome, it does not reboot when it loses connectivity. If you simply wait for it to come back, you may wait a long time. But it does reboot... eventually

- If there *is* a valid clock, it does not reboot for 30 days! However it does restart the wifi radio every day at about 13h00 to potentially unstuck it and rescan for strongest AP. (the normal AP rescan is disabled for reasons of UDP packet stability)
- If there is a valid clock, and the clock is regularly resynced with NTP or HA, it never reboots.
- If there is *no* valid clock, *but* it needs one (if would not need one if only a wall switch input is configured), it will reboot every 6 hours

Additionally, if it does not find any WiFi, on wifi cycle or reboot, it will start the recovery access point, as is normal for ESPhome

Basically this means if you really lost the device, you may need to wait until next day 13h10 or even up 30 days unless if you manually power cycle.
However, during all this time, the device might still be faithfully switching the light from its internal clock and connected sensors. If it did not actually die completely. 

### Status

Early version. Expect changes.
