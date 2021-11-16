# Supported operations

## Support for operations

Supported operations would be a part of thin-edge data model and supported by `operation configuration` files in the filesystem.

Many `thin-edge.io` components can read operation files and use them to perform operations, but only one component shall report operations to an external connection (if supported by provider, e.g. for c8y cloud it shall be the mapper).

Supported operations are listed per supported cloud.

Supported operations can be modified using `tedge cli` tool.

Operations supported by `thin-edge.io` shall be stored in the `operations` directory of `thin-edge.io` configuration, it is `/etc/thin-edge/operations` by default.
Exactly one operation type shall be expressed by each operation file in the filesystem.

Per cloud operations are achieved by use of subdirectories, e.g. `c8y/` for Cumulocity IoT, e.g.: `/etc/tedge/operations/c8y/c8y_SoftwareUpdate` (in this example `c8y_` in the name of operation and is required as it is actual name of the operation in c8y).

The user can add operations by adding the operation file to the filesystem at any time of `thin-edge.io` lifecycle.

Operation files are in `toml` format and can be empty files denoting no additional configuration or parameters are required (e.g. natively supported operations or trivial operations).

The operation file name is the same as the operation name as it is used to determine the exact value to be forwarded to the cloud.

> Consideration: FS permissions, what to set, who should be able to change it?

## Operation configuration

Operations may need to be configured to be executed as intended. For example `c8y_LogfileRequest` requires additional parameter to be set, `log_type` in this case. This can be achieved by adding a configuration settings in the operation file following the proposed structure.

All operation files are `toml` format.

Some tables are optional, but if present they must be in the following format:

### Examples

#### Example 1

Given operation `restart`, the operation file would be created in `/etc/tegde/operations/c8y` with the filename `c8y_Restart` so the full path would be `/etc/tegde/operations/c8y/c8y_Restart`.
Assuming `restart` operation requires no additional parameters as well as can be executed directly in the executor (assume a mapper) no additional configuration is required.

An empty `toml` file is still valid `toml` file and can be used to indicate no additional configuration is required.

Given a mapper the flow would be as follows:

1. Mapper reads directory for the cloud which it supports i.e. `/etc/tegde/operations/c8y`.
2. Mapper finds the operation file for the operation `restart` in the directory `c8y` with filename `c8y_Restart` which is the only operation.
3. Mapper takes the filename as operation name and reads the operation file.
4. Empty operation file means there are no additional configuration parameters and mapper is ready to send supported operations message to c8y which contains the list of supported operations i.e. `c8y_Restart`.
5. Cloud operator wants restart the device therefore they send the operation message to the device which mapper interprets as a restart operation and mapper executes restart.

#### Example 2 (special case)

Given operation `c8y_LogfileRequest`, the operation file would be created in `/etc/tegde/operations/c8y` with the filename `c8y_LogfileRequest` so the full path would be `/etc/tegde/operations/c8y/c8y_LogfileRequest`.
Operation `c8y_LogfileRequest` requires additional parameter `log_type` to be set.

With the operation file the following structure is written:

```toml
[exec]

command = "/etc/tedge/plugins/c8y_LogfileRequest"
root = true

[extras]
  # Additional configuration if the operation requires additional configuration
  log_type = ["error"]
```

Given a mapper the flow would be as follows:

1. Mapper reads directory for the cloud which it supports i.e. `/etc/tegde/operations/c8y`.
2. Mapper finds the operation file for the operation `c8y_LogfileRequest` in the directory `c8y` with filename `c8y_LogfileRequest` which is the only operation.
3. Mapper takes the filename as operation name and reads the operation file.
4. Operation file contains additional configuration parameters and mapper shall know (i.e. implements) how to interpret the configuration (in this case mapper has to add new fragment to the device type therefore can read the `log_type` configuration and send new `log_type` message) ready to send supported operations message to c8y which contains the list of supported operations i.e. `c8y_LogfileRequest`. E.g. `114,c8y_LogfileRequest`
5. Cloud operator wants to retrieve logs from the device therefore they send the operation message to the device which mapper interprets as a `c8y_LogfileRequest` operation and mapper executes `c8y_LogfileRequest`.

#### Example 3

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

#### Example 4

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

### Operation file structure

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

Tables `[exec]` or `[mqtt]` must be present if config is not empty.
Tables `[exec]` and `[mqtt]` are mutually exclusive and only one of them can be used.
Table `[noop]` can be introduced for operations that do not require any additional configuration nor additional binary invocations.

Basic tables names are fixed and are `exec`, `mqtt`, `noop` and `extras`. Additional tables can be added as needed.

## Operation invocation

Operations executors, specifically invoked binaries are executed by `thin-edge.io`. As some of these binaries require elevated permissions (e.g. sudo), they are executed by `thin-edge.io` as a user with sudo privileges (can be indicated in configuration if root is required).
Some operations may be invoked ad-hoc at the operation request and therefore require `exec` table to be present where all if any parameters to the call shall be defined.

Some operations may be long running daemons and therefore may require `mqtt` table to be present.

> Considerations: How to pass parameters to the operation, e.g. expected operation remote access requires the device to connect to a specific websocket and it's URL is passed as part of the operation message and needs to be passed to the executing binary?
>
> * Is adding schema to the operation message required? This adds significant amount of complexity to the message format and it's not clear how to pass parameters to the operation.
> * Is hardcoding the operation message format required? Can this be a quick fix for drop 1?
> * Should operations executor be able to accept plain parameters and is it safe?

## `thin-edge.io` tooling for operations management

`thin-edge.io` provides cli tool for operations management.

* use `tedge cli` command to add or remove operations one by one, list all operations, list all operations per cloud
  * use new tedge subcommand `tedge operations`
  * `tedge operations` supports following operations:
    * `add cloud_name operation_name [--config configuration_filepath]` - adds single operation to the list if doesn't exist
    * `remove cloud_name operation_name` - removes single operation from the list if exists
    * `list [cloud_name]` - lists all operations, unless specific cloud table name provided, then lists only operations for the cloud if exists

e.g.:

* `tedge operations add c8y c8y_Restart`
* `tedge operations add c8y c8y_LogfileRequest --config ./logfile_config`

* Future extension should provide a tool to create operations files - OUT OF SCOPE.
* Some configuration templates are going to be provided in the `thin-edge.io` repository.

Naming and details subject to change and comments.

## Use in tedge components

`thin-edge.io` mappers should pickup operations per cloud from operations repository (filesystem), but an executor like agent to should be provided to execute them (e.g. permissions or state control). This way the executor can be configured to use different operations for different components.

## Persistance

As thin-edge.io configuration directory shall be installed in static storage supported operations will be persistent on the system and will last over reboots and power downs.

## Adding supported operations

### Adding supported operations remotely

Using `tedge cli` to add supported operations allows any tedge components (or even any device system component) to extend the list of supported operations on demand.

`tedge cli` tool can be scripted and therefore when installing new components using thin-edge.io software management supported operations can be added as a part of installation script (e.g. for apt/deb `postinst` script may execute necessary steps), or if it is a custom plugin supporting other package the finalize phase could invoke some metadata/postinstall script in the `finalize` phase.

> Add paragraph about how operations will be reloaded after new operations are added, e.g. which components is responsible for mapper/bridge restart etc.

```shell
#!/bin/sh

set -e

tedge operations add c8y c8y_Apama
tedge operations add c8y c8y_BatchAnalytics --config ./batch_analytics_config
```

> Note: In cases when tedge cli is not installed (currently not an option) one can use direct file modification to add or remove supported operations.
