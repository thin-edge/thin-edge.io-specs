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

On request of an operation, the sm-agent determines the appropriate operation as follow:
* If the operation is global, the operation is forwarded to all the actual plugins (that is those they are not aliases).
* If the operation is specific to a module with an associated type, the operation is directed to the plugin for that type.
* If the operation is specific to a module with no associated type, the operation is directed to the plugin named default.
* An error is returned by the sm-agent if no plugin can be associated to the operation.

## Plugin API

* A plugin must at least implement all the commands used by the sm-agent of thin-edge.
  But, a plugin can implement any other commands.
  This also applies to the options for these commands.
  A plugin can support more options than used by thin-edge.
  This gives a way to derive a specialized plugin from a more generic one.  

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

### The `list` command

When called with the `list` command, a plugin returns the list of modules that have been installed with this plugin.

```shell
$ debian-plugin list
...
{"type":"debian","name":"collectd-core","version":"5.8.1-1.3"}
{"type":"debian","name":"mosquitto","version":"1.5.7-1+deb10u1"}
...
```

The list is returned using the [jsonlines](https://jsonlines.org/) format.

* __`type`__: the software module type name of the module. This name must match the name returned by the `type` command.
* __`name`__: the name of the module. This name is the name that has been used to install it and that need to be used to remove it.
* __`version`__: the version currently installed. This is a string that can only been interpreted in the context of the plugin.

### The `prepare` command

TBD. The idea is twofold.
1. Give an opportunity to the plugin to update its dependencies before a sequence of operations.
2. Start a transaction, in case the plugin is able to manage rollbacks. 

### The `finalize` command

TBD.
1. An opportunity to remove any unnecessary module after a sequence of operations.
2. Commit or rollback the sequence of operations. 

### The `install` command

TBD.

```
$ plugin install NAME [--version VERSION] [--file FILE]
```

### The `remove` command

```
$ plugin remove NAME [--version VERSION]
```
