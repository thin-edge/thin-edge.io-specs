# Extensible support of operations

The main features of thin-edge can be extended with plugins that provide specific support for new operations,
that can be then triggered from the cloud or from other components.

An operation can be as simple as executing ad-hoc commands or as complex as installing new software versions
on the thin-edge device. Other examples are the abilities to upload log files
or to open an ssh-tunnel from the cloud to the device.

On a device, an operation is materialized by an executable that interacts with the cloud end-point via thin-edge.
* Some operations are provided by thin-edge (for instance *Software Management*).
* New operations can be added by tier parties using any programming language.
* In order to be open and flexible, thin-edge sets no constraint on the protocol
  used by a plugin to interact with the cloud and the device local services.
* For each supported cloud, Thin-edge provides the mechanisms:
  * to register operation plugins on the device,
  * to notify the connected cloud instance with the set of operations supported by the device,
  * to notify the appropriate operation plugin when an operation request is triggered from the cloud.
* The implementation of an operation might be cloud specific or not.
  * If not, the implementation has to provide protocol-translation mechanisms around the main operation mechanism.

TOC:
* [Requirements](./requirements.md)
* [Principles](./principles.md)
* [Adding new operations](./thin-edge-supported-operations.md)
