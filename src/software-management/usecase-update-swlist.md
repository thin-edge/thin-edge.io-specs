
Introduction
============
That document specifies the "Software Management" use-case "Update SW List". 
The purpose of the use-case is to install new software or removing existing from
Cloud side on the device.
For more information about the context and other use-cases see "Software 
Management" use-case specification in: [src/software-management/README.md](/src/software-management/README.md)

The sequence diagram below indentifies all involved components as well as all 
message flows between those. Thereby the components are represented as objects,
starting from the very right side with the actor "Cloud" reaching to the very left side with the
actor "Package Manager". Further details (if any) are defined in linked 
sub-specifications.

Further explanations about diagram's elements or it's interpretation could be found below the diagram.


![Sequence Diagram Update SW-list](/src/software-management/uc-update-swlist.png)

Components
==========

Name | "Cloud" 
--- | --- 
**Purpose** | The cloud triggers the "update swlist" operation, and finally receives it's result. The trigger request contains the sw-list to the SM mapper on the device.
**Sequence** | (1) The Cloud sends a request to start an update to the SM mapper on the device. The request contains the sw-list.
&nbsp;| (2) The cloud gets feedback from SM mapper (status "executing") when update was started.
&nbsp;| (3) The cloud gets feedback from SM mapper (status "successful") when update was sucessfully processed.
&nbsp;|&nbsp;
**Further Spec**| Spec about details of interface between Cloud and SM Mapper is under construction. See [Ticket CIT-439](https://cumulocity.atlassian.net/browse/CIT-439)

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
  
Name | "SM mapper" 
--- | --- 
**Purpose** | The SM mapper abstracts the specific Cloud to the SM agent. For each Cloud a specific "SM-mapper" might required to be implemented. In that spec the SM mapper for Cumulocity is outlined.
**Sequence** | (1) When the SM mapper receives an update request from Cloud it forwards the request to the SM agent. If the sw-list contained in the Cloud request does not match the SM agent's interface translation has to be done by the SM mapper.
&nbsp;| (2) SM agent sends feedback to cloud (status "executing"). 
&nbsp;|&nbsp;
**Further Spec**| Spec about details of interface between SM Mapper and SM Agent is under construction. See [Ticket CIT-411](https://cumulocity.atlassian.net/browse/CIT-411)

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
  
Name | "SM agent"
--- | --- 
**Purpose** | The SM agent is the core component in thin-edge that manages the Software Management functionality.
**Sequence** | (1) On incoming update request from the SM mapper the prepare command is send to all Package Manager Plugins.
&nbsp;| (2) Agent iterates over the sw-list that is contained in the SM mappers request. <br/><br/> **TO-BE-DECIDED-#1:** Some delta comparision shall occur at some place, to avoid installing already installed packages and avoid removing not existing ones. To be decided whether delta-comparision shall occur in SM Agent or Package Manager Plugin. (?) <br/><br/> **TO-BE-DECIDED-#2:** Two options for ordering sw-list: Option 1: Keeping order inside sw-list from Cloud to enable Cloud to dictate the order of processing. That would allow the cloud to manage package dependencies, or to decide to process all packages of one Package Manager at once. Option 2: Hardcoding some as best expected order into the agent (e.g. to process package type by package type to avoid potential swapping between multiple triggered Package Managers). (?)<br/>
&nbsp;| For each particular package the command ("install" or "remove") is send to responsible Package Manager Plugin. "Command" and "responsible Package Manager Plugin" are determined based on information that are part of the sw-list for each package.
&nbsp;|&nbsp;
**Further Spec**| Spec about details of SM Agent and it's interface to SM Mapper is under construction. See [Ticket CIT-411](https://cumulocity.atlassian.net/browse/CIT-411) <br/>Spec about interface between SM agent and Package Manager Plugin see [src/software-management/plugin-api.md](https://github.com/thin-edge/thin-edge.io-specs/blob/main/src/software-management/plugin-api.md)

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
  
Name | Package Manager Plugin
--- | --- 
**Purpose** | To abstract the device specific Package Manager (e.g. Debian's "APT", Canonical's "Snap" or Red Hat’s “RPM”).
**Sequence** | (1) Receives prepare command from SM agent to do some prepare action, if any.
&nbsp;| (2) Receives command install or removed including Package name from SM agent, and instructs according Package Manager to do so.
&nbsp;| (3) Receives finalize command from SM agent to do some finish action, if any.
&nbsp;|&nbsp;
**Further Spec**| Spec about interface between SM agent and Package Manager Plugin see [src/software-management/plugin-api.md](https://github.com/thin-edge/thin-edge.io-specs/blob/main/src/software-management/plugin-api.md)

Open Topcis
===========

(1) All unhappy paths missing. Need to be specified.

(2) Open Decision about delta-comparision between current sw-list and sw-list. See "TO-BE-DECIDED-#1" above.

(3) Open Decision about stable order in SW-list. See "TO-BE-DECIDED-#2" above.
