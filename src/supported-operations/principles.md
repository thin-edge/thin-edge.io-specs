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
  * Which command to run is specified by the plugin configuration file.
  * Similarly, the configuration file specifies on behalf of which user the plugin command has to be run.
  * ```toml
    [exec]
    command = "/etc/tedge/plugins/c8y_LogfileRequest"
    user = "root"
    ```
* Note that the interpretation of these configuration files is cloud specific.
  * The above examples are Cumulocity specific.
  * The `c8y` mapper needs to know that the file names under `/etc/tegde/operations/c8y` are Cumulocity operation names
    that have to be declared using a `114` smartRest request.
  * The `c8y` mapper needs also to know the `c8y_LogfileRequest` plugin has to be awaken.
