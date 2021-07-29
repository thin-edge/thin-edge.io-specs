
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


![Sequence Diagram Update SW-list](images/uc-update-swlist.svg)

Components
==========

Name | "Cloud" 
--- | --- 
**Purpose** | The cloud triggers the "update swlist" operation, and finally receives it's result. The trigger request contains the sw-list to the Cloud mapper on the device.
**Sequence** | (1) The Cloud sends a request to start an update to the Cloud mapper on the device. The request contains the sw-list.
&nbsp;| (2) The cloud gets feedback from Cloud mapper (status "executing") when update was started.
&nbsp;| (3) The cloud gets feedback from Cloud mapper (status "successful") when update was sucessfully processed.
&nbsp;|&nbsp;
**Further Spec**| Spec about details of interface between Cloud and Cloud Mapper is under construction. See [Ticket CIT-439](https://cumulocity.atlassian.net/browse/CIT-439)

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
  
Name | "Cloud mapper" 
--- | --- 
**Purpose** | The Cloud mapper abstracts the specific Cloud to the SM agent. For each Cloud a specific "Cloud-mapper" might required to be implemented. In that spec the Cloud mapper for Cumulocity is outlined.
**Sequence** | (1) When the Cloud mapper receives an update request from Cloud it forwards the request to the SM agent. If the sw-list contained in the Cloud request does not match the SM agent's interface translation has to be done by the Cloud mapper.
&nbsp;| (2) SM agent sends feedback to cloud (status "executing"). 
&nbsp;|&nbsp;
**Further Spec**| Spec about details of interface between Cloud Mapper and SM Agent is under construction. See [Ticket CIT-411](https://cumulocity.atlassian.net/browse/CIT-411)

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
  
Name | "SM agent"
--- | --- 
**Purpose** | The SM agent is the core component in thin-edge that manages the Software Management functionality.
**Sequence** | (1) On incoming update request from the Cloud mapper the prepare command is sent to all Package Manager Plugins.
&nbsp;| (2) The SM agent splits sw-list into separate lists per package-type (e.g. "sw-list_pkgType1", "sw-list_pkgType2", ...).
&nbsp;| (3) The command exec-list is sent with list "sw-list_pktType1" as argument to the Package Manager Plugin for package-type 1. That command allows the plugin to handle the whole list in one command. On the other side the plugin is free to just return \<not-implemeneted\> and instead use a 2nd option provided by agent. The 2nd Option feeds plugin package-by-package, that is more simple but less flexible. Therefore follow next step below.
&nbsp;| (4) If exec-list has return \<not-implemeneted\> agent iterates over "sw-list_pktType1". For each particular package the command ("install" or "remove") is send to responsible Package Manager Plugin. The according "command" is determined based on information that is part of the sw-list for each package.
&nbsp;| Same steps (3 and 4 from above) will be executed for each splitted list "sw-list_pktType\<i\>" and according Package Manager Plugin.
&nbsp;| **TO-BE-DECIDED-#1:** Some delta comparision shall occur at some place, to avoid installing already installed packages and avoid removing not existing ones. To be decided whether delta-comparision shall occur in SM Agent or Package Manager Plugin. (?) <br/><br/>
&nbsp;|&nbsp;
**Further Spec**| Spec about details of SM Agent and it's interface to Cloud Mapper is under construction. See [Ticket CIT-411](https://cumulocity.atlassian.net/browse/CIT-411) <br/>Spec about interface between SM agent and Package Manager Plugin see [src/software-management/plugin-api.md](https://github.com/thin-edge/thin-edge.io-specs/blob/main/src/software-management/plugin-api.md)

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
&nbsp;| (2) Receives command "exec-list" including sw-list with all packages for according package-type. The sw-list contains also operations per each package as contained in orignial sw-list from Cloud. The Package Manager Plugin can use that sw-list to instruct the according Package Manager in a specific order to reach intented software configuration. If the order is not relevant for the specific Package Manager the Package Manager Plugin is free to return \<not-implemented\> here. Then SM agent will feed the plugin package-by-package (see next step below), that might allow a much more simple plugin implementation. 
&nbsp;| NOTE: For more Details (e.g. about reason for feeding with one list vs. feeding package-by-package) and how to encode \<not-implemented\> see Package Manager Plugin specififaction referenced below. PLEASE NOTE: Details "feeding with one list" are not yet part of the Package Manager Plugin specififaction, but will be added soon.
&nbsp;| (3) If plugin has return \<not-implemented\> in step 2 above, plugin receives package-by-package command install or removed including Package name from SM agent, and instructs according Package Manager to do so.
&nbsp;| (4) Receives finalize command from SM agent to do some finish action, if any.
&nbsp;|&nbsp;
**Further Spec**| Spec about interface between SM agent and Package Manager Plugin see [src/software-management/plugin-api.md](https://github.com/thin-edge/thin-edge.io-specs/blob/main/src/software-management/plugin-api.md)

Open Topcis
===========

(1) All unhappy paths missing. Need to be specified.

(2) Open Decision about delta-comparision between current sw-list and sw-list. See "TO-BE-DECIDED-#1" above.

(3) Open Decision about stable order in SW-list. See "TO-BE-DECIDED-#2" above.
