# Outdoor lamp controller for ESPHome

A small ESPHome controller for outdoor lamps: one relay, one wall input, and a daily cycle that follows the sun, a schedule, a light sensor, or any mix of them.

## Usage

### What it does

**It switches outdoor lamps on a sun based schedule.** Simple enough. 

It works with or **without stable network.** It only needs to sync time once, for example when the lamp is placed far from the WiFi. Each lamp calculates its own cycle. For the rest there is no reliance on connectivity, internet service, or any central controller like Home Assistant. If WiFi, Home Assistant or the router goes away, the lamps keep switching right up to the next power cycle when they loose the clock. 

Yet despite not relying on centralized control, **multiple devices still work as one.** Lamps that share a group name configure and switch together, and show up in Home Assistant. Change a setting on one lamp and the whole group is updated. Whichever lamp reaches a switching moment first drives the group, and the others follow virtually instantly so they visually appear as a single wired lamp chain. A lamp rebooting or reconnecting asks the group what is going on in case it missed any configuration updates. 

### Main controlling modes

| Switch | The lamp is driven by |
|---|---|
| Use sun | Sun up and sun down, calculated locally |
| Use schedule | One pair of on / off times |
| Use daylight sensor | An external light sensor on the wall input |

<img width="702" height="121" alt="image" src="https://github.com/user-attachments/assets/5af03ed3-9612-4dd1-9887-a7aff4ce18fd" />

They combine in the way you would expect:

- **Sun only**: on from sundown to sunup, the whole night.
- **Schedule only**: on from the on time to the off time.
- **Sun + schedule**: on from sundown until the off time (say 23:00), off during the night, and on again from the on time (say 05:30) until sunup. A short, efficient lamp cycle that still lights the dark hours you care about.
- **Sensor on top of any of these**: a dark, overcast sky can turn the lamp on early or keep it on later.
- **Sensor only**: the lamp simply follows the sensor, and shares that with the group, only one need a sensor, and they all switch as one.
- **None**: the lamp only follows the wall switch (or Home Assistant) and drives the group to all switch as one.

**Twilight level.** The Twilight level select decides what counts as sun up and down: Normal (the official sunrise and sunset), Half Civil, or Civil twilight(the lamp stays on longer).

**Works far north.** Dusk and dawn near midsummer can fall around midnight, or the sun may not set at all. The lamp follows the actual position of the sun, I think all edge cases are accounted for. .

**Optional wall switch.** Without the sensor, the input acts as wall switch in a hotel or hallway switch setup: every flip of toggles the lamp, and the whole group follows. Multiple locations can hold a switch. Any manually switched state stays until the next cycle update. 

**Feedback info** Two text sensors show the next sun up and sun down, and the resulting lamp switch times.

### All settings

<img width="700" height="677" alt="image" src="https://github.com/user-attachments/assets/9c755008-fd48-4ea6-a985-16a26015acf0" />

### Quick start

1. Copy `outdoor-lamp.yaml` and add `wifi_ssid`, `wifi_password` and `ota_password` to your `secrets.yaml`.
2. Check the pins in the `substitutions` block against your board.
3. Compile and Flash it.
4. Optionally, adopt it in Home Assistant, and set the Device label and rename HA Friendly name to the same.<br>
   Even if you adopt in HA, it may be easiest to configure using the device Web UI
6. Set your location, timezone, and choose your cycle drivers.
7. For several lamps: give them the same Device Group. They find each other and update each other

## Detailed help

### Hardware

| Function | Default pin | Notes |
|---|---|---|
| Relay | GPIO12 | Set `inverted: true` on the relay output if your board is active low |
| Wall input | GPIO13 | Switch to GND, internal pull-up |
| Status LED | GPIO2 | Onboard LED of most ESP12F modules, blinks every 10 seconds |

All pins, the UDP port and the group addresses are substitutions at the top of the file. All lamps in a group must use the same port and addresses.

The lamp is off at boot until the cycle has a valid clock and decides it should be on.

### Entities

| Entity | What it does |
|---|---|
| Lamp | The relay |
| Device Label | A short name for you, for example `Chalet14` or `Lamp03` |
| Device Group | Lamps with the same group name act as one |
| Use sun | Cycle follows the sun |
| Use schedule | Cycle follows the schedule times |
| Use daylight sensor | Switch input is a light sensor instead of a wall switch |
| Twilight level | Normal, Half Civil or Civil |
| Schedule on / Schedule off | The pair of times of the schedule |
| Location Latitude / Longitude | Used for the sun calculation, one decimal is enough |
| Timezone | POSIX timezone text, see below |
| Twilight Info | The next sun up and sun down |
| Lamp Schedule | The resulting on and off times |

Most configuration entities are disabled by default in Home Assistant. Enable the ones you need, however it may be easiest to simply configrure from the Device Web UI
All parameters are used and shared amoung the group, regardless if they are enabled in HA or not. 

### Setting up location and time

Set the latitude and longitude of the lamp. The defaults in the file are for you to replace.

The timezone is a POSIX string, not a name. Examples:

| Place | Timezone text |
|---|---|
| Central Europe (Amsterdam, Stockholm, Berlin) | `CET-1CEST,M3.5.0,M10.5.0/3` |
| UTC | `UTC0` |
TODO: if TZ is blanked out, populate it with timezone from HA

The time comes from Home Assistant or from NTP. On every WiFi connect the lamp tries the DHCP gateway as NTP server first, and a public pool server second, so no addresses are hardcoded. Without a valid clock the cycle does nothing and the wall switch keeps working. The only exception is sensor-only mode, which does not need a clock.

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

Times may be past midnight. Schedule off at 00:30 is fine, and so is a schedule that runs from 23:30 to 03:00.

**Use sun swaps the times.** When you switch Use sun on or off by hand, Schedule on and Schedule off are swapped. With the sun, the schedule normally means "on in the evening until late", and without the sun it means "on in the morning". You will probably still adjust them. The swap does not happen at boot, and not when the change comes from the group.

**The schedule is simple on purpose.** One pair of times, every day. No weekends, no late Saturdays.

**Sun levels.**

| Twilight level | Sun elevation |
|---|---|
| Normal | -0.833 degrees, the official sunrise and sunset |
| Half Civil | -3 degrees, adds about 15 minutes of daylight|
| Civil | -6 degrees, adds about 30 minutes |

Check the actual info text for actual times, as the "about 15 minutes" is in moderate conditions. But far up north near midsummer can add infinite time (sun never sets or rises)

### Wall switch and daylight sensor

The wall input has two jobs, chosen by Use daylight sensor.

- **Off (default): wall switch.** One or more switches can exist in multiple locations, every change of any switch toggles the lamp, like a hotel or hallway switch setup. The first state after power-up is ignored, so the position of the switch at boot does not toggle the lamp.
- **On: daylight sensor.** The input is a state. The sensor is read by the lamp it is connected to, and that lamp shares the resulting lamp state with the group. Only one lamp needs a sensor.

### Groups

Every lamp has a Device Group. Lamps in the same group, on the same network, keep these in sync:

- the lamp state
- Use sun, Use schedule
- Twilight level
- latitude, longitude, timezone
- Schedule on and Schedule off

These stay local to each lamp: Device Group, Device Label and Use daylight sensor.

How it behaves:

- A change on one lamp is sent to the group, and applied by the others.
- Every lamp also runs its own cycle. The first lamp to reach a switching moment moves the group.
- A lamp that boots does not announce its own state. It asks the group for the current configuration instead.
- An empty group name means no sending and no receiving: a stand-alone lamp.
- The group name cannot contain a colon.
- Lamps send each message to both the broadcast and a multicast address. Your network must allow broadcast and multicast between WiFi clients. Turn off client isolation if you use it.

### Home Assistant

Rename the device in Home Assistant to the Device Label, so you know which lamp is which. The ESPHome device name itself is `outdoor-lamp` plus the last bytes of the MAC address.

### Troubleshooting

| Problem | Check |
|---|---|
| The lamp never follows the cycle | Lamp Schedule says "No valid clock": check WiFi, NTP and Home Assistant time |
| Lamp Schedule says "Manual switch only" | Use sun, Use schedule and Use daylight sensor are all off |
| Switching at the wrong time of day | Timezone text, latitude and longitude |
| Lamps do not follow each other | Same group name, same UDP port and addresses, network allows broadcast and multicast |
| New lamp does not get the group settings | It asks again when WiFi reconnects, or when you change its group name |
| Lamp is on when the cycle says off | A manual or group change lasts until the cycle next changes state |
| Wall switch does nothing | Use daylight sensor is on, so the input is read as a sensor |

### Status

Early version. Expect changes.
