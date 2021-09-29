# C8Y Mapper Operation Handling

*!!ATTENTION!! We support only `c8y_SoftwareUpdate` in the release `0.3`. 
Ignore `c8y_DeviceProfile` for now.*

In this page, we focus on the contract between C8Y Mapper and C8Y Cloud. 
If you want to know the mapping rules, 
please refer to [Thin Edge JSON Mapping to/from C8Y](./thin-edge-json-mapping-to-from-c8y.md).

## Flow at SM Agent Startup

```mermaid
sequenceDiagram
    participant SM Agent
    participant C8Y Mapper
    participant C8Y Cloud

    alt If SM Agent reports failure of last SoftwareUpdateOperation on a startup
      SM Agent ->> C8Y Mapper: Operation status FAILED + current SoftwareList
      C8Y Mapper ->> C8Y Cloud: SmartREST 502: Update operation status to FAILED
      C8Y Mapper ->> C8Y Cloud: SmartREST 116: Send current c8y_SoftwareList
    end

    SM Agent ->> C8Y Mapper: Declare SoftwareUpdate and SoftwareList capability
    C8Y Mapper ->> C8Y Cloud: SmartREST 114: Send c8y_SoftwareUpdate as SupportedOperations
    
    alt If receiving both SoftwareUpdate and SoftwareList capabilities
      C8Y Mapper ->> SM Agent: Software List Request
      SM Agent -->> C8Y Mapper: Software List
      C8Y Mapper ->> C8Y Cloud: SmartREST 116: Send current c8y_SoftwareList
      
      C8Y Mapper ->> C8Y Cloud: SmartREST 500: Get PENDING operations
      C8Y Cloud -->> C8Y Mapper: SmartREST 528: SoftwareUpdate operation and others
      Note right of C8Y Mapper: the following flow is the same as the flow in runtime
    end
```

|ID|Description|Example Payload|Type|
|---|---|---|---|
|[502](https://cumulocity.com/guides/device-sdk/mqtt/#a-name502set-operation-to-failed-502a)|Set operation to FAILED|`502,c8y_SoftwareUpdate,"Permission denied"`|Publish|
|[116](https://cumulocity.com/guides/device-sdk/mqtt/#a-name116set-software-list-116a)|Set software list|`116,software1,version1,url1,software2,version2,url2`|Publish|
|[114](https://cumulocity.com/guides/device-sdk/mqtt/#a-name114set-supported-operations-114a)|Set supported operations|`114,c8y_SoftwareUpdate`|Publish|
|[500](https://cumulocity.com/guides/device-sdk/mqtt/#a-name500get-pending-operations-500a)|Get PENDING operations|`500`|Publish|
|[528](https://cumulocity.com/guides/device-sdk/mqtt/#a-name528update-software-528a)|Update Software|`528,external_id,software1,version1,url1,install,software2,version2,url2,delete`|Subscribe|

This flow is consists of 4 parts.
1. Report operation failed to C8Y Cloud.
2. Receive device capabilities from SM Agent and translate as `C8Y_SupportedOperatons` and report them to C8Y Cloud.
3. Trigger a Set Software List request to SM Agent as a response of receiving Software List capability. 
4. Trigger a Get PENDING operations request to C8Y Cloud as a response of receiving Software Update capability.

Both "SM Agent is up but C8Y Mapper is down" and "SM Agent is down but C8Y Mapper is up" cases must be considered here.
Namely, the device capabilities must be delivered to C8Y Mapper in any case.

Other notes:
- Collecting all device's capabilities (e.g. SoftwareUpdate, Restart, etc.) are required 
  so that the mapper sends SmartREST `114` with all necessary supported operations.
- Collecting all capabilities is also required for all child device plugins, to create
  according child devices in the Cloud (if not yet exist) using automatic device creation
  on data push. Thereby it is enough to push capabilities to a device that not yet exist 
  to create that device (using SmartRest `114`).
- SM Agent might publish more capabilities than C8Y cloud supports. 
  In this case, the mapper doesn't need to subscribe the unsupported capability topics.
- `c8y_SoftwareUpdate` is supported in c8y version 10.7 and onwards.
- C8Y mapper can consider that the SM Agent is ready for a new operation 
  after the agent publishes device's capabilities.
- SmartREST `500` returns all the operations in the status `PENDING`.
- SmartREST `500` may return not only `528`.

## Flow in runtime phase for `c8y_SoftwareUpdate` operation

```mermaid
sequenceDiagram
    participant SM Agent
    participant C8Y Mapper
    participant C8Y Cloud
    
    C8Y Cloud ->> C8Y Mapper: SmartREST 528: SoftwareUpdate operation and others
    C8Y Mapper ->> C8Y Mapper: Put operations to FIFO queue
    C8Y Mapper ->> C8Y Mapper: Wait until the SM Agent completes processing the last SoftwareUpdate operation(if any)
    C8Y Mapper ->> C8Y Mapper: Pick up the oldest c8y_SoftwareUpdate operation

    C8Y Mapper ->> SM Agent: Software Update Request
    
    SM Agent ->> C8Y Mapper: Operation status EXECUTING
    C8Y Mapper ->> C8Y Cloud: SmartREST 501: Update operation status to EXECUTING
        
    alt software update successful
        SM Agent ->> C8Y Mapper: Operation status SUCCESSFUL + current SoftwareList
        alt the size of software list is small enough
            C8Y Mapper ->> C8Y Cloud: SmartREST 116: Send current c8y_SoftwareList
            C8Y Mapper ->> C8Y Cloud: SmartREST 503: Update operation status to SUCCESSFUL
        else the size of software list is above the threshold 
            C8Y Mapper ->> C8Y Cloud: SmartREST 502: Update operation status to FAILED
        end
    else software update failed
        SM Agent ->> C8Y Mapper: Operation status FAILED + current SoftwareList
        C8Y Mapper ->> C8Y Cloud: SmartREST 116: Send current c8y_SoftwareList
        C8Y Mapper ->> C8Y Cloud: SmartREST 502: Update operation status to FAILED
    end
```

|ID|Description|Example Payload|Type|
|---|---|---|---|
|[528](https://cumulocity.com/guides/device-sdk/mqtt/#a-name528update-software-528a)|Update Software|`528,external_id,software1,version1,url1,install,software2,version2,url2,delete`|Subscribe|
|[501](https://cumulocity.com/guides/device-sdk/mqtt/#a-name501set-operation-to-executing-501a)|Set operation to EXECUTING|`501,c8y_SoftwareUpdate`|Publish|
|[116](https://cumulocity.com/guides/device-sdk/mqtt/#a-name116set-software-list-116a)|Set software list|`116,software1,version1,url1,software2,version2,url2`|Publish|
|[503](https://cumulocity.com/guides/device-sdk/mqtt/#a-name503set-operation-to-successful-503a)|Set operation to SUCCESSFUL|`503,c8y_SoftwareUpdate`|Publish|
|[502](https://cumulocity.com/guides/device-sdk/mqtt/#a-name502set-operation-to-failed-502a)|Set operation to FAILED|`502,c8y_SoftwareUpdate,"Permission denied"`|Publish|

Note:
- C8Y cloud might publish `c8y_SoftwareUpdate`(`528`) and also other operations.
- The mapper has responsibility to keep all received PENDING operations in FIFO queue.
- The mapper considers that SM Agent is ready for receiving a new operation 
  either **when it receives Operation status SUCCESSFUL/FAILED + current SoftwareList** 
  or **when it receives device capability (at agent startup only)**.
- SM Agent can process only one Software Update operation at one time. 
  Therefore, c8y Mapper should pick up the oldest c8y_SoftwareUpdate operation.
- C8Y UI blocks to create more than one `c8y_SoftwareUpdate` operation at the same time. 
  However, still user can create more than one operation from REST API.
  Also for each child-device a parallel operation can be triggered from C8Y UI and REST API.
- If one operation includes a couple of packages updates, and if one of those package failed, 
  we have to send `FAILED`.
