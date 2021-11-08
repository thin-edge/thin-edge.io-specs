# Supported operations

## Support for operations

Supported operations would be a part of tedge config.

Supported operations are listed per supported cloud.

Can be modified by hand or using tedge cli tool:

* modify fields in `tedge.toml` file:
  * proposed to use `operations` key in a cloud specific toml table
  * the key is an `array` of `strings` with *literal values* for supported operations (no further modifications required by users, like prepending cumulocity operations with `c8y_`)
  * that means that the key operations would exist in the cloud table like so (example for c8y):

```toml
[...]

[c8y]
operations = ["c8y_SoftwareUpdate", "...", ...]
```

Q: How to support parametrized operations / features for operations?
A1: Maybe use operations name as allowed key in config and allow list of parameters
A2: Maybe add prefix to the key from above.

```toml
[...]

[c8y]
operations = ["c8y_SoftwareUpdate", "...", ...]
c8y_SoftwareUpdate = [""]
```

* use `tedge cli` command to add or remove operations one by one, list all operations, list all operations per cloud
  * use new tedge subcommand `tedge operations`
  * `tedge operations` supports following operations:
    * `add` cloud_name operation_name - adds single operation to the list if doesn't exist
    * `remove` cloud_name operation_name - removes single operation from the list if exists
    * `list [cloud_name]` - lists all operations, unless specific cloud table name provided, then lists only operations for the cloud if exists
  * introduction of a concept for parametrized operations - an operation which requires additional configuration or setting (eg c8y log request also requires supported log types)
    * `add` param cloud_name parameter_name - adds single parameter to the list if doesn't exist
    * `remove` param cloud_name parameter_name - remove single parameter to the list if doesn't exist
    * `list` param [cloud_name]` - lists all parameter, unless specific cloud table name provided, then lists only parameters for the cloud if exists

Eg:

* `tedge operations add c8y c8y_Apama`
* `tedge operations add c8y c8y_BatchAnalytics`

Naming and details subject to change and comments.

> Alternative (1)
>
> * use `tedge cli` command to add or remove operations one by one, list all operations, list all operations per cloud
>   * use tedge subcommand `tedge config`
>   * add new subcommand for `tedge config` to add remove and list:
>     * `tedge config add operations c8y c8y_Apama`
>     * `tedge config remove operations c8y c8y_Apama`
>   * `tedge config` currently supports following operations:
>     * `set`
>     * `unset`
>     * `list` [cloud_name]
>
> Why this approach:
>
> * Is consistent with current config API
>
> Why not this approach:
>
> * May be confusing
> * `certificate` subcommand also modifies config
> * `operations` seems to be specialized similarly to `certificate`
>

## Use in tedge components

As tedge components use tegde configuration to obtain other settings adding new setting for supported operations seems like a logical step to extend it.
There is no need to add new API for tedge config as it already supports retrieving values from config file.

tedge config may require extension to support arrays of strings to be retrieved, but this can be achieved without much effort.

Eg:
tedge c8y mapper can easily obtain list of supported operations from the tedge config interface by calling `tedge_config::query(Setting)` interface.

## Persistance

As config is static it will be persistent on the system and will last over reboots and power downs.

## Adding supported operations

Using tedge config to add supported operations allows any tedge components (or even any device system component) to extend the list of supported operations on demand. To add or remove supported operations one can either modify tedge.toml file directly or use tedge cli tool:

`tedge cli` tool can be scripted and therefore when installing new components using tedge software management it could be called as a part of the `finalize` process of the plugin.

```shell
#!/bin/sh

set -e

tedge operations add c8y c8y_Apama
tedge operations add c8y c8y_BatchAnalytics
```

In cases when tedge cli is not installed (currently not an option) one can use direct file modification to add or remove supported operations:

```shell
add example to add with sed or awk
```
