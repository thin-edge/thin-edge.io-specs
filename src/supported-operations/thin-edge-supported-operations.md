# Operation Configuration Examples

All operation configuration files use the `toml` format.

```toml
[exec]
  # Exec configuration if the operation requires command execution
  # Required
  command = "echo"
  # Optional
  root = true

[mqtt]
  # MQTT configuration if the operation requires MQTT communication, e.g. forwarding message JSON on the bus
  topic = "tedge/logs"

[extras]
  # Additional configuration if the operation requires additional configuration
  log_type = ["error"]
```

* If the operation doesn't require additional configuration, an empty `toml` file can be used.
* Basic tables names are fixed and are `exec`, `mqtt` and `extras`. Additional tables can be added as needed.
* Tables `[exec]` or `[mqtt]` must be present if config is not empty.
* Tables `[exec]` and `[mqtt]` are mutually exclusive and only one of them can be used.

## Example 1

Given operation `restart`, the operation file would be created in `/etc/tegde/operations/c8y` with the filename `c8y_Restart` so the full path would be `/etc/tegde/operations/c8y/c8y_Restart`.
Assuming `restart` operation requires no additional parameters as well as can be executed directly in the executor (assume a mapper) no additional configuration is required.

An empty `toml` file is still valid `toml` file and can be used to indicate no additional configuration is required.

Given a mapper the flow would be as follows:

1. Mapper reads directory for the cloud which it supports i.e. `/etc/tegde/operations/c8y`.
2. Mapper finds the operation file for the operation `restart` in the directory `c8y` with filename `c8y_Restart` which is the only operation.
3. Mapper takes the filename as operation name and reads the operation file.
4. Empty operation file means there are no additional configuration parameters and mapper is ready to send supported operations message to c8y which contains the list of supported operations i.e. `c8y_Restart`.
5. Cloud operator wants restart the device therefore they send the operation message to the device which mapper interprets as a restart operation and mapper executes restart.

## Example 2 (special case)

Given operation `c8y_LogfileRequest`, the operation file would be created in `/etc/tegde/operations/c8y` with the filename `c8y_LogfileRequest` so the full path would be `/etc/tegde/operations/c8y/c8y_LogfileRequest`.
Operation `c8y_LogfileRequest` requires additional parameter `log_type` to be set.

With the operation file the following structure is written:

```toml
[exec]

command = "/etc/tedge/plugins/c8y_LogfileRequest"
user = "root"

[init]
topic = "c8y/s/us"
message = "116,error"

[extras]
  # Additional configuration if the operation requires additional configuration
  log_type = ["error"]
```

> Alternatives:
>
> Option 1: on init of mapper config will contain a command to be called e.g. sent `log_type` message to c8y, only the component knows about it (mapper doesn't need to care about it)
> Option 2: explicit additional message to be send by the mapper if required/defined using a table in config file e.g. `[init]` like in the example above.
>
> Done

Given a mapper the flow would be as follows:

1. Mapper reads directory for the cloud which it supports i.e. `/etc/tegde/operations/c8y`.
2. Mapper finds the operation file for the operation `c8y_LogfileRequest` in the directory `c8y` with filename `c8y_LogfileRequest` which is the only operation.
3. Mapper takes the filename as operation name and reads the operation file.
4. Operation file contains additional configuration parameters and mapper shall know (i.e. implements) how to interpret the configuration (in this case mapper has to add new fragment to the device type therefore can read the `log_type` configuration and send new `log_type` message) ready to send supported operations message to c8y which contains the list of supported operations i.e. `c8y_LogfileRequest`. E.g. `114,c8y_LogfileRequest`
5. Cloud operator wants to retrieve logs from the device therefore they send the operation message to the device which mapper interprets as a `c8y_LogfileRequest` operation and mapper executes `c8y_LogfileRequest`.

## Example 3

Given an operation which requires communication with another component over the bus e.g. `c8y_SoftwareUpdate` and no additional configuration is required.

Following operation file could be used:

```toml
[mqtt]

request = "tedge/commands/req/software/update"
response = "tedge/commands/res/software/update"
```

1. Mapper reads directory for the cloud which it supports i.e. `/etc/tegde/operations/c8y`.
2. Mapper finds the operation file for the operation `c8y_SoftwareUpdate` in the directory `c8y` with filename `c8y_SoftwareUpdate` which is the only operation.
3. Mapper takes the filename as operation name and reads the operation file.
4. Operation file contains additional configuration parameters and mapper shall know (i.e. implements) how to interpret the configuration (in this case mapper has to forward the request to an executor on the bus) ready to send supported operations message to c8y which contains the list of supported operations i.e. `c8y_SoftwareUpdate`. E.g. `114,c8y_SoftwareList`
5. From this point mapper should subscribe to the provided response topic: `tedge/commands/res/software/update`.
6. From this point whenever mapper receives a request for the operation `c8y_SoftwareUpdate` it shall forward the request to the executor on the bus on provided topic: `tedge/commands/req/software/update` and would expect a response on the provided topic: `tedge/commands/res/software/update`.
7. Cloud operator wants to update software on the device therefore they send the operation message to the device which mapper interprets as a `c8y_SoftwareUpdate` operation and mapper forwards the operation request to an executor (in this case the agent) on provided topic `tedge/commands/req/software/update`.
8. Executor processes the request and sends a response on provided topic `tedge/commands/res/software/update`.
9. Mapper translates the response to the cloud format and sends it to the cloud.

## Example 4

Given multiple operation files in operations directory and disregarding additional configuration following operations have been registered: `c8y_LogfileRequest`, `c8y_SoftwareUpdate`, `c8y_Restart`.
Directory content:

```shell
$ ls -l /etc/tegde/operations/c8y
-rw-rw-r-- 1 user user    688 Jan 1 00:01 c8y_LogfileRequest
-rw-rw-r-- 1 user user    331 Jan 1 00:01 c8y_SoftwareUpdate
-rw-rw-r-- 1 user user     40 Jan 1 00:01 c8y_Restart
```

1. Mapper reads directory for the cloud which it supports i.e. `/etc/tegde/operations/c8y`.
2. Mapper finds the operation files in the directory `c8y`.
3. Mapper takes the filenames of all the files in this directory and reads the operation file.
4. Mapper collates the list of supported operations and when it's ready sends supported operations message to c8y which contains the list of supported operations i.e. `c8y_LogfileRequest`, `c8y_SoftwareUpdate`, `c8y_Restart`. E.g. `114,c8y_LogfileRequest,c8y_SoftwareUpdate,c8y_Restart`

