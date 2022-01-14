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
* The agent is the process that runs and monitors the operation commands.
* The cloud mappers trigger the operations:
  - listening for remote requests,
  - translating the cloud specific requests into local tedge requests,
  - reporting back the operation progress and outcome.
* An operation or any local process can trigger an operation
  by sending the appropriate message to the agent.
* The agent maintains a journal where the progress of the operations is logged 
* This journal is actionable, i.e. contains the information
  to execute a scheduled operation or to stop a running process.
* On restart, the agent checks the journal
  for operations to start, cancel or restart.
* The agent uses an MQTT topic to report operation progress
  with (start, done, cancel, abort) events.
* A mapper can listen operation progress
  to report back this progress to the cloud end-point.