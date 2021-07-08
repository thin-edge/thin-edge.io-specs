# Contract between Mapper and C8Y Flow for 0.3

*!!ATTENTION!! We support only `c8y_SoftwareUpdate` in the release `0.3`. Ignore `c8y_DeviceProfile` for now.*

With SmartREST 2.0, the topic to publish: `s/us` and the topic to subscribe: `s/ds`.

## Flow at Mapper Start-up

```mermaid
sequenceDiagram
    participant SM Agent
    participant C8Y Mapper
    participant C8Y Cloud

    C8Y Mapper ->> SM Agent: Software List Request
    SM Agent -->> C8Y Mapper: Software List
    C8Y Mapper ->> C8Y Cloud: SmartREST 116: Send current c8y_SoftwareList
```

Links and example message payloads:

|ID|Description|Example Payload|Type|
|---|---|---|---|
|[114](https://cumulocity.com/guides/device-sdk/mqtt/#a-name114set-supported-operations-114a)|Set supported operations|`114,c8y_SoftwareUpdate`|Publish|
|[116](https://cumulocity.com/guides/device-sdk/mqtt/#a-name116set-software-list-116a)|Set software list|`116,software1,version1,url1,software2,version2,url2`|Publish|

Note:
- SmartREST `114` must contain all necessary supported operations. We can't send one by one.
- `c8y_SoftwareUpdate` is supported in c8y version 10.7 and onwards.
- Request SoftwareList from c8y Mapper should have a retain flag. In case that SM Agent is down when c8y Mapper starts up, SM Agent can consume the request.

## Flow in runtime phase for `c8y_SoftwareUpdate` operation

```mermaid
sequenceDiagram
    participant SM Agent
    participant C8Y Mapper
    participant C8Y Cloud

    SM Agent ->> C8Y Mapper: Get PENDING SoftwareUpdate operations Request
    C8Y Mapper ->> C8Y Cloud: SmartREST 500: Get PENDING operations

    C8Y Cloud -->> C8Y Mapper: SmartREST 528: SoftwareUpdate operation
    
    C8Y Mapper ->> C8Y Mapper: Pick up the oldest c8y_SoftwareUpdate operation
    alt Translation from SmartREST to ThinEdgeJSON failed
        C8Y Mapper ->> C8Y Cloud: SmartREST 502: Update operation status to FAILED
    else Translation from SmartREST to ThinEdgeJSON successful
        C8Y Mapper ->> SM Agent: Software Update Request
    
        SM Agent ->> C8Y Mapper: Operation status EXECUTING
        C8Y Mapper ->> C8Y Cloud: SmartREST 501: Update operation status to EXECUTING
        
        alt software update successful
            SM Agent ->> C8Y Mapper: Operation status SUCCESSFUL + current SoftwareList
            C8Y Mapper ->> C8Y Cloud: SmartREST 116: Send current c8y_SoftwareList
            alt Sending c8y_SoftwareList successful
                C8Y Mapper ->> C8Y Cloud: SmartREST 503: Update operation status to SUCCESSFUL
            else 
                C8Y Mapper ->> C8Y Cloud: SmartREST 502: Update operation status to FAILED
            end
        else software update failed
            SM Agent ->> C8Y Mapper: Operation status FAILED + current SoftwareList
            C8Y Mapper ->> C8Y Cloud: SmartREST 116: Send current c8y_SoftwareList
            C8Y Mapper ->> C8Y Cloud: SmartREST 502: Update operation status to FAILED
        end
    end
```

Links and example message payloads:

|ID|Description|Example Payload|Type|
|---|---|---|---|
|[500](https://cumulocity.com/guides/device-sdk/mqtt/#a-name500get-pending-operations-500a)|Get PENDING operations|`500`|Publish|
|[528](https://cumulocity.com/guides/device-sdk/mqtt/#a-name528update-software-528a)|Update Software|`528,external_id,software1,version1,url1,install,software2,version2,url2,delete`|Subscribe|
|[501](https://cumulocity.com/guides/device-sdk/mqtt/#a-name501set-operation-to-executing-501a)|Set operation to EXECUTING|`501,c8y_SoftwareUpdate`|Publish|
|[116](https://cumulocity.com/guides/device-sdk/mqtt/#a-name116set-software-list-116a)|Set software list|`116,software1,version1,url1,software2,version2,url2`|Publish|
|[503](https://cumulocity.com/guides/device-sdk/mqtt/#a-name503set-operation-to-successful-503a)|Set operation to SUCCESSFUL|`503,c8y_SoftwareUpdate`|Publish|
|[502](https://cumulocity.com/guides/device-sdk/mqtt/#a-name502set-operation-to-failed-502a)|Set operation to FAILED|`502,c8y_SoftwareUpdate,"Permission denied"`|Publish|

Note:
- SmartREST `500` returns the all operations in the status `PENDING`.
- SmartREST `500` may return not only `528`. The mapper should ignore other numbers.
- SM Agent can proceed only one Software Update operation at one time. Therefore, c8y Mapper should pick up the oldest c8y_SoftwareUpdate operation that comes by `500`.
- There is a possibility that the assumption of `type` from SmartREST `528` to ThinEdgeJSON translation failed. In this case, mapper should send `FAILED` to C8Y cloud.
- C8Y UI blocks to create more than one `c8y_SoftwareUpdate` operation at the same time. However, still user can create more than one operation from REST API.
- If one operation includes a couple of packages updates, and if one of those package failed, we have to send `FAILED`.

## Declare device capability [Under the discussion]

```mermaid
sequenceDiagram
    participant SM Agent
    participant C8Y Mapper
    participant C8Y Cloud

    C8Y Mapper ->> SM Agent: Inquire device's capabilities
    SM Agent -->> C8Y Mapper: Inform SoftwareUpdate capability
    
    C8Y Mapper ->> C8Y Mapper: Collect all device's capabilities
    C8Y Mapper ->> C8Y Cloud: SmartREST 114: Send c8y_SoftwareUpdate (or more) as SupportedOperations
```

Links and example message payloads:

|ID|Description|Example Payload|Type|
|---|---|---|---|
|[114](https://cumulocity.com/guides/device-sdk/mqtt/#a-name114set-supported-operations-114a)|Set supported operations|`114,c8y_SoftwareUpdate`|Publish|
