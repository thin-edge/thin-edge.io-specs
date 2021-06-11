# Concepts

* __Software Module__: A specific piece of software.
  Examples: the Mosquitto V2.02 debian package, or an ML model in the form of one or more ONNX or PMML file(s). 
* __Software Type__: A way to package, distribute, install and update software modules.
  Examples: Debian package, Docker image, ML Model.
* __Package manager__: A tool to install software modules of given type.
* __Software List__: A set of reference to software modules.
* __Reported Software List__:
  The software list that is stored in the device digital twin inside the IoT platform.
  It represents the current status of the device as perceived by the IoT platform and the device operator.
  Assuming the device is connected to the IoT platform ,
  the reported software list should be eventually the same as the current software list. 
* __Desired Software List__: The software list that specifies which software the device should run in the future.
  This list can be either complete or partial.
  A partial software list is a list of software modules to be to be added or removed from the device.
  A complete list contains all the software modules.
* __Current software list__: The software list that is currently installed on the device, as reported by the agent. 
