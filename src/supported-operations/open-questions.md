# Open questions

## Operation configuration file
* What kind of pattern to match the listen topics?
  * The preference is for now along [`glob`](https://docs.rs/glob/latest/glob/) patterns.
* Configuration file permissions, what to set, who should be able to change it?
  
## Operation execution
* Which component is responsible for responding to the cloud with the status (EXECUTING, SUCCESS or FAILURE) of the operation?
  * The c8y mapper or the operation plugin?
  * The proposal is to let this responsibility to the operation plugin - because the protocol can be operation-specific
    as [this is actually the case for Cumulocity on success](https://cumulocity.com/guides/device-sdk/mqtt/#updating-operations).
    That said one can implement a c8y operation wrapper that runs an inner operation (`c8y-operatiom my-operation`)
    and handles for the latter the protocol with Cumulocity.
* Which component is responsible for executing the operations triggered by the mapper on behalf of the cloud user?
  * The c8y mapper or the agent?
  * The proposal is:
     * The operations are executed by the agent.
     * The mapper is stateless: translating messages back and forth between the cloud end-point and the agent.
* How to pass parameters to the operation?
  * Do we need an `[extra]` set of parameters in the operation Toml file?
* How an operation plugin can be initialized?
  * Is this the responsibility of the plugin installer? the mapper?

## Extensions
* Reload the set of operations if an operation is added, removed or changed?
* Do we need to provide a cli support to install operations?
  * Something along the lines of `tedge operations` as proposed [here](./command-line-support.md).
  * We can, but in any case this will be done only once the operation support well in place.
* Do we need to be able to trigger operations through MQTT rather than be running a process?
  * Do we need an `[mqtt]` set of parameters in the operation Toml file?
