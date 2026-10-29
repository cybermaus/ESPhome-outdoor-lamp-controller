# Outdoor lamp controller for ESPHome

A small ESPHome controller for outdoor lamps: one relay, one wall input, and a daily cycle that follows the sun, a schedule, a light sensor, or any mix of them.

## Usage

### What it does

**It switches outdoor lamps on a sun based schedule.** Simple enough. 

<img width="695" height="134" alt="image" src="https://github.com/user-attachments/assets/c761939d-1417-4cd1-a9ae-41f5cd895fe9" />

It works **without central control** or even somewhat without stable network. It only needs to sync time once, for example when the lamp is placed far from the WiFi.
Each lamp calculates its own cycle. For the rest there is no reliance on connectivity, internet service, or any central controller like Home Assistant.
If WiFi, Home Assistant or the router goes away, the lamps keep switching on their own clock.

### Yet despite not relying on centralized control, multiple devices still work as one.
Lamps that share a group name configure and switch together,
and show up in Home Assistant. Change a setting on one lamp and the whole group is updated. Whichever lamp reaches a switching moment first drives
the group, and the others follow virtually instantly so they visually appear as a single wired lamp chain. A lamp rebooting or reconnecting asks
the group what is going on in case it missed any configuration updates. 

It also works far up north, where sunset is sometimes part midnight, and evening twilight merges into morning twilight. It should even work
above the polar cicle, but that has only been tested in lab conditions. 

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

**Works far north.** Near midsummer dusk and dawn can fall around midnight, or the sun may not set at all. 
The lamp follows the actual position of the sun.

**Optional wall switch.** Can be a wall switch, a daylight sensor, or a motion sensor. As wall switch, input acts a hotel or hallway
switch setup: every flip of any switch toggles the lamp and the whole group follows. Multiple locations can hold a wall switch.
Any manually switched state stays until the next sun or schedule cycle update. 

**Feedback info** Two text sensors show the next sun up and sun down, and the resulting lamp switch schedule.

### All settings

<img width="696" height="791" alt="image" src="https://github.com/user-attachments/assets/a595d86d-c08c-4e0d-aad9-479defa5204a" />

### Quick start

1. Copy `outdoor-lamp.yaml` and add `wifi_ssid`, `wifi_password` and `ota_password` to your `secrets.yaml`.
2. Check the pins in the `substitutions` block against your board.
3. Compile and Flash it.
4. Optionally, adopt it in Home Assistant, and set the Device label and rename HA Friendly name to the same.<br>
   Even if you adopt in HA, it may be easier to configure using the device Web UI
5. Set your location, timezone, and choose your cycle drivers.
6. For several lamps: give them the same Device Group. They find each other and update each other

## Detailed help

### Hardware

Unless if you have a classic (v1 though v3) Shelly1, you need to modify at least these 
values in the substitutions at the top of the code. 
| Function | Default pin | Notes |
|---|---|---|
| Relay | GPIO4 | Shelly 1 not inverted|
| Wall input | GPIO5 | Shelly 1 no pull-up not inverted|
| Status LED | GPIO2 | Blinks every 10 seconds unused for Shelly 1 |

### Entities

| Entity | What it does |
|---|---|
| Lamp | The Lamp state |
| Lamp Schedule | The resulting on and off times |
| Device Label | A short name for you, for example `Chalet14` or `Lamp03` |
| Device Group | Lamps with the same group name act as one |
| Use sun | Cycle follows the sun |
| Use schedule | Cycle follows the schedule times |
| Twilight level | Normal, Half Civil or Civil |
| Twilight Info | The next sun up and sun down |
| Schedule on / Schedule off | The pair of times of the schedule |
| Location Latitude / Longitude | Used for the sun calculation |
| Location Timezone | POSIX timezone text |
| Motion | Motion is detected |
| Sensor | Daylight sensor active (it's dark) |
| Connect daylight sensor | Switch input is a light sensor instead of a wall switch |
| Connect motion sensor | Switch input is a motion sensor instead of a wall switch |

Most configuration entities are disabled by default in Home Assistant. Enable the ones you need, however it
may be easier to leave them disabled and perform configuration on the Device Web UI. 
All parameters are used and shared among the group, regardless if they are enabled in HA or not. 

### Setting up location and time

Latitude and Longitude as one decimal values, good enough for sun calculations.<br>
The timezone is a POSIX string, not a name. Examples:

| Place | Timezone text |
|---|---|
| Central Europe<br>(Amsterdam, Stockholm, Berlin) | `CET-1CEST,M3.5.0,M10.5.0/3` |
| UK & Ireland | `GMT0BST,M3.5.0/1,M10.5.0` |
| Finland, Romania | `EET-2EEST,M3.5.0/3,M10.5.0/4` |
| UTC | `UTC0` |
| `blanks` | timezone and location are reset with example |

The time comes from Home Assistant or from NTP. On every WiFi connect the lamp tries the DHCP gateway as NTP server
first, and a public NTP pool server second, no addresses are hardcoded. Without a valid clock the cycle does nothing
but the wall switch and any hardware sensors keep working

### The cycle in detail

| Use sun | Use schedule | Sensor | Result |
|---|---|---|---|
| On | Off | Off | On while the sun is down, the whole night |
| Off | On | Off | On from Schedule on to Schedule off |
| On | On | Off | On while the sun is down, except between Schedule off and Schedule on |
| Any | Any | On | On dark days, turn on a bit earlier, turn off a bit later, as driven by the sensor. |
| On | On | On | like above, but do not keep on at night. |
| Off | Off | On | Lamp follows the sensor |
| Off | Off | Off | No cycle, manual only |

**The schedule is simple on purpose.** One pair of times, every day. No weekends, no late Saturdays.

**Sun levels.**

| Twilight level | Sun elevation |
|---|---|
| Normal | -0.833 degrees, the official sunrise and sunset |
| Half Civil | -3 degrees, adds about 15 minutes of daylight|
| Civil | -6 degrees, adds about 30 minutes |

Check the actual info text for actual times, as the above "about 15 minutes" is in moderate conditions. But far up north near
midsummer can add larger periods, even infinite time (sun never sets or rises)

### Wall switch and sensors

The wall switch input can be used to connect a regular switch, a daylight sensor, or a motion sensor.<br>
*Sidenote: a daylight sensor is actually lack-of-daylight sensor, on when light is missing.*

Anyway, the motion sensor turns on the lamp, even in the dead of night when normally a light pause might exists.<br>
If there is no valid clock, it might even turn on the light during daytime

The daylight sensor however, only turns on the light a little earlier in the evening, or keeps it on little later in the morning.<br>
If there is no valid clock, it just works as a regular daylight sensor. 

If it is just a wall switch, it works as a hotel or hallway switch. Any switch action on any connected switch will toggle the light<br>
and the light will stay until the next switch action, or the next calculated sun and schedule switch action.

### Groups

Every lamp has a Device Group. Lamps in the same group, on the same network, keep all but local configuration in sync.
Local configuration are the Device Group, Device Label, Connect Sensor and Connect Motion.

How it behaves:

- A change on one lamp is sent to the entire group.
- The group name cannot contain a colon.
- The first lamp to reach a switching moment moves the group.
- An empty group name means no sending is even attempted. A stand-alone lamp. But a named group of only one lamp is also allowed. 
- Various mechanisms are implemented to avoid configuration storms or stale config to reappear.

### Home Assistant

The ESPHome device name itself is `outdoor-lamp-<mac>` so it is best to rename the device in Home Assistant to the Device Label, so you know which lamp is which.

The lamps do not need HA, but they do report into HA for easy monitoring, but also so that you can drive HA automation from the lamps or from any lamp connected sensors.

Or in the other direction, HA automation can drive the group. So you can have a motion or daylight sensor not hardware connected to this group, and make HA set the relevant switch on
only one device so they will all respond. 

### Troubleshooting

| Problem | Check |
|---|---|
| The lamp never follows the cycle | Lamp Schedule says "No valid clock": check WiFi, NTP and Home Assistant time |
| Lamp Schedule says "Manual switch only" | Use sun, Use schedule are off |
| Switching at the wrong time of day | Timezone text, latitude and longitude |
| Lamps do not follow each other | Same group name, same UDP port and addresses, network allows broadcast, router client isolation |
| New lamp does not get the group settings | It asks again when WiFi reconnects, or when you change its group name |
| Lamp is on when the cycle says off | A manual or group change lasts until the cycle next changes state |
| Wall switch does nothing | Use daylight sensor is on, so the input is read as a sensor |
| Device lost from the network | See reboot/recover section below |

### Reboot and recovery

Because the clock is so important, unlike normal ESPHome, it does not reboot when it looses connectivity. If you simply wait for it to come back, you may wait a long time. But it does reboot... eventually

If there is a valid clock, it does not reboot for 7 days! However if does did not see any HA or NTP time sync, it will restart WiFi every 6 hours with a 10 minute pause.
Eventually after 7 days without time sync, it gives up and reboots. Which means it looses the clock. Hopefully the full stack reset will unstuck something and it finally reconnects.

If there is no valid clock, *and* it needs one (if would not need one if only a wall switch is configured), it will reboot every 6 hours

Basically this means you may need to wait 7 days unless if you manually power cycle. It also means that if the WiFi access point itself fails, this device will continue to work for 7 days, but then it will reboot, loose time, and still not find the AP. If you do not want that, remove the 7 day reboot from the code, it will restart WiFi every 6 hours forever.

If it does finally reboot, maybe triggered by a manual power cycle, it will start a recovery access point, as is normal for ESPhome.

### Status

Early version. Expect changes.
