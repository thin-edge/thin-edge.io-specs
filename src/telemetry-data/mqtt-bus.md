<!-- toc -->

# The Thin Edge MQTT bus

## Sending measurements to thin-edge.io

The `thin-edge.io` framework exposes some MQTT endpoints that can be used by local processes
to exchange data between themselves as well as to get some data forwarded to the cloud.
It will essentially act like an MQTT broker against which you can write your application logic.
Other thin-edge processes can use this broker as an inter-process communication mechanism by publishing and 
subscribing to various MQTT topics.
Any data can be forwarded to the connected cloud-provider as well, by publishing the data to some standard topics.

All topics with the prefix `tedge/` are reserved by `thin-edge.io` for this purpose.
To send measurements to `thin-edge.io`, the measurements represented in Thin Edge JSON format can be published 
to the `tedge/measurements` topic.
Other processes running on the thin-edge device can subscribe to this topic to process these measurements.

If the messages published to this `tedge/measurements` topic is not a well-formed Thin Edge JSON, 
then that message won’t be processed by `thin-edge.io`, not even partially,
and an appropriate error message on why the validation failed will be published to a dedicated `tedge/errors` topic.
The messages published to this topic will be highly verbose error messages and can be used for any debugging during development.
You should not rely on the structure of these error messages to automate any actions as they are purely textual data 
and bound to change from time-to-time.

More topics will be added under the `tedge/` topic in future to support more data types like events, alarms etc.
So, it is advised to avoid any sub-topics under `tedge/` for any other data exchange between processes.

Here is the complete list of topics reserved by `thin-edge.io` for its internal working:

| Topic | Description |
| --- | --- |
| `tedge/` | Reserved root topic of `thin-edge.io` |
| `tedge/measurements` | Topic to publish measurements to `thin-edge.io` |
| `tedge/errors` | Topic to subscribe to receive any error messages emitted by `thin-edge.io` while processing measurements|

## Sending measurements to the cloud

The `thin-edge.io` framework allows users forward all the measurements generated and published to
`tedge/measurements` MQTT topic in the thin-edge device to any IoT cloud provider that it is connected to,
with the help of a *mapper* component designed for that cloud.
The responsibility of a mapper is to subscribe to the `tedge/measurements` topic to receive all incoming measurements 
represented in the cloud vendor neutral Thin Edge JSON format, to a format that the connected cloud understands.
Refer to [Cloud Message Mapper Architecture](./mapper.md) for more details on the mapper component.
