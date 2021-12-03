# `thin-edge.io` tooling for operations management

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

`thin-edge.io` mappers should pickup operations per cloud from operations repository (filesystem),
but an executor like agent to should be provided to execute them (e.g. permissions or state control).
This way the executor can be configured to use different operations for different components.


## Adding supported operations

### Adding supported operations remotely

Using `tedge cli` to add supported operations allows any tedge components (or even any device system component) to extend the list of supported operations on demand.

`tedge cli` tool can be scripted and therefore when installing new components using thin-edge.io software management supported operations can be added as a part of installation script (e.g. for apt/deb `postinst` script may execute necessary steps), or if it is a custom plugin supporting other package the finalize phase could invoke some metadata/postinstall script in the `finalize` phase.


```shell
#!/bin/sh

set -e

tedge operations add c8y c8y_Apama
tedge operations add c8y c8y_BatchAnalytics --config ./batch_analytics_config
```

> Note: In cases when tedge cli is not installed (currently not an option) one can use direct file modification to add or remove supported operations.
