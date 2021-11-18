# Requirements for Operation Support

## Use cases
* One should be able to add new features to thin-edge with operation plugins.
* An operation can be as simple as executing ad-hoc commands on behalf of a remote use
  or as complex as installing new software versions on the thin-edge device.

## Extensibility
* Thin-edge should be liberal on the use-cases for an operation plugin.
* No constraint on the programming language.
* An operation plugin can be provided by thin-edge or a tier party.

## Cloud specificities
* An operation plugin can be cloud specific or not.
* Each cloud mapper might need specific supports from the plugins (e.g reporting the operation progress and status).
* Each cloud mapper might provide specific supports to the plugins (e.g appropriate bridge topics).
* An operation might make sense only on a specific cloud.

## Installation
* Declaring a new operation must be scriptable, notably to be added in installation scripts.
* The set of installed operations must persist and last over reboots and power downs.
* The device owner can add and remove operations at any time of the device lifecycle,
  thin-edge must trigger in the background any necessary registration and initialisation.
* Thin-edge should provide a support to enforce the installation constraints,
  notably only one plugin should be installed for a given operation for a given cloud.
* Some operations may need to be configured.
  For example, `c8y_LogfileRequest` requires additional parameter to be set, `log_type` in this case.
* Some operations may require elevated permissions to be executed (e.g. sudo),
  and thin-edge must then provide the mechanisms to run this operation accordingly.


## Discovery
* Thin-edge should be able to list all the available operations on a device.
* Supported operations are to be grouped per supported cloud.
* Only one component shall report the set of available operations to the cloud.

## Invocation
* This is an implementor choice to run an operation plugin as a daemon or on request.
* If run as a daemon, it must even be feasible to only declare the operation to thin-edge (on doing so to the cloud),
  and to let the daemon managing the entire protocol for that operation.
  The typical example here is the `c8y-sm` mapper which handles the Software Management operations for Cumulocity.
* If run on request, one must be able to declare *when* the operation will be triggered (on which event),
  and *how* (notably with which parameters).

## CLI support
* It would be convenient to manage the set of supported operations using the `tedge cli` tool.
