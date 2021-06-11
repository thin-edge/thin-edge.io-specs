# Software management

With software management, a device operator specifies which software modules should run on the IoT devices of a fleet.

This functionality is split between the IoT platform and the device as follows:
* On the IoT platform, devices are grouped into fleets of devices,
  and the device owner specifies which software modules should run on the devices of each fleet.  
* On the device, the __software management agent__ ensures that the software modules running on the device
  are exactly those modules which have been specified in the IoT platform.

This document specifies the thin-edge __software management agent__,
* its interaction with the cloud
* its interaction with the package managers actually used to install the software modules,
* the way it can be extended to interact with different cloud providers,
* the way it can be extended to use different package managers.


## Overview

```mermaid
flowchart TD
   cloud1((C8y Tenant))
   mapper1(C8y Mapper)
   cloud2((Other Cloud))
   mapper2(Cloud Specific Mapper)
   cloud_API[\Software Management API/]
   agent(Software Management Agent)
   plugin_API[/Package Manager Plugin API\]
   plugin1(Debian Plugin)
   plugin2(Apama Plugin)
   plugin3(External Plugin)
   module1[(Debian Packages)]
   module2[(Apama Models)]
   module3[(Specific Modules)]

   cloud2 <--> mapper2
   mapper2 <--> cloud_API

   cloud1 <--> mapper1
   subgraph thin-edge
       mapper1 <--> cloud_API
       cloud_API <--> agent
       agent <--> plugin_API
       plugin_API <--> plugin1
       plugin_API <--> plugin2
   end
   plugin1 <--> module1
   plugin2 <--> module2

   plugin_API <--> plugin3
   plugin3 <--> module3

   click plugin_API "./plugin-api.html"

```


```mermaid
sequenceDiagram
    participant Software Plugin
    participant SM Agent
    participant Cloud Mapper
    participant Cloud
    participant Cloud Operator
    Cloud Mapper->>SM Agent: GetCurrentSoftwareList
    SM Agent->>Software Plugin: plugin-cmd list
    Software Plugin-->>SM Agent: list cmd output
    SM Agent-->>Cloud Mapper: CurrentSoftwareList
    Cloud Mapper->>Cloud: CurrentSoftwareList
    Cloud Operator->>Cloud: Install s1:v1,s3:v3 Uninstall s2:v2
    Cloud->>Cloud Mapper: DesiredSoftwareListOperation
    Cloud Mapper-->>Cloud: DesiredSoftwareListOperation ACK
    Cloud Mapper->>SM Agent: DesiredSoftwareList
    SM Agent->>Software Plugin: plugin-cmd uninstall s2:v2
    Software Plugin-->>SM Agent: uninstall cmd output
    SM Agent->>Software Plugin: plugin-cmd install s1:v1,s3:v3
    Software Plugin-->>SM Agent: install cmd output
    SM Agent-->>Cloud Mapper: DesiredSoftwareListOperationStatus
    Cloud Mapper-->>Cloud: DesiredSoftwareListOperation SUCCESSFUL
```

