# Introduction


Thin-edge self-update allows a Cloud Operator to update one thin-edge version that is installed on a device to a newer version, via the Cloud connection. The feature is referred as self-update cause that update process is managed by thin-edge own's Software Management.

# Scope
  * The self-upfate makes use of package manager "APT", but "APT" is not completely fail-safe.<br/>
    For really fail-safe self-update the device would need to provide some transaction-safe package manager and some powerfail-safe filesystem.
     
  * Consequences: 
     * Self-update can fail and could left the system in an unstable state. The device might be disconnected from Cloud.
     * Potential fail reasons (without claim of completeness):
       * No rollback of already processes package of one self-update request, when some subsequent package of same request fails.
         <br/>-> leads to partially executed update. 
       * Power-fail during update.
         <br/>-> leads to partially executed update
         <br/>-> can corrupt the package currently about to be updated
         <br/>-> can corrupt the filesystem
       * Installation of incompatibel set of thin-edge package versions (e.g. SM Agent of one release, and Mapper of another release).
         <br/>-> thin-edge function might not given, when signifcant change happend between both release
       * Tool(s) required by thin-edge installation procedure missing on the device (e.g. 'grep').
         <br/>-> according package installation will stop.
         
  * Measures for more robusness:
    * There are plans for thin-edge packages to reduce risks for installation failures:
      <br/>TODO: For both, add some reference (some ‚ticket, discussion, ...)
      1. Providing one thin-edge core package that contains everything needed for self-update (assumedly "Mapper", "SM Agent" and "SM Plugin"). 
          So self-update just needs that one-and-only package; all other thin-edge packages can be update via usual Software Management.
      2. Avoiding use of external tools in package installation logic (move from Shell scripts to Rust binary).
    * If the custom device provides some transaction-safe package manager and some powerfail-safe filesystem, the device owner can 
      adapt the `tedge` plugin to make use of both. 

# Expected Pre-Conditions 

The device and it's thereon installed thin-edge has to follow precondition below to allow successful self-update:

1) The current (old) thin-edge on the device must be installed with usual thin-edge installation. That is to assure new thin-edge version that is about to be installed has all external and internal files, folders and resources (e.g. config files) available.
2) Deployment of thin-edge and mosquitto must not be changed manually after initial installation (e.g. moving/renaming installed files, moving/renaming configuration files used by thin-edge, renaming users/groups created during installation, ...).

*Hint: Before executing self-update on devices in the field it shall be tested on a kind of golden device that equals the device(s) in the field in system-environment, configuration and thin-edge version.*


# Execution Flow and involved Components

![Sequence Diagram Update SW-list](images/self-update.drawio.svg)


# Design Principles

### Details about the SM Plugin

* Thin-edge updates are managed by a specific software management module type named `tedge`.

* The existing SM plugin "apt" will be used and extended to manage that module type "tedge". 
  * Same executable will be used for both module types, so a softlink must be located in `/etc/tedge/sm-plugins/` named `tedge`, that points to the plugin’s executable „apt“.

* As the plugin serves both module types "tedge" and "apt", the SM Agent will execute the plugin twice with the LIST command; once for each mode type. To distinguish packages listed by APT package manager per module type all packages prefixed with "tedge_" are assumed as module type "tedge".
  * When the plugin is called with LIST command for module type "tedge", packages prefixed with "tedge_" must be reported.
  * When the plugin is called with LIST command for module type "apt", all packages withoutprefix "tedge_" must be reported.

* The agent must request updates on the `tedge` plugin using the `update-list` API of this plugin. So the plugin can process the entire update request without any further SM Agent action. This is required because the agent will be restarted by the plugin.

* Before any package update for module type "tedge" starts, all needed installation resources for the entire update request must be downloaded (i.E. all \*.deb packages for APT). That is to get the plugin uncoupled from any potential network issue during update procedure.
  * For all packages with some URL given in the update request the SM Agent manages download before plugin start, and provides local file paths to the plugin. If one or more package download fails, the SM Agent cancels the entire update request before starting the plugin.
  * For all packages without any URL given in the update request the Package Manager (i.E. APT) will manage the download as part of the update. Therefore implementation of plugin command update-list calls APT separately for download and for installation.
    * First all packages are downloaded by calling APT with command line option `--download-only`.
    * When all packages were successfully downloaded packages are installed by calling APT again with option `--no-download`.
    * If one or more downloads fail, the plugin will not install any package, but return with error.

  * **Important:** For module type „apt“ behavior must be kept as it is currently implemented. That means packages must be downloaded by APT as needed as part of the installation. Otherwise disc space could explode in case of large update requests.

* After processing the complete update request the plugin will publish the overall exit code as MQTT retain message to topic `tedge/plugins/software/tedge`, and exits.<br/>
  NOTE: That is, since during processing the update the SM Agent will restart, and the restarted SM Agent process will be disconnected from plugin process and it's exit code.


### Details about the SM Agent

* Module type „tedge“ must appear exclusively in update requests:
  * The SM Agent must reject any update request that contains module type „tedge“ and other module types. 
    If a request appears that contains "tedge" and any other module type the SM Agent must report error to Cloud and shall create the log message:
    * „Update failed since request contains module type ‚tedge‘ and others &mdash; but module type ‚tedge‘ must be used exclusively.“

* Handling of SM Agent restart:<br/>
  NOTE: Restart of SM Agent by module type tedge is valid, or even expected.

  * At the place where the SM Agent stores an upcoming software update request in the `persistance_store`, the names of the contained module types must be stored.
  * At the place where SM Agent (re)starts and checks the `persistance_store` for any update request, it must avoid to send an error to the Cloud when the update request contains module type „tedge“.<br/>

* Transfer of plugins exit code to the SM Agent:<br/>
  NOTE: During execution of plugin for module type „tedge“ it is valid (or even expected) that the SM Agent will restart. The restarted SM Agent process will be disconnected from still running plugin's process and it's exit code.

  * For module type tedge the SM agent must accept the final exit code of a plugin execution via MQTT retain message on topic `tedge/plugins/software/tedge`, send according update result message to the mapper, and clear the retain message on the topic afterwards.
  * It might happen that the SM Agent does not restart and keeps connected to the plugin for module type "tedge" (e.g. when agent is not part of the update request, or the update fails). Also then the exit code that is reported by the plugin via MQTT must be used, instead of the exit code reported from plugin's process exit.

# Alternative Considerations

That section evaluates the alternative approach to implement a seperate plugin for module type "tedge", instead of extending existing APT plugin. A main motivation for a separate plugin would be to avoid overload of the existing APT plugin and keep it as simple as possible.

Evaluation of both alternatives:

* **Option 1 "Extending APT plugin":** Adds complexity to APT plugin, as below:
  * Adding another regular expression into LIST command to filter for package name with and without prefix "tedge".
  * Adding use of MQTT client to publish exit-code as MQTT message.
  * Adding another APT call to download all packages before any installation happens.

* **Option 2 "Implementing a separate `tedge` plugin":** Leads to duplicated code:
  * Almost everything of the existing APT plugin's code is needed also for the separate plugin that manages module type "tedge". 

Conclusion:

Changes in APT plugin required for option 1 are small in comparision to amount of duplicated code for option 2.

Decision: Start with Option 1 (extending the existing APT plugin). If changes in APT plugin get more complex as expected, switch to option 2 can happen easy, but before self-update is released.
