# Decisions deferred for later

## How to handle "messages from the future"?

"Messages from the future" are the messages having an event timestamp higher than their processing time.
As messages from the future are less likely to occur as long as the messages are generated from the thin-edge device itself,
we are deferring the handling of such messages to a future release when we add support for measurements comong from external time sources where, this issue could be far more serious.
Until then such messages from the future will be dropped.

## How to handle conflicting messages?

While trying to add a measurement to a batch, and if another conflicting measurement of the same type is already present in that batch,
the currect specification proposes to split the current batch into two so that both the conflicting messages are in separate batches created with that split.
This conflict resolution mechanism can be revisited later on.

Some alternate options are:
* Replace the older measurement with the newer measurement with higher event timestamp
* Create a dedicated singleton batch for the conflicting measurement and send it separately without affecting the current active batch

Replcaing older measurements with newer ones can be done as the simpler interim solution, instead of batch splitting.

## How to handle duplicate messages(same messages with same type, value and timestamp delivered multiple times)?

The current specification proposes to reject duplicate messages which can be identified based on their timestamp and their message type.
Another option is to have the batcher subscribe to the MQTT broker with QoS 2, and ensure that the measurement publishers publish with QoS 2 as well, so that it doesn't receive duplicate messages at all.
Other smarter alternatives can be explored in the future.
