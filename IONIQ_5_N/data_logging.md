# Data logging for Hyundai IONIQ 5 N

## CAN bus data

Passively listening to the CAN bus is a great way to get a lot of data with a high refresh rate.
In other cars I'm seeing 10, 20 or even 100 updates per second for some important data channels.

The nice thing about CAN data is that once you find it, the data mapping for driver inputs is
relatively easy to find. For example, to find the steering wheel angle, you can turn the steering
wheel back and forth while parked in the parking lot, and the go from the lowest available CAN IDs
to higher CAN IDs and see if any of the bytes on those ID seem to correlate with the movement of the
steering wheel.

Some cars expose the data from the CAN bus on the OBD-II port, but some don't. I haven't personally
verified whether the 5 N exposes the data, but based on second-hand knowledge it doesn't.

### Ideal strategy

TODO: Find good spot(s) to plug into the CAN bus near the glovebox or the driver's footwell,
preferably without having to cut the wires / isolation.

TODO: Find which CAN IDs and bytes map to what parameters.

### Alternative strategy

Good people at comma.ai have found some CAN buses coming in to/out of the HDA front-facing cameras.
[Here](https://github.com/commaai/neo/blob/master/car_harness/v3/Hyundai_Q_Harness.pdf) is the
wiring diagram for their "Hyundai Q" harness. You can buy it at https://comma.ai/shop/car-harness

If we can't find a more convenient spot, this might be an acceptable place to get data from the CAN
bus.

### CAN IDs and data mapping

Good people at comma.ai have also posted a
[dbc mapping](https://github.com/commaai/opendbc/blob/master/opendbc/dbc/hyundai_canfd.dbc)
for Hyundai cars, I suspect there will be significant overlap with the protocols used by ECUs in the
5 N.

TODO: Verify and document key data channels.

## Data available via the OBD-II port

Many lap timer apps (RaceChrono, Torque, etc.) allow reading data from the car's OBD-II port using
custom PIDs and equations, and Bluetooth/OBD-II dongles such as OBDLink MX+ and OBDLink LX.

The OBD-II port in the driver's footwell provides some data, but it's harder to find the data
mapping, since OBD-II is a request/response protocol. You have to first find the right ECU to send a
request to, then find the request code, and only then can you look at the bytes of the response.

Importantly, the request/response nature of communication lowers the data refresh rate. In case of
the IONIQ 5 N, things are even more complicated, because most of the useful OBD-II PIDs I've found
so far use multi-packet responses, which takes multiple underlying request/responses to get a single
packet of data. Even querying a single PID the best I got so far was 10 updates per second shared
across all data channels. As a result, the OBD-II data is not well suited for quickly changing
parameters such as the brake pedal pressure or steering angle.

OBD-II can still be useful for some slower changing parameters, such as air temperature, tire
pressures, battery level, etc. They can also be useful to reference when deciphering the CAN bus for
parameters that the driver doesn't directly control, e.g. battery current or voltage.

### Recommended channels for RaceChrono

Given the really low overall refresh rate, it's recommended to use as few channels as possible.
Note that if you already use a channel with a certain OBD-II header and PID, adding another channel
to RaceChrono with the same OBD-II header and PID should not affect the refresh rate, as both
channels will be decoded from the same OBD-II responses.

When using lap timers / data loggers such as RaceChrono, it's very useful to have the speedometer
data, as that allows automatically aligning the rest of the OBD-II data with the GPS data, thus
improving the synchronization between those two sources.

You can get the data like shown in [this](https://www.youtube.com/watch?v=v7Y9Sffpea4) video
if you use the following setup with RaceChrono:

**Fast channels:**

Channel | OBD-II header | PID      | Equation | Notes
------- | ------------- | -------- | -------- | -----
Air temperature | 0x7B3 | 0x220100 | `H * 0.5 - 40` | Value in degrees C, RaceChrono takes care of the conversion. Same header/PID as Speed, so "free".
Battery level | (empty) | 0x220101 | `bytesToUint(raw, 5, 1) * 0.5` | Same header/PID as Power, so "free".
Power (kW) | (empty)    | 0x220101 | `bytesToUInt(raw, 13, 2) * bytesToInt(raw, 11, 2) * 0.01` | Negative values for regen.
Speed   | 0x7B3         | 0x220100 | `bytesToUInt(raw, 30, 1) / 3.6` | Curious how this works past 256 km/h! :)

You can optionally add channels from the "Other channels" section, but I recommend only adding them
as "slow" channels, otherwise the refresh rate of the recommended channels will suffer.

### Other channels

Besides the channels recommended above, you might also find these useful:

**"Free" channels:**

These channels use the same header/PID values as the recommended ones, so they don't affect the overall refresh rate:

Channel | OBD-II header | PID      | Equation | Notes
------- | ------------- | -------- | -------- | -----
Battery current (A) | (empty) | 0x220101 | `bytesToInt(raw, 11, 2) * 0.1` | Negative values for regen.
Battery temperature 1 | (empty) | 0x220101 | `bytesToInt(raw, 16, 1)` | This is the min temperature of the battery?
Battery temperature 2 | (empty) | 0x220101 | `bytesToInt(raw, 15, 1)` | This is the max temperature of the battery?
Battery voltage (V) | (empty) | 0x220101 | `bytesToUInt(raw, 13, 2 * 0.1` |
Engine RPM (Front) | (empty) | 0x220101 | `bytesToInt(raw, 56, 2)` | Drops to zero in e-Shift mode and when using ACC.
Engine RPM (Rear) | (empty) | 0x220101 | `bytesToInt(raw, 54, 2)` |

**"Expensive" channels:**

These channels use different header/PID combinations, and thus adding them lowers the overall refresh rate:

Channel | OBD-II header | PID      | Equation | Notes
------- | ------------- | -------- | -------- | -----
Accelerator position (%) | 0x7E2 | 0x22E004 | `bytesToUInt(raw, 10, 1) * 0.5` | It's a shame it's not a "free" channel!
Battery level | (empty) | 0x220105 | `bytesToUInt(raw, 32, 1) * 0.5` | Another way to get battery level. Doesn't seem to be useful.
Brake position (%) | 0x7D1 | 0x220104 | `bytesToUInt(raw, 39, 2) / 70.0` | Same scale as in the N Menu. It's a shame it's not a "free" channel!
Steering angle | 0x730 | 0x22F010 | `bytesToIntLe(raw, 13, 2) * 0.1` | It's a shame it's not a "free" channel!
Tire pressure (FL) | 0x7A0 | 0x22C00B | `bytesToUInt(raw, 5, 1) * 1.3885254` | Value in kPA, RaceChrono then converts to psi if needed.
Tire pressure (FR) | 0x7A0 | 0x22C00B | `bytesToUInt(raw, 10, 1) * 1.3885254` | Value in kPA, RaceChrono then converts to psi if needed.
Tire pressure (RL) | 0x7A0 | 0x22C00B | `bytesToUInt(raw, 15, 1) * 1.3885254` | Value in kPA, RaceChrono then converts to psi if needed.
Tire pressure (RR) | 0x7A0 | 0x22C00B | `bytesToUInt(raw, 20, 1) * 1.3885254` | Value in kPA, RaceChrono then converts to psi if needed.

TODO: add pointers to more things accessible via OBD-II.

