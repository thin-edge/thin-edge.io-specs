# Thin-Edge-Json

Thin Edge JSON is a lightweight format used in `thin-edge.io` to represent measurements data.
This format can be used to represent single-valued measurements, multi-valued measurements
or a combination of both along with some auxiliary data like the timestamp at which the measurement(s) was generated.

## Single-valued measurements

Simple single-valued measurements like temperature or pressure measurement with a single value can be expressed as follows:

```json
{
    "temperature": 25
}
```

where the key represents the measurement type, and the value represents the measurement value.
The keys can only have alphanumeric characters, and the "_" (underscore) character but must not start with an underscore.
The values can only be numeric.
String, Boolean or other JSON object values are not allowed.

## Multi-valued measurements

A multi-valued measurement is a measurement that is comprised of multiple values. Here is the representation of a
`three_phase_current` measurement that consists of `L1`, `L2` and `L3` values, representing the current on each phase:

```json
{
    "three_phase_current": {
      "L1": 9.5,
      "L2": 10.3,
      "L3": 8.8
    }
}
```

where the key is the top-level measurement type and value is a JSON object having further key-value pairs 
representing each aspect of the multi-valued measurement.
Only one level of nesting is allowed, meaning the values of the measurement keys at the inner level can only be numeric values.
For example, a multi-level measurement as follows is NOT valid: 

```json
{ 
    "three_phase_current": {
        "phase1": {
            "L1": 9.5
        },
        "phase2": {
            "L2": 10.3
        },
        "phase3": {
            "L3": 8.8
        }
    }
}
```

because the values at the second level(`phase1`, `phase2` and `phase3`) are not numeric values.

## Grouping measurements

Multiple single-valued and multi-valued measurements can be grouped into a single Thin Edge JSON message as follows:

```json
{ 
    "temperature": 25,
    "three_phase_current": {
        "L1": 9.5,
        "L2": 10.3,
        "L3": 8.8
    },
    "pressure": 98 
}
```

The grouping of measurements is usually done to represent measurements collected at the same instant of time.

## Auxiliary measurement data

When `thin-edge.io` receives a measurement, it will add a timestamp to it before any further processing.
If the user doesn't want to rely on `thin-edge.io` generated timestamps,
an explicit timestamp can be provided in the measurement message itself by adding the time value as a string 
in ISO 8601 format using `time` as the key name, as follows:

```json
{ 
    "time": "2020-10-15T05:30:47+00:00", 
    "temperature": 25, 
    "location": { 
        "latitude": 32.54, 
        "longitude": -117.67, 
        "altitude": 98.6 
    }, 
    "pressure": 98 
}
```

The `time` key is a reserved keyword and hence can not be used as a measurement key.
The `time` field must be defined at the root level of the measurement JSON and not allowed at any other level,
like inside the object value of a multi-valued measurement.
Non-numeric values like the ISO 8601 timestamp string are allowed only for such reserved keys and not for regular measurements. 

Here is the complete list of reserved keys that has special meanings inside the `thin-edge.io` framework
and hence must not be used as measurement keys:

| Key | Description |
| --- | --- |
| time | Timestamp in ISO 8601 string format |
| type | Internal to `thin-edge.io` |

