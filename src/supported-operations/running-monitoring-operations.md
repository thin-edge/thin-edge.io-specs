# Running and monitoring operations
  
## Requirements
* The progress of each operation request must be logged in a journal,
  with timestamped entries of when the operation has been 
  scheduled, launched, cancelled, completed or aborted.
* Each command execution must be recorded in a log
  accessible to the device owner and the remote operator.
  This log contains the full command path and arguments,
  plus the recorded standard output and error of the process.
* The device owner should be able to define constraints
  when a command can be launched / stopped:
  - Can two instances of the same operation run concurrently?
  - Can the command run concurrently with another?
  - Has the process to be detached from the agent daemon?
  - Has an operation to be aborted on agent restart?

## Proposed design
* The operations are defined by the configuration files in [`/etc/tedge/operations`](./principles.md).
* The cloud mappers trigger the operations.
  * This is done on reception of operation requests from the cloud:
    each operation configuration defines the topic to listen, and the message-pattern to match.
  * The responsibility of the mapper is only to translate cloud specific messages into thin-edge requests.
    When an operation request is received, the mapper combines the request and the operation configuration
    into execution information, and delegates the execution to the agent.
* The agent is the process that runs and monitors the operation commands.
  * The agent listen for operation request on the `tedge/operations/start` topic.
  * Each request provides the information to execute and monitor the operation:
     * a request unique identifier,
     * the operation name,
     * the command path,
     * an array of command arguments (at least the message that triggered the request),  
     * the user running the command.
  * ```json
    { "id": "abcdef",
      "operation": "c8y_LogfileRequest",
      "command": "/usr/bin/c8y_logfile",
      "args": ["522,DeviceSerial,logfileA,2013-06-22T17:03:14.000+02:00,2013-06-22T18:03:14.000+02:00,ERROR,1000"],
      "user": "tedge"
    }
    ```
* The agent schedules the operations reporting the progress on the `tedge/operations/status` topic.
  * ```json
    { "id": "abcdef",
      "operation": "c8y_LogfileRequest",
      "status": "scheduled"
    }
    ```
  * `"status": "schedule"`: on reception of a well-formatted operation request.
  * `"status": "executing"`: just before executing the operation.
  * `"status": "successful"`: when the operation returns with a zero exit status.
  * `"status": "failed"`: when the operation returns with a non-zero status.
  * NOTE: One might also consider `cancelled` and `aborted` operations to deal with user cancellation
    and operations killed by a signal or a power failure.
* The mapper might subscribe to the `tedge/operations/status` topic
  and translate the operation status into cloud specific messages.
  * NOTE: There is open question here.
    * Does the c8y mapper translate the `tedge/operations/status` messages into equivalent smartRest2 messages?
    * Or does the c8y operation publish their progress directly to `c8y/s/us`?
    * The former would be simpler for the operator implementor. 
      However, most operations will in any case have to dialog over `c8y/s/us` and `c8y/s/ds`
      to implement the specificities of the operation protocol.
* The execution of a command is logged in file name after the operation name and the request id.
  * `/var/log/tedge/agent/{operation-name}-{operation-request-id}-{starting-datetime}.log`
  * These logs are build as for the sm plugin processes
    (see [logged_command.rs](https://github.com/thin-edge/thin-edge.io/blob/main/crates/core/plugin_sm/src/logged_command.rs))
    with 4 points.
    * The command path and the parameters.
    * The exit status.  
    * The stdout of the process.
    * The stderr of the process.
* The `tedge/operations/status` topic and `/var/log/tedge/agent` logs are used to monitor the operations on a device.
  The agent also maintains a journal used to recover the previous state on restart.
  * This journal is actionable, i.e. contains the information
    to execute a scheduled operation or to stop a running process.
  * Each time an operation is scheduled, launched, completed,
    the agent logs this state in its journal with all the information to resume that state and proceed to the next step.
  * On restart, the agent checks the journal for operations to start or cancel.
    * An operation that was scheduled is rescheduled.
    * An operation that was running is marked as aborted.
    * NOTE: can we check that a detached process is still running (comparing pid or program name)?
  * NOTE: A key open question is: what mechanism to implement this journal?
    * The requirements are:
      * Atomic append of an entry.
      * Crash-safe.  
      * Sequential read on restart.
      * Old entries removal.
    * The alternatives are:
      * An append-only text file. Can be acceptable for a first drop, but will stay fragile.
      * Sqlite fills the bill and much more but also introduces a new external dependency.
      * An embedded database as [`sled`](https://github.com/spacejam/sled) is closer to our requirements,
        but might be a bit heavy for an embedded device. 
      * A commit log as [`commitlog`](https://lib.rs/crates/commitlog) can be the perfect fit.
        It raises questions about adoption/support though.