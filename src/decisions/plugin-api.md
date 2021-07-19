# Plugin API

## JsonLines is used to return the list of software modules

Three options has been considered:

* CSV with spaces as separators
   * Pros: usual in that context,
   * Cons: fragile (what if an epl module name contains whitespaces?)
* Jsonlines
   * Pros: robust (except against multi-line strings), flexible, extensible, 
   * Cons: unusual
* Json
   * Pros: robust, flexible, extensible,
   * Cons: unusual is that context, not so easy to generate from a shell script (trailing comma),

## Full json messages will be considered later for errors

It might be better to communicate errors via the JSON that's returned rather than an error code.
There's more flexibility - for example better error messages.

However, this pushes more responsibilities to the plugin implementor. Hence, we start just with exit status.
This will be reconsidered when we will have better understanding of the whole inter-actions
between the sm-agent and the plugins.

## The `type` command is removed

This can be reconsidered when we will have better understanding of the whole inter-actions
between the sm-agent and the plugins.

The main question is "do we need to provide the device admin several ways to bind a software module to a plugin?".

* The response can be no. In cumulocity, the device admin will have to be explicit: "collectd:debian" "mypkg:debian" and "myanaltics.epl:apama".
* The response can be yes, leveraging default and file extensions. In cumulocity, the device admin is given more freedom: "collectd" (use default) "mypkg:debian" (explicit) and "myanaltics.epl" (use extension).

If we opt for yes, we need to manage this n-1 relation and ensure that a sw list is built with no duplicates.
* An idea has been to use a type command to distinguish between an alias and an actual plugin.
* An option is to use virtual links so the the system can check the plugin is an alias without support from the plugin. 
* One can have an external conf.
* KISS would be say no to such n-1 relations.

