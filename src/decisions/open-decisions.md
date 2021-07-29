# Open Decisions

That document lists open decisions in scope of thin-edge.

Each table below outlines an open decision to be made. 

## Open Decisions about Software Management


**Decision-001**
----------------

**Problem Statement:** When shall current sw-list reported to the cloud?

**Reference:** Use Case Diagram "SW Management" (see reference "TO-BE-DECIDED-#1"): https://github.com/thin-edge/thin-edge.io-specs/blob/0cc561632947220cbf5d8a204be8a4a9ad9abb5a/src/software-management/README.md

**Option #1**
- **Description:**  Just on Device/Agent start<br/>
- **Implications:**  TBD<br/>

**Option #2**
- **Description:** Periodically? (Last might capture also manually installed packages).<br/>
See ref: https://app.archbee.io/docs/9iGX1hbDjwAeMfyO9A3YE/coxr9CuTWSjk0eE1Nzgoj <br/>
-> Section: "Update Profile Operation"<br/>
-> Snippet: "As part of the above use cases, this use cases extracts the current software list from the device. [...] Additionally, this use cases could be called periodically by e.g. a cron job.".<br/>
- **Implications:** TBD <br/>

**Comment:** Due to agent's request-response-model periodically trigger can be realized outside of agent.

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;


**Decision-002**
----------------

**Problem Statement:** Some delta comparision shall occur at some place, to avoid installing already installed packages and avoid removing not existing ones. <br/>
To be decided whether delta-comparision shall occur in SM Agent or Package Manager Plugin.

**Reference:** Sequence Diagram "Update SW-list" (see reference "TO-BE-DECIDED-#1"): https://github.com/thin-edge/thin-edge.io-specs/blob/0cc561632947220cbf5d8a204be8a4a9ad9abb5a/src/software-management/usecase-update-swlist.md

**Option #1**
- **Description:** Delta-comparision by SM Agent. <br/>
See ref: https://app.archbee.io/docs/9iGX1hbDjwAeMfyO9A3YE/coxr9CuTWSjk0eE1Nzgoj <br/>
-> Section: "Update Software Operation" <br/>
-> Steps underneath "The operation is executed as follows" <br/>
and <br/>
-> Section: "Hiding Software Module" <br/>
-> Snippet: "Certain packages should not be uninstallable. By not listed them, they are also not uninstalled based on delta algorithm described under "update software" above (only software modules that are part of the current software list can be uninstalled.)"
- **Implications:** TBD <br/>

**Option #2**
- **Description:** Delta-comparision by Package Manager Plugins<br/>
TODO: more Details will be added here.<br/>
- **Implications:** TBD <br/>

**Comment:** -


&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;



**Decision-003**
----------------

**Problem Statement:** In which order shall items of sw-list be processed? Shall order from Cloud be kept, or shall thin-edge re-order the list for some reason?

**Reference:** Sequence Diagram "Update SW-list" (see reference "TO-BE-DECIDED-#2"): https://github.com/thin-edge/thin-edge.io-specs/blob/0cc561632947220cbf5d8a204be8a4a9ad9abb5a/src/software-management/usecase-update-swlist.md

**Option #1**
- **Description:**  Keeping order inside sw-list from Cloud to enable Cloud operator to dictate the order of processing.<br/>
- **Implications:**  Would allow the cloud operator:<br/>
 - to manage package dependencies<br/>
     &nbsp;&nbsp;&nbsp;a) within on package type (E.g.: 1st install debain package "foo", 2nd install debian package "bar")<br/>
     &nbsp;&nbsp;&nbsp;b) to manage package dependencies beyond package types (E.g.: 1st install debian package "foo", 2nd install Docker package "bar")<br/>
     &nbsp;&nbsp;&nbsp;c) to install package type per package type (E.g.: 1st install all Debain packages, 2nd install Docker packages)<br/>
NOTE: That or a similar solution would be required if Package Management auto-dependency solving is disabled or not available (see Decision 4). <br/>
Pro: Gives Freedom to the Cloud Operator<br/>
Con: Gives responsibility to the Cloud Operator <br/>
Disadvantage: a) Not a declarative semantics anymore b) Cloud must have detail knowledge about package types on different devices, impossible to scale?<br/>

**Option #2**
- **Description:**  Hardcoding some as best expected order into the agent. <br/>
Example order:  - 1st process per package type ("debian", ...) - then 2nd per operation ("remove, install"). <br/>
- **Implications:** No chance to solve dependencies on Cloud-side. See also Decision 4. <br/>

**Option #3**
- **Description:** Define plugin execution order inside agent, e.g. using numbers in the plugin filename? Example: <br/>- 01debain<br/>- 02docker<br/>- 03apama <br/>
- **Implications:** TBD <br/>

**Comment:** 

D&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;


**Decision-004**
----------------

**Problem Statement:** Shall auto dependency solving used if package manager supports it?

**Reference:** None

**Option #1**
- **Description:** Yes, let Package Manager solve dependencies automatically. <br/>
See also ref https://app.archbee.io/docs/9iGX1hbDjwAeMfyO9A3YE/coxr9CuTWSjk0eE1Nzgoj <br/>
-> Section: "Update Software Operation" <br/>
-> Snippet: "Just top-level packages are listed, because the software type specific Installer is responsible for dealing with dependency management." <br/>
- **Implications:** Convenient for Cloud-Operator, but could lead to unexpected / problematic behaviour as below: <br/>
-> Could lead exceeded current sw-list size reported to cloud. Since Cloud operator is unknown of auto-dep installed package he could not fix that by added them to black-list / filter for current sw-list. <br/>
-> Could lead to unexpected flash capacity load, in case of large dependencies. <br/>
-> Could lead to invisible package on Cloud side (e.g. when auto-deps are blacklisted). These invisible package might be an issue from security point of view. They could be affected by vulnerabilities but Cloud Operator is not aware of these Packages (invisible). Even he would be aware of them he would not be able to update or remove them (invisible). <br/>
See also ref https://app.archbee.io/docs/9iGX1hbDjwAeMfyO9A3YE/coxr9CuTWSjk0eE1Nzgoj <br/>
-> Section: "Why is Software Management important" <br/>
-> Snippet: "As all other software too, also the software on the device needs to be updated to fix e.g. security bugs. This is even required by e.g. EU Standards."


**Option #2**
- **Description:**  No, prohibit that Package Manager solve dependencies automatically. <br/>
Dependency Management need to be done on Cloud-side, e.g. by adding all depending package into the update sw-list request, in the correct order. <br/>
See also Decision 3.
- **Implications:** More effort and responsibility for Cloud Operator. <br/>

**Option #3**
- **Description:**  Assume the plugin will install the require dependencies but let the user a way to list them if not. This implies that the all the packages are installed in the order specified by the the cloud operator. However, the operator will have to provide the correct order with the full set of dependencies only for the plugin that requires it. <br/>
See also Decision 3.
- **Implications:** More effort and responsibility for Cloud Operator. <br/>

**Comment:** Could be finally decided by Customer by adapting according Package Manager Plugin. But if provide no way to solve dependencies manually (see Decision 3) thats not an option for the Customer. 




