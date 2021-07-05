
Introduction
============
That document specifies the "Software Management" use-case "Update SW List". 
The purpose of the use-case is to install new software or removing existing from
Cloud side on the device.
For more information about the context and other use-cases see "Software 
Management" use-case specification (TODO: !!add link!!).

The sequence diagram below indentifies all involved components as well as all 
message flows between those. Thereby the components are represented as objects,
starting from the very right side with the actor "Cloud" reaching to the very left side with the
actor "Package Manager". Further details (if any) are defined in linked 
sub-specifications.

Further explanations about diagram's elements it's interpretation could be found below the diagram.


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
&nbsp;| More details about the interface between SM mapper and Cloud are defined in: TODO: !!add link to "contract between mapper and C8Y".

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
&nbsp;| More details about the interface between SM mapper and SM agent are defined in: TODO: !!add link to "interface specification between SM agent and SM mapper".

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
&nbsp;| (2) Agent iterates over the sw-list that is contained in the SM mappers request. <br/><br/> TO-BE-DECIDED-#1: To be decided whether delta-comparision between current sw-list and sw-list contained in the Mappers update request shall be done here by the agent.<br/><br/> TO-BE-DECIDED-#2: Order of sw-list shall be kept stable to enable Cloud to dictate the oder of processing, e.g. allow managing dependencies or to process all items of a Package Manager at once.<br/>
&nbsp;| For each particular package the command ("install" or "remove") is send to responsible Package Manager Plugin. "Command" and "responsible Package Manager Plugin" are determined based on information that are part of the sw-list for each package.
&nbsp;|&nbsp;
&nbsp;| More details about the interface between SM mapper and SM agent are defined in: TODO: !!add link to "interface specification between SM agent and SM mapper".

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
&nbsp;| More details about the interface between SM mapper and Package Manager Plugin are defined in: TODO: !!add link to "interface specification between SM agent and SM mapper".

Open Topcis
===========

(1) All unhappy paths missing. Need to be specified.

(2) Open Decision about delta-comparision between current sw-list and sw-list. See "TO-BE-DECIDED-#1" above.

(3) Open Decision about stable order in SW-list. See "TO-BE-DECIDED-#2" above.
