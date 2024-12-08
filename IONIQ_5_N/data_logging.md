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

TODO: Find good spot(s) to plug into the CAN bus, preferably without having to cut the wires / isolation.

TODO: Find which CAN IDs and bytes map to what parameters.

## Data available via the OBD-II port

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

TODO: add points on how to query various parameters via OBD-II.
