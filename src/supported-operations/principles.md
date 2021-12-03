## Proposal 

* An operation is implemented by an executable that is responsible for:
  * the interactions with the cloud (using the locally bridged MQTT topics),
  * requesting any required parameters,
  * reporting the operation progress,
  * returning any expected results.
* On installation, an operation is declared to thin-edge using a configuration file put in an operation directory.
  * This directory is organized in sub-directories per cloud and per operation.
  * Each operation is represented by a configuration file (using the TOML file format).
  * ```shell
    $ ls -l /etc/tegde/operations/c8y
    -rw-rw-r-- 1 user user    688 Jan 1 00:01 c8y_LogfileRequest
    -rw-rw-r-- 1 user user    331 Jan 1 00:01 c8y_SoftwareUpdate
    -rw-rw-r-- 1 user user     40 Jan 1 00:01 c8y_Restart
    ```
* An operation might run independently of thin-edge.
  * In that case, the operation plugin has to run as a daemon (listening for requests)
    and ensure that this daemon is enabled of device re-start.
  * The operation daemon is responsible for triggering the operations.
  * For thin-edge, the operation just needs to be declared using an empty TOML file. 
  * Thin-edge is only responsible for notifying that the operation is available.
* An operation might be executed on request.
  * Thin-edge is then responsible for listening for requests and spawning processes to handle these.
  * The operation configuration provide the topic and the pattern matching the awaking events.
  * Which command to run is specified by the plugin configuration file.
  * Similarly, the configuration file specifies on behalf of which user the plugin command has to be run.
  * ```toml
    [exec]
    topic = "c8y/s/ds"
    on_message = "522,*"
    command = "/etc/tedge/plugins/c8y_LogfileRequest"
    user = "root"
    ```
* Note that the interpretation of these configuration files is cloud specific.
  * The above examples are Cumulocity specific.
    * See [SmartRest2 operation templates](https://cumulocity.com/guides/device-sdk/mqtt/#subscribe-templates) for details.
  * The `c8y` mapper needs to know that the file names under `/etc/tegde/operations/c8y` are Cumulocity operation names
    that have to be declared using a `114` smartRest request.
  * The `c8y` mapper needs also to know the `c8y_LogfileRequest` plugin has to be awaken.


### To be clarified
* Configuration file permissions, what to set, who should be able to change it?
* How the set of operations is reloaded after a new operation has been added?
* Which component is responsible for executing the operation command for a request?
  * The c8y mapper is definitely the component that has to listen for requests
    and to translate operation requests into plugin commands.
    But, it would be better to have the agent dealing with command execution and monitoring.
    The price to pay is a new indirection level (similar to what is done between the sm-c8y mapper and the agent).
* How to pass parameters to the operation?
  * e.g. a remote access operation requires the device to connect to a specific websocket
    and it's URL is passed as part of the operation message and needs to be passed to the executing binary.
  * An option is to pass the whole message from the cloud as the contract implies
    that the operation executable will know what to do with it.
* We could have regex for the topic/topic+message to match the operation and wake appropriate executor?
* Should operations executor be able to accept plain parameters and is it safe? Security considerations.
