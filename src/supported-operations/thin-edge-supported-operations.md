# Supported operations

## Support for operations

Supported operations would be a part of thin-edge data model and supported by `operation configuration` files in the filesystem.

Supported operations are listed per supported cloud.

Supported operations can be modified using tedge cli tool.

Operations supported by `thin-edge.io` would be stored in the `operations` directory of `thin-edge.io` configuration, it is `/etc/thin-edge/operations` by default.
Single operation type is expressed by operation configuration file in the filesystem.

Many `thin-edge.io` components can read operation configuration files and use them to perform operations, but only one component should report operations to an external connection (if supported by provider, eg. for c8y cloud it could be mapper).

> Alternatives:

### Option 1

Operations use prefixes to identify the cloud, eg `c8y_SoftwareUpdate`, `az_SoftwareUpdate`.

### Option 2

Per cloud operation is achieved by use of subdirectories, eg `c8y/`, eg: `/etc/tedge/operations/c8y/c8y_SoftwareUpdate` (in this example `c8y_` in the name of operation is required as it is actual name of the operation in c8y, this could be avoided but more on this later), ...

> Done

The user can add operations by adding the operation configuration file to the filesystem at any time of `thin-edge.io` lifecycle.

Configuration files are in toml format and can be empty files denoting no additional configuration or parameters are required (eg, natively supported operations or trivial operations).

The configuration file name is the same as the operation name.

> Consideration: FS permissions, what to set, who should be able to change it?

## Operation configuration

Operations may need to be configured to be executed as intended. For example `c8y_LogfileRequest` requires additional parameter to be set, `log_type` in this case. This can be achieved by adding a configuration settings in the configuration file following the proposed structure.

All operation configuration files are `toml` format.

Some tables are optional, but if present they must be in the following format:

```toml
[exec]
  # Exec configuration if the operation requires command execution
  # Required
  command = "echo"
  # Optional
  args = ["hello"]
  root = true

[mqtt]
  # MQTT configuration if the operation requires MQTT communication, eg forwarding message JSON on the bus
  # Required
  topic = "tedge/logs"
  # Optional
  broker = "mqtt.tedge.io"
  port = 1883
  username = "username"
  password = "password"

[extras]
  # Additional configuration if the operation requires additional configuration
  # Required
  log_type = ["error"]
  # Optional
  log_level = "error"
```

Tables `[exec]` or `[mqtt]` must be present if config is not empty.
Tables `[exec]` and `[mqtt]` are mutually exclusive and only one of them can be used.
Table `[noop]` can be introduced for operations that do not require any additional configuration nor additional binary invocations.

Basic tables names are fixed and are `exec`, `mqtt`, `noop` and `extras`. Additional tables can be added as needed.

## Operation invocation

Operations executors, specifically invoked binaries are executed by `thin-edge.io`. As some of these binaries require elevated permissions (eg, sudo), they are executed by `thin-edge.io` as a user with sudo privileges (can be done in indicated in configuration if root required).
Some operations may be invoked ad-hoc at the operation request and therefore require `exec` table to be present where all if any parameters to the call shall be defined.

Some operations may be long running daemons and therefore may require `mqtt` table to be present.

## `thin-edge.io` tooling for operations management

`thin-edge.io` provides cli tool for operations management.

* use `tedge cli` command to add or remove operations one by one, list all operations, list all operations per cloud
  * use new tedge subcommand `tedge operations`
  * `tedge operations` supports following operations:
    * `add cloud_name operation_name [--config configuration_filepath]` - adds single operation to the list if doesn't exist
    * `remove cloud_name operation_name` - removes single operation from the list if exists
    * `list [cloud_name]` - lists all operations, unless specific cloud table name provided, then lists only operations for the cloud if exists

Eg:

* `tedge operations add c8y c8y_Restart`
* `tedge operations add c8y c8y_LogfileRequest --config ./logfile_config`

* Future extension should provide a tool to create operations configuration files - out of scope for this spec. Some configuration templates are provided in the `thin-edge.io` repository.

Naming and details subject to change and comments.

## Use in tedge components

`thin-edge.io` mappers should pickup operations per cloud from operations repository (filesystem), but it is recommended to have a central executor like agent to execute them (eg permissions or state control). This way the executor can be configured to use different operations for different components.

## Persistance

As config is static it will be persistent on the system and will last over reboots and power downs.

## Adding supported operations

Using `tedge cli` to add supported operations allows any tedge components (or even any device system component) to extend the list of supported operations on demand.

`tedge cli` tool can be scripted and therefore when installing new components using tedge software management the operation it can be added as a part of installation script (eg for apt/deb postinst script may execute necessary steps), or if it is a custom plugin supporting other package the finalize phase could invoke some metadata/postinstall script in the `finalize` phase.

```shell
#!/bin/sh

set -e

tedge operations add c8y c8y_Apama
tedge operations add c8y c8y_BatchAnalytics --config ./batch_analytics_config
```

In cases when tedge cli is not installed (currently not an option) one can use direct file modification to add or remove supported operations.
