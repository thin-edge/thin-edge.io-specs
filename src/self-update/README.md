# Introduction


Thin-edge self-update allows a Cloud Operator to update one thin-edge version that is installed on a device to a newer version, via the Cloud connection. The feature is referred as self-update cause that update process is managed by thin-edge own's Software Management.


# Requirements

* Be default self-update shall use Debian Package Manager APT.
  * NOTE: APT Package Manager provides a very basic rollback mechanism that still can fail (e.g. due to some unexpected device state or power-loss).
* Device Owner shall be able to use a power-fail safe filesystem and transaction-safe package manager (instead of APT) - if the device support any.
  * Therefore an extensible plugin shall be provided.
* In thin-edge package's installation logic no external tools (e.g. grep, sed, ...) shall be used. Instead Rust code/libraries shall be used.
  * This is to reduce external dependencies and so reduce risc for failures due missing tools or wrong version of tools.

# Expected Pre-Conditions 

The device and it's thereon installed thin-edge has to follow precondition below to allow successful self-update:

1) The current (old) thin-edge on the device must be installed with usual thin-edge installation, or at leat as usual thin-edge installation would do. That is to assure new thin-edge version that is about to be installed has all external and internal files, folders and resources (e.g. config files) available.
2) Deployment of thin-edge and mosquitto must not be changed manually after initial installation (e.g. moving/renaming installed files, moving/renaming configuration files used by thin-edge, renaming users/groups created during installation, ...).

3) Package Dependencies need to be solved using SW Management before self-update. This is since self-update will disable auto-dep solving to reduce additional risk during self-update.

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
  * For all packages with some URL given in the update request the SM Agent manages download before plugin start, and provides local file paths to the plugin.<br/>
    TODO: What happens if one package download fails in SM agent? Does plugin maybe need to check if all files are locally available?
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

# Options for more Robustness
NOTE: Options below to be decided and addressed in a 2nd drop.

### Robustness in Packages' installation logic
* Use of external tools (e.g. grep, sed, ...) shall be avoided in package's installation logic. Instead Rust code/libraries shall be used.
  * That is to reduce external dependencies and so reduce risc for failures.
* At very beginning of the installation/removal logic all required/touched external ressources (e.g. config files as "mosquitto.conf", external tools (if any left), ...) shall be checked for availibility/accessibility.
* If one external ressource is not available installation/removal procedure shall fail before any action was performed.

### Robustness in Self-Update processing
* When "tedge" plugin was started, no other update request for any other plugin must be in progress or started by the SM Agent.
  * Reason: APT will restart the SM Agent during upgrade, what would result into half-executed update procedures of other parallel running plugins and could lead to corrupted system behaviour.
  * It would be valid to execute any update requests for other plugins when tedge has completely finished. 
    It would be also valid to reject any other update request and report an error to Cloud while tedge update is running. 
* When executing APT the plugin will disable auto dependency solving.
  * This is to avoid installing more than the requested packages to reduce risc for potential failures.
* SM Agent shall fail when plugin for "tedge" does not support "update_list".
  * This is since an update request will fail when there is more than one package and the agent is concerned
* SM Agent shall fail when update request with packages for module type "tedge" contains also other module types.
