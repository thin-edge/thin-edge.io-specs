# Software Management Agent

The software management agent (referred to as `SM Agent` in the rest of this document) is the component that's responsible for the software management operations on a Thin Edge device.
It primarily interacts with a `Cloud Agent` and a `Software Plugin`.

The `Cloud Agent` is the process that's responsible for cloud message mapping as well as other cloud-specific processing logic.
The `Cloud Agent`'s behaviour handling of software management requests/response are describe in detail here[../c8y-mapper-operation-handling.md]

The `Software Plugin` handles the installation/removal of software modules with the help of package manager, when called by the `SM Agent`.
The `Software Plugin` specification is captured in detail here[../plugin-api.md]
# SM Agent Startup

The sequence of operations and message exchanges happening on every startup of the sm-agent (initial startup on `tedge connect`, service restart, device restarts etc).



```mermaid
sequenceDiagram
    participant Software Plugin
    participant SM Agent
    participant Cloud Agent
    alt If SoftwareUpdateOperation in-progress flag found in persistence store
        SM Agent->>Cloud Agent: SoftwareUpdateOperation FAILED
        SM Agent-->>SM Agent: Clear SoftwareUpdateOperation in-progress flag from persistence store
    end

    SM Agent->>Cloud Agent: Get all PENDING operations
    alt If any SoftwareUpdateOperation is PENDING
        Cloud Agent-->>SM Agent: SoftwareUpdateOperation
    end

```

On every startup, sm-agent checks if a `SoftwareUpdateOperation` was in progress before the startup, from its persistent store.
If yes, it means that the sm-agent crashed or the device got restarted while the update operation was in-progress.
As long as we don't support resumption of software update operations, it's better to just mark the last operation failed so that the users can retry.

For now, persisting just a flag that the `SoftwareUpdateOperation` is in-progress is sufficient.
Once we start supporting software update resumption after crashes/restarts, the entire software update list itself will have to be persisted and updated as the operation is being processed.

On startup, the agent also prompts the mapper to send any PENDING operation that it may have received while the agent was not up.
The mapper will respond back with a `SoftwareUpdateOperation` if any were received and PENDING.

# SM Agent Runtime

The SM Agent needs to handle two kinds of requests from the cloud: software list request and software update request.

The sequence of operations on the receipt of a software list request is as follows:

```mermaid
sequenceDiagram
    participant Software Plugin
    participant SM Agent
    participant Cloud Agent
    Cloud Agent-->>SM Agent: SoftwareListRequest
    loop Each Plugin
        SM Agent->>Software Plugin: plugin-cmd list
        Software Plugin-->>SM Agent: list cmd output
    end
    SM Agent->>Cloud Agent: SoftwareListResponse
```


The sequence of operations on the receipt of a software update request is as follows:


```mermaid
sequenceDiagram
    participant Software Plugin
    participant SM Agent
    participant Cloud Agent

    Cloud Agent-->>SM Agent: SoftwareUpdateOperation

    SM Agent->>SM Agent: Persist SoftwareUpdateOperation in-progress
    SM Agent->>Cloud Agent: SoftwareUpdateOperation EXECUTING
    loop Each plugin
        SM Agent->>Software Plugin: plugin-cmd prepare

        loop Each software module to be uninstalled
            SM Agent->>Software Plugin: plugin-cmd uninstall module
            Software Plugin-->>SM Agent: Exit code + stdout/stderr
        end

        loop Each software module to be installed
            SM Agent->>Software Plugin: plugin-cmd install module
            Software Plugin-->>SM Agent: Exit code + stdout/stderr
        end

        SM Agent->>Software Plugin: plugin-cmd finalize
    end

    loop Each plugin
        SM Agent->>Software Plugin: plugin-cmd list
        Software Plugin-->>SM Agent: list cmd output
    end
    SM Agent->>Cloud Agent: SoftwareList

    alt If SoftwareUpdateOperation successful and SoftwareListStatus successful
        SM Agent->>Cloud Agent: SoftwareUpdateOperation SUCCESSFUL
    else
        SM Agent->>Cloud Agent: SoftwareUpdateOperation FAILED
    end

    SM Agent-->>SM Agent: Clear SoftwareUpdateOperation in-progress flag from persistence store
```

The SM Agent will process only one `SoftwareUpdateOperation` at a time.
If a duplicate operation is received while in the middle of processing one operation, the new request will be ignored.
Ignorning is okay as the SM Agent expects to retrieve it later on, after the current operation processing is complete, from the mapper via its PENDING requests queue.
The mapper can choose to persist such PENDING requests on its own if the cloud that it supports doesn't support such queueing.
But, the SM Agent won't persist such requests.


While processing the software update list, all the packages to be uninstalled are processed first,
before installing the ones to be installed as that offers a more predictable behaviour.

While installing/uinstalling the modules one by one, we have the option to either fail-fast as soon as one installation/uninstallation fails or keep track of the failures and continue installing/uninstalling the rest of the software modules.
Fail-fast would be a better choice as in the case of a failure, the user is more likely to retry that operation after making any changes to the original software update list that he prepared.

# Thin Edge JSON Specification for Commands

A topic scheme like `tedge/commands/<component>/<action>/req` is used for inbound operation requests.
The corresponding operation response need to be sent to `tedge/commands/<component>/<action>/res`.

For example, the request to fetch the software list from the agent needs to be sent to `tedge/commands/software/list/req` and the corresponding software list response will be sent to `tedge/commands/software/list/res`.
Similar scheme can be used for other operations as well in future as captured in the following table:

| Operation          | Request Topic                          | Response Topic                         |
| ------------------ | -------------------------------------- | -------------------------------------- |
| Get Software List  | `tedge/commands/software/list/req`     | `tedge/commands/software/list/res`     |
| Software Update    | `tedge/commands/software/update/req`   | `tedge/commands/software/update/res`   |
| Sync Software List | `tedge/commands/software/sync/req`     | `tedge/commands/software/sync/res`     |
| Apply Profile      | `tedge/commands/profile/apply/req`     | `tedge/commands/profile/apply/res`     |
| Get Configuration  | `tedge/commands/configuration/get/req` | `tedge/commands/configuration/get/res` |
| Set Configuration  | `tedge/commands/configuration/set/req` | `tedge/commands/configuration/set/res` |
| Get Log            | `tedge/commands/log/get/req`           | `tedge/commands/log/get/res`           |
| Restart  device    | `tedge/commands/control/restart/req`   | `tedge/commands/control/restart/res`   |
| Remote  connect    | `tedge/commands/control/connect/req`   | `tedge/commands/control/connect/res`   |

Having such dedicated topics for each command enables Thin Edge components to selectively subscribe to only the commands that they're interested in.
If one component wants to subscribe to all commands for a single component like `software`, it can still subscribe to `tedge/commands/<component>/+/req`.
If one component wants to subscribe to all commands, then it can even subscribe to `tedge/commands/+/+/req`.

## Ordering of operations along multiple topics

Since MQTT doesn't guarantee ordered delivery of messages across different topics, the ordering of actions for a single component,
or even the ordering of actions between different components will have to be controlled by the publisher, which is the `Cloud Agent`.
When strict ordering is required between commands, like a software update command followed by a device restart command,
the `Cloud Agent` needs to issue the software update request first and wait for its response and only then issue the device restart request.
It can also send unordered commands like a log request or remote control parallelly, even when some other ordered commands are being executed.

# Thin Edge JSON Specification for Software Management Commands
## Get PENDING Operations

Topic to publish the request to: `tedge/commands/pending/req`

There's no payload to send.

This request is not a specific one to retrive only the PENDING `SoftwareUpdateOperation`s,
but rather a prompt to the mapper to send all the PENDING requests to their appropriate `inboud/#` topics.
The mapper, on receipt of this request will publish any PENDING operations to the designated topics like `tedge/commands/software/list/req`, `tedge/commands/configuration/set/req` etc based on which all operations are PENDING.
If there are no PENDING operations, the mapper won't send any response.

## Software List Operation

### Thin Edge JSON Software List Request


Topic to publish the software list request to: `tedge/commands/software/list`

Request payload: 

```json
{
    "id": 123
}
```

Some unique id must be generated by the requestor and this `id` is sent back in the response for correlation.

### Thin Edge JSON Software List Response

Topic to subscribe for the software list response: `tedge/outbound/software/list`

Payload format:

```json
{
    "id": 123,
    "status": "SUCCESSFUL",
    "list": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0",
                },
                {
                    "name": "collectd",
                    "version": "5.7"
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "nginx",
                    "version": "1.21.0",
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6",
                }
            ]
        }
    ]
}
```

**Payload fields:**

In the top-level array, there will be one entry each for every plugin on the device, if the plugin reports a non-empty software list, when queried for one.

* `id` is used to correlate any response from the mapper while processing the software list. If the mapper fails to process the list, the error will published 
* `type` captures the type of software module that's being reported in the list.
  It will be the name of the plugin that reports this list.
  It can be optional and can be empty for the default software module type of the device, if a default plugin is configured on the device.
* `list` is an array of software modules represented as JSON objects. This field is mandatory.
* `name` in the software module JSON captures the name of the software module, which is mandatory.
* `version` in the software module JSON captures the name of the software module, which is optional.

If fetching the software list had failed, the reponse would have indicated a failure as follows:

```json
{
    "id": 123,
    "status": "FAILED",
    "reason": "Request timed-out"
}
```

## Software Update Operation

### Thin Edge JSON Software Update Request

Topic to subscribe to: `tedge/commands/software/update`

Payload format:

```json
{
    "id": 123,
    "software-update": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0",
                    "action": "install"
                },
                {
                    "name": "collectd",
                    "version": "5.7",
                    "url": "https://collectd.org/download/collectd-tarballs/collectd-5.12.0.tar.bz2",
                    "action": "install"
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "nginx",
                    "version": "1.21.0",
                    "action": "install"
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6",
                    "action": "remove"
                }
            ]
        }
    ]
}
```

### Thin Edge JSON Software Update Response

Once a software-update operation is received, it must be acknowledged with an EXECUTING response, followed by a SUCCESSFUL or FAILED response.

Topic to subscribe for the software update response: `tedge/commands/software/update/res`

#### Executing Status Payload

```json
{
    "status": "EXECUTING"
}
```

#### Successful Status Payload

```json
{
    "status": "SUCCESSFUL",
    "current-software-list": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0",
                },
                {
                    "name": "collectd",
                    "version": "5.7"
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "nginx",
                    "version": "1.21.0",
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6",
                }
            ]
        }
    ]
}
```

Sending the current software list along with the status will help the cloud providers to show the most up-to-date software list after an update was performed, which would include any extra depepndencies that got installed/removed as part of the update.

#### Failed Status Payload

```json
{
    "status":"FAILED",
    "reason":"Partial failure: Couldn't install collectd and nginx",
    "current-software-list": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0",
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "mongodb",
                    "version": "4.4.6",
                }
            ]
        }
    ]
}
```

Sending the current software list along with the status even in the case of a failure will help the cloud providers to show the most up-to-date software list, especially in the case of partial failures, which would contain the modules and dependencies that got installed, even though the overall update failed.
