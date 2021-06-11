# Package Manager Plugin API

A plugin for a package manager is an executable acting as a facade
between the thin-edge software management agent and a specific package manager.

##

## The `type` command

When called with the `type` command, a plugin returns the software module type it can be used for.

```shell
$ debian-plugin type
debian
```

This name is used by the software management agent to determine which plugin, amongst all the available ones,
is appropriate for an operation.

## The `install` command



## The `remove` command

## The `list` command

When called with the `list` command, a plugin returns the list of modules that have been installed with this plugin.

```shell
$ debian-plugin list
...
{"type":"debian","name":"collectd-core","version":"5.8.1-1.3"}
{"type":"debian","name":"mosquitto","version":"1.5.7-1+deb10u1"}

...
```

The list is returned using the [jsonlines](https://jsonlines.org/) format.

## The `prepare` command

## The `finalize` command
