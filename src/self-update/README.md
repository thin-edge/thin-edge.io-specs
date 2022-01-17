# Introduction

That document specifies the feature self-update for thin-edge.

The self-update allows a Cloud Operator to update one thin-edge version that is installed on a device to a newer version, via the Cloud connection. The feature is referred as self-update cause that update process is managed by thin-edge own's Software Management.

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

### Run-Time Behaviour
* Self-Update is provided based on usual thin-egde's Software Management.
  * A plugin for module type "tedge-update" specifically handles thin-edge self-update.
* The plugin uses Debian Package Manager APT to upgrade requested thin-edge packages.
* When executing APT the plugin will disable auto dependency solving.
  * This is to avoid installing more than the requested packages to reduce risc for potential failures.
* The plugin shall take all requested packages at once from SM Agent (i.E. Plugin API command "update-list" shall be used).
* The plugin follows the contract of the Software Management Plugin API, with the exception of the way to report final result to the SM Agent:
  * As the new started SM Agent will be disconnected from tedge-update plugin's child-process, the plugin will store potential `stdout` and the final exit status to some persistent file.
  * The SM Agent will pickup that file and generates according update response for the mapper.
* Potentially the spawned plugin-process need to detouched it-self from parent process SM Agent to avoid any impact when SM Agent process is stopped by APT upgrade procedure.

### Deployment
* The "tedge-update" plugin must not be contained in any thin-edge package that is updated by that plugin (it cannot overwrite it-self while running).
  * The "tedge-update" plugin shall be delivered as individual thin-edge package.
  * That "tedge-update" plugin package can be updated using Software Management as any other Software Module (e.g. with normal "APT" module type).

### Robustness in Packages' installation logic
* Use of external tools (e.g. grep, sed, ...) shall be avoided in package's installation logic. Instead Rust code/libraries shall be used.
  * That is to reduce external dependencies and so reduce risc for failures.
* At very beginning of the installation/removal logic all required/touched external ressources (e.g. config files as "mosquitto.conf", external tools (if any left), ...) shall be checked for availibility/accessibility.
* If one external ressource is not available installation/removal procedure shall fail before any action was performed.

### Robustness in Self-Update processing
* When "tedge-update" plugin was started, no other update request for any other plugin must be in progress or started by the SM Agent.
  * Reason: APT will restart the SM Agent during upgrade, what would result into half-executed update procedures of other parallel running plugins and could lead to corrupted system behaviour.
  * It would be valid to execute any update requests for other plugins when tedge-update has completely finished. 
    It would be also valid to reject any other update request and report an error to Cloud while tedge upgrade is running. 
* The plugin shall download all requested packages before any upgrade starts.
  * This is to avoid any communication breakdown during upgrade processing.
  * If one download fails, no package at all shall be upgrade and error shall reported.


