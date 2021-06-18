# Package Manager Plugin API

Thin-edge uses plugins to delegate to the appropriate package managers and installers
all the software management operations: installation of packages, uninstallations and queries.

* A package manager plugin acts as a facade for a specific package manager.
* A plugin is an executable that follows the [plugin API](./#plugin-api).
* On a device, several plugins can be installed to deal with different kinds of package.
* Each plugin is given a name that is used by thin-edge to determine the appropriate plugin for a module.
* All the operations on a module are directed to the plugin which name is the module type name.
* Among all the plugin, one can be distinguished as the default plugin.
* The default plugin is invoked when no module type can be determined by the system.
* Several plugins can co-exist for a given package manager as long as they are given different name.
  Each can implement a specific software management policy.

## Plugin repository

* To be used by thin-edge, a plugin has to stored in the directory `/etc/tedge/sm-plugins`.
* The same plugin can be given different names, using virtual or hard links.
* One of the plugin can be given the name `default`. This plugin is then used as the default plugin.
* To make the difference between a plugin alias and an actual plugin,
  thin-edge invokes the [`type`](./#the_type_command) function of the plugin.
  If alias plugin returns the type name of the actual plugin,
  while an actual plugin returns its own name.

On start-up and sighup, the sm-agent registers the plugins as follow:
1. Iterate over the executable file of the directory `/etc/tedge/sm-plugins`.
2. Check the executable is indeed a plugin, calling the [`type`](./#the_type_command) command.
3. Register as actual plugin any plugin which name is the module type.
3. Register as plugin alias any plugin which module type is a different plugin.

On request of an action, the sm-agent determines the appropriate plugin for an action as follows:
* If the action has to be dispatched to all the plugins (as a `list` action),
  the action is forwarded only to the actual plugins (that is those they are not aliases).
* If the action is specific to a module with an associated type, the action is directed to the plugin for that type.
* If the action is specific to a module with no associated type, the action is directed to the plugin named default.
* An error is returned by the sm-agent if no plugin can be associated to the operation.

## Plugin API

* A plugin must at least implement all the commands used by the sm-agent of thin-edge.
  But, a plugin can implement any other commands.
  This also applies to the options for these commands.
  A plugin can support more options than used by thin-edge.
  This gives a way to derive a specialized plugin from a more generic one.
* A plugin might have a configuration file.
  * It can be a list of remote repositories, or a list of software modules to be excluded.
  * These configuration files are managed from the cloud via the sm-agent (TODO: how).

### Input, Output and Errors

* The plugins are called by the sm-agent using a child process for each action.
* For the current list of commands there is no input beyond the command arguments,
  and a plugin can close its `stdin`.
* The `stdout` and `stderr` of the process running a plugin command are captured by the sm-agent.
  * These streams might be forwarded to cloud (depending of the cloud mapper).
  * These streams don't have to be the streams returned by the underlying package manager.
    It can be a one sentence summary of the error, redirecting the administrator to the package manager logs.
* A plugin must return the appropriate exit status after each command.
    * In no cases, the error status of the underlying package manager should be reported.
* The exit status are interpreted by sm-agent as follows:
    * __`0`__: success.
    * __`1`__: usage. The command arguments cannot be interpreted, and the command has not been launched.
    * __`2`__: failure. The command failed and there is no point to retry.
    * __`3`__: retry. The command failed but might be successful later (for instance, when the network will be back).
* If the command fails to return within 5 minutes, the sm-agent reports a timeout error:
    * __`4`__: timeout.

### The `type` command

When called with the `type` command, a plugin returns the software module type it can be used for.

```shell
$ /etc/tedge/sm-plugins/debian type
debian
```

This name is used by the software management agent to determine which plugin, amongst all the available ones,
is appropriate for an operation.

Note that a plugin alias returns the type of the referenced plugin.
```
$ /etc/tedge/sm-plugins/deb type
debian
$ /etc/tedge/sm-plugins/epl type
apama
```

Contract:
* This command take no arguments.
* If an error status is returned, the executable is removed from the list of plugins.

### The `list` command

When called with the `list` command, a plugin returns the list of modules that have been installed with this plugin.

```shell
$ debian-plugin list
...
{"type":"debian","name":"collectd-core","version":"5.8.1-1.3"}
{"type":"debian","name":"mosquitto","version":"1.5.7-1+deb10u1"}
...
```

Contract:
* This command take no arguments.
* If an error status is returned, the executable is removed from the list of plugins.
* The list is returned using the [jsonlines](https://jsonlines.org/) format.
    * __`type`__: the software module type name of the module. This name must match the name returned by the `type` command.
    * __`name`__: the name of the module. This name is the name that has been used to install it and that need to be used to remove it.
    * __`version`__: the version currently installed. This is a string that can only been interpreted in the context of the plugin.

### The `prepare` command

The `prepare` command is invoked by the sm-agent before a sequence of install and remove commands

```
$ /etc/tedge/sm-plugins/debian prepare
$ /etc/tedge/sm-plugins/debian install x
$ /etc/tedge/sm-plugins/debian install y
$ /etc/tedge/sm-plugins/debian remove z
$ /etc/tedge/sm-plugins/debian finalize
```

For many plugins this command will do nothing. However, It gives an opportunity to the plugin to:
* Update the dependencies before a sequence of operations.
   Notably, a debian plugin can update the `apt` cache issuing an `apt-get update`.
* Start a transaction, in case the plugin is able to manage rollbacks.

Contract:
* This command take no arguments.
* No output is expected.
* If the `prepare` command fails, then the planned sequences of actions (.i.e the whole sm operation) is cancelled. 

### The `finalize` command

The `finalize` command closes a sequence of install and remove commands started by a `prepare` command.

This can be a no-op, but this is also an pportunity to:
* Remove any unnecessary module after a sequence of operations.
* Commit or rollback the sequence of operations. 

Contract:
* This command take no arguments.
* No output is expected.
* If the `finalize` command fails, then the planned sequences of actions (.i.e the whole sm operation) is reported as failed,
  even if all the atomic operations has been successfully completed.

### The `install` command

The `install` command installs a single package module, possibly with some expected version.

```
$ plugin install NAME [--version VERSION] [--file FILE]
```

Contract:
* The command requires a single mandatory argument: the module name.
  * This module name is meaningful only to the plugin
    and is transmitted unchanged from the cloud to the plugin.
* An optional version string can be provided.
  * This version string is meaningful only to the plugin
    and is transmitted unchanged from the cloud to the plugin.
  * The version string can include constraints (as at least that version),
    from the sm-agent this is no more than a string.
* An optional file path can be provided.
  * When the device administrator provides an url,
    the sm-agent downloads the module on the device
    
error if the requested version cannot be installed
* --file vs --url
* mismatch between VERSION and FILE -> 
* exact version or version constraints?

### The `remove` command

```
$ plugin remove NAME [--version VERSION]
```

idem
