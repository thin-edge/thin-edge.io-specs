# Mapping to/from C8Y

## Device Capabilities

### Interpret Device Capabilities to SmartREST Set Supported Operations (114)

Device capabilities are reported as empty messages published to dedicated topics.

|Name in Thin Edge|Topic|C8Y Name|
|---|---|---|
|Software Update|tedge/capabilities/software/update|c8y_SoftwareUpdate|
|Software List|tedge/capabilities/software/list|unused|
|Software Sync|tedge/capabilities/software/sync|unused|

C8Y Mapper needs to report only `c8y_SoftwareUpdate` from this table,
therefore, the mapper needs to subscribe only `tedge/capabilities/software/update`.
If the mapper observed that an empty payload message is published, the mapper

1. publishes an empty payload message to `tedge/capabilities/software/update` to clear the retained message.
2. publishes SmartREST(114) to `c8y/s/us` as follwing.

```
114,c8y_SoftwareUpdate
```

***Future Extension:***\
If C8Y Mapper supports more than `c8y_SoftwareUpdate` operation (e.g. `c8y_Restart`),
the mapper should subscribe more capabilities topics,
and sends corresponded C8Y SupportedOpeation types with one `114` message.


## Software List Operation

### Send Thin Edge JSON Software List Request

Outgoing on the topic `tedge/commands/req/software/list` to SM Agent.

```json
{
    "id": 123
}
```

The mapper must generate a unique ID.
Refer to [SM Agent specification](../sm-agent.md) for more details.

### Translate from Thin Edge JSON Software List Response to SmartREST Set Software List (116) 

The Thin Edge JSON message comes on the topic `tedge/commands/res/software/list` from SM Agent.

```json
{
    "id": 123,
    "status": "successful",
    "currentSoftwareList": [
        {
            "type": "debian",
            "modules": [
                {
                    "name": "nodered",
                    "version": "1.0.0"
                },
                {
                    "name": "collectd",
                    "version": "5.7"
                }
            ]
        },
        {
            "type": "docker",
            "modules": [
                {
                    "name": "nginx",
                    "version": "1.21.0"
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6"
                }
            ]
        }
    ]
}
```

The mapper translates the Thin Edge JSON message to SmartREST (116) format. 

We set rules how to represent `type` in C8Y Cloud
since C8Y doesn't have `type` field in `c8y_SoftwareList` structure.
- The mapper adds `type` as a suffix of a package **version** after `::` (double-colon) 
  if provided. If a version contains `::`, the mapper adds `::` at the end of version 
  even the `type` is default.
  - Example 1: if the package type is `debian`, and the version is `1.0.0`,
    the version that the mapper reports is `1.0.0::debian`.
  - Example 2: if the package type is blank (meaning the agent uses the "default" type),
    and the version is `1.0.0`, the version that the mapper reports is `1.0.0`.
  - Example 3: if the package type is `debian`, 
    and the version is `1.0.0::1` (containing `::` delimiter as a package version),
    the version that mapper reports is `1.0.0::1::debian`.
  - Example 4 (corner case): if the package type is blank,
    and the version is `1.0.0::1` (containing `::` delimiter as a package version),
    the version that mapper reports is `1.0.0::1::`.
- Keep URL fields blank.

The SmartREST(116) message goes out on the topic `c8y/s/us` to Mosquitto bridge.

```
116,nodered,1.0.0::debian,,collectd,5.7::debian,,nginx,1.21.0::docker,,mongodb,4.4.6::docker,
```


## Software Update Operation 

### Send SmartREST 500 Get PENDING operations

The SmartREST 500 message goes out on the topic `c8y/s/us` to Mosquitto bridge.

```
500
```

Then, the mapper will receive PENDING operations if they exist.

### Translate from SmartREST Software Update Operation (528) to Thin Edge JSON Software Update Request

The SmartREST `528` message comes onto the topic `c8y/s/ds` from C8Y Cloud.

```
528,external_id,nodered,1.0.0::debian, ,install,collectd,5.7::debian,https://collectd.org/download/collectd-tarballs/collectd-5.12.0.tar.bz2,install,nginx,1.21.0::docker, ,install,mongodb,4.4.6::docker,,delete
```

There are rules how to convert from SmartREST to ThinEdgeJSON.
1. Mapper gets `type` from the version. 
   Check where is the last `::` (double-colon) and assume that the keyword after the last `::` is the type.
    - If nothing follows after the last `::`, the mapper considers the `type` is "default".
      e.g. `1.0.0::1::`
2. If no `::` is provided in a version, the mapper considers the `type` is "default".
   Namely, leave the value of `type` blank.
3. If the `URL` field is empty or a " " (space), mapper considers that the package should be installed from the standard repository.


Then, the translated Thin Edge JSON goes on the topic `tedge/commands/req/software/update` to SM Agent.

```json
{
    "id": 123,
    "updateList": [
        {
            "type": "debian",
            "modules": [
                {
                    "name": "nodered",
                    "version": "1.0.0",
                    "action": "install"
                },
                {
                    "name": "collectd",
                    "version": "5.7",
                    "url": "https://collectd.org/download/collectd-tarballs/collectd-5.12.0.tar.bz2",
                    "action": "install"
                }
            ]
        },
        {
            "type": "docker",
            "modules": [
                {
                    "name": "nginx",
                    "version": "1.21.0",
                    "action": "install"
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6",
                    "action": "remove"
                }
            ]
        }
    ]
}
```

**Attention**:
- Uninstallation terminologies are different in C8Y and Thin Edge JSON. C8Y uses `delete`, 
  although Thin Edge JSON uses `remove`.
  


### Translate from Thin Edge JSON Software Update Executing to SmartREST Operation Update EXECUTING (501)

Incoming to `tedge/commands/res/software/update` from SM Agent.

```json
{
    "id": 123,
    "status": "EXECUTING"
}
```

The mapper translates it and publishes on `c8y/s/us`.

```
501,c8y_SoftwareUpdate
```

### Translate from Thin Edge JSON Software List Response (Successful) to SmartREST Set Software List (116) and Operation Update SUCCESSFUL (503)

The Thin Edge JSON message comes onto the topic `tedge/commands/res/software/update` from SM Agent.

```json
{
    "id": 123,
    "status": "successful",
    "currentSoftwareList": [
        {
            "type": "debian",
            "modules": [
                {
                    "name": "nodered",
                    "version": "1.0.0"
                },
                {
                    "name": "collectd",
                    "version": "5.7"
                }
            ]
        },
        {
            "type": "docker",
            "modules": [
                {
                    "name": "nginx",
                    "version": "1.21.0"
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6"
                }
            ]
        }
    ]
}
```

The mapper translates it into two messages and publishes onto `c8y/s/us`.

The first message is SmartREST `116`, that is `c8y_SoftwareList`.
The translation rules are the same as described in 
[From Thin Edge JSON Software List Response to c8y_SoftwareList (116)](#translate-from-thin-edge-json-software-list-response-to-smartrest-set-software-list-116).

The second message is operation update to `SUCCESSFUL`.

```
503,c8y_SoftwareUpdate
```

### Translate from Thin Edge JSON Software Update Operation FAILED to SmartREST Set Software List (116) and Operation Update FAILED (502) 
The Thin Edge JSON message comes on the topic `tedge/commands/res/software/update` from SM Agent.

```json
{
    "id": 123,
    "status":"failed",
    "reason":"Partial failure: Couldn't install collectd and nginx",
    "currentSoftwareList": [
        {
            "type": "debian",
            "modules": [
                {
                    "name": "nodered",
                    "version": "1.0.0"
                }
            ]
        },
        {
            "type": "docker",
            "modules": [
                {
                    "name": "nginx",
                    "version": "1.21.0"
                }
            ]
        }
    ],
    "failures":[
        {
            "type":"debian",
            "modules": [
                {
                    "name":"collectd",
                    "version":"5.7",
                    "action":"install",
                    "reason":"Network timeout"
                }
            ]
        },
        {
            "type":"docker",
            "modules": [
                {
                    "name": "mongodb",
                    "version": "4.4.6",
                    "action":"remove",
                    "reason":"Other components dependent on it"
                }
            ]
        }
    ]
}
```

The mapper translates the Thin Edge JSON to SmartREST `116` and `502` then publishes them to `c8y/s/us`.

The first message is SmartREST `116`, that is `c8y_SoftwareList`.
The translation rules are the same as described in 
[From Thin Edge JSON Software List Response to c8y_SoftwareList (116)](#translate-from-thin-edge-json-software-list-response-to-smartrest-set-software-list-116).

The second message is operation update to `FAILED` with a following rule.

- Use the only parent field `reason` as a failure reason to report.

```
502,c8y_SoftwareUpdate,"Partial failure: Couldn't install collectd and nginx"
```

### Send SmartREST Operation Update FAILED (502) only if sending SoftwareList (116) failed

It's possible that sending `c8y_SoftwareList` (116) fails due to the huge payload size.
In this case, the mapper should send only operation `FAILED` to C8Y cloud.

Topic to publish: `c8y/s/us`,

```
502,c8y_SoftwareUpdate,"Failed to send the current software list after software update operation"
```
