# Thin-edge data model

## Motivations

* Move telemetry data around thin-edge components: the devices, the cloud, the sensors, the child devices, the analytics engines ...
* Ease the ingestion of telemetry data coming from various sources, possibly implemented by tiers.
* Ease the transfer of telemetry data to various cloud.
* Ease the exchange of telemetry data between processing component running on a thin-edge device.

## Current state of thin-edge JSON version 0.5

Benefits:
* Simple to learn, yet powerful
* Requires no configuration
* Atomic messages can be grouped into larger messages
  * making the transfers more efficient,
  * providing a better UX on the cloud console,
  * and reducing the customer's bill (based on the message rate).
* Writing a mapper between thin-edge JSON and another format is simple,
  * as long as there is one-one correspondence between the messages.

Limitations:
* Many data sources *already* exist and produce data that can be mapped to thin-edge JSON,
  * but are not *actually* thin-edge JSON,
  * making mandatory yet another data mapper.
* Thin-edge JSON is focused on a format, while what we need first is a data model.
* The current format parser is too rigid,
  * rejecting the whole message when only one part is not meaningful.
* Many data sources are publishing data with one topic per kind of data
  * while thin-edge expect all the data on a single topic.
* Writing a mapper that groups messages as those produced by `collectd` is a more complex task than expected.
  * One should leverage the effort done for `collectd` and use it 
* All the data published on `tedge/measurements` are sent to the cloud.
  * What if we wish to filter some data out?
  * What if we wish to pre-process the data to only forward an aggregate?
* The meaning and purpose of multi-measurements is not clear.
* Possible name collisions with reserved key words (`time` and `type`)?
* Non numeric values are rejected.
* Accept only ISO-8601 timestamp, while Unix Epoch is often used by data sources. 
* The child device identifiers are expected to be in the topic name,
  * while some data sources published their identifier along the message.
* There is no way to attach metadata to a data source (e.g the unit).
* There is no way to attach metadata to a data point (e.g the quality of the measurement).

## Requirements for thin-edge JSON version 1.0

* Provide a clear semantics of the data that can be moved around thin-edge components.
  * Given two sets of messages, are they equivalent or not?
* Provide a reference mapper that move data around topics from different producers and consumers.
* Prefer convention to configuration
  * By convention, messages are exchanged over pre-defined topics - with a format defined by the topic.
  * Configuration is required only to overcome a convention.
    Ingesting data from `collectd`, should be as easy as telling thin-edge
    to ingest not only `tedge/measurements` but also `collectd/raspberrypi`.
* Ingest data from the main device as well as from child devices / named components.
* Provide clear equivalence between messages independently of their shape. For instance, between:
  * `{ "memory": { "used": 1085927424 } }` published on `tedge/measurements`
  * `{ "memory/used": 1085927424 }` published on `tedge/measurements`
  * `1085927424` published on `some/input/memory/used`
* Accept ISO-8601 and Unix Epoch timestamps
* Provide time-window batching of messages
* Take thin-edge as a reference to provide simple mapping configuration.
  * e.g: `[collectd/raspberrypi/${measurement.group}/${measurement.name}] ${measurement.epoch}:${measurement.value}`
* Let the data sources add any meta-data even if not processed by thin-edge.
  * It must be okay for a sensor to provide a serial number along its measurements.
  * It must be okay for an OPCUA data source to tag the measurements with a quality code. 

__Non goals__:
* No complex message processing done by the reference mapper.
  * The reference mapper only decomposes the input messages into measurement constituents
    and recomposes output messages from these constituents.
  * Any value transformations is delegated to external components as Apama.
* No binary message format.
  * The focus of thin-edge is on JSON and CSV message payload.
  * In another kind of payload is produced or consumed by a component,
    these messages will have to processed by a tier mapper.

## Proposal

### Data model

__Series of measurements__

* One have to distinguish the time-series from the data points.
  * A metric is a time-series of values, the data points over time.
  * A measurement is a data point, the value taken by a metric at some time.
* A metric, the time-series, is defined by:
  * A source: i.e. the device, a child device, a sensor or a process.
  * A type name that uniquely identifies the measurement for a given source.
* A measurement, a data-point, is defined by:
  * A metric
  * A timestamp
  * A value
* Meta-data might be attached to a metric:
  * A group or a tag related to the source: a location, a device type, a serial number ...
  * The unit of the measurements.
* Meta-data might be attached to a measurement:
  * The quality of the measurement (see e.g. [OPC Quality Code](https://honeywellprocess-community.force.com/opcsupport/s/article/What-are-the-OPC-Quality-Codes)).

__Measurement collection over MQTT__ 

Thin-edge use MQTT topics to move telemetry data around.
* An MQTT topic
  * can be specific to a metric (e.g. `collectd/raspberrypi/memory/used`)
  * or multiplex several metrics (e.g. `tedge/measurements`).
* The key attributes that define a metric (source and type) can be:
  * implicit (any measurement published on `tedge/measurements` is coming from *the* device),
  * given by the message payload (as the type is for `{ "temperature": 1085927424 }` ),
  * given by the topic name (as for `collectd/raspberrypi/memory/used`).
* The timestamp for a measurement can be:
  * directly attached to the value (as for a collectd data point),
  * share by a set of measurements (as in `{ "time": "2020-10-15T05:30:47+00:00", "temperature": 25, "pressure": 98 }`),
  * or even implicit (and ambiguously defined as the current time of the consumer).

To describe what is published on a topic, we need the following constituents:
* The topic name and rules to extract metric attributes from it.
* The message payload and rules to extract measurement attributes from it.
* The values for any implied attributes.

For instance, for collectd:

```
topic = 'collectd/raspberrypi/${metric.group}/${metric.name}'
payload = CSV('${measurement.epoch}:${measurement.value}')
implied = {
   metric.source = device
}
```

Or for thin-edge JSON 0.5:

```
topic = 'tedge/measurements/${metric.source}'
payload = JSON('{ time: "${measurement.time}", "${metric.name}":${measurement.value} }')
```

## Message transformations

A main goal is to be able to move data from one set of topics to another.
For that one need to defined transformation that doesn't alter the semantics of the messages.
The point is not necessarily to implement all these transformations but to leverage them.

__Grouping measurements with a shared timestamp__

The canonical thin-edge JSON message:

```
{
    "time": T1,
    "temperature": 25
    "pressure": 98
}
```

... can be decomposed into a list of messages that duplicate the timestamp:

```
[
    {
        "time": T1,
    	"temperature": 25
    },
    {
        "time": T1,
    	"pressure": 98
    }
]
```

Vice-versa, the latter message can be transformed into the former:
* provided the timestamps are the same,
* or close enough (say within the same millisecond).

__Grouping measurements with a shared metric group__

The canonical thin-edge JSON multi-valued measurement:

```
{
    "memory": {
        "used": 1085927424,
        "cached": 1374941184,
        "free": 874941184,
    }
}
```

is equivalent to:

```
{
    "memory/used": 1085927424,
    "memory/cached": 1374941184,
    "memory/free": 874941184,
}
```

## Thin-edge Mapper

TBD

### Default behavior
### Configuration
