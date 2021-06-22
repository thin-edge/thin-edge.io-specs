# Message Batching

A deterministic message bacthing algorithm that builds batches based on the timestamp at which the message is generated rather than the timestamp at which the message was received by the batcher.

## Terminology

* **Event time:** The time at which a measurement/event is *generated*
* **Processing time:** The time at which that measurement/event is *received/processed*
* **Batching window:** A `batching window` is defined as the maximum allowed gap between the event timestamps of the first message in a batch and the last message in a batch. It is important to note that the `batching window` is based on the *event timestamps*.
* **Max message delay:** The `maximum message delay` is defined as the maximum acceptable delay between the even generation time and the event processing time for a message received by the batch processor. If an event generated at time `t` is received by the batch processor *after* the maximum acceptable delay, it will considered too old and won't be processed further.
* **Batching timeout:** The `batching timeout` is the maximum time that the batcher will wait for the last message after a batch has started. The `batching timeout` can be defined as the `first message processing time` + `batching window` + `max message delay`

## Requirements

* Different measurements generated within a `batching window` must be batched together.
* Batching should not be just a static periodic batching at regular intervals but instead must be dynamic in such a way that the batching is started on the receipt of the first message in a logical batch and (ideally) ends on the receipt of the last message from that logical batch.
* The `batching window` helps in limiting the number of messages that get batched up together.
* The `batching timeout` prevents the batcher from waiting forever to receive the last message in a batch after the batch has started on the receipt of the first message. 
* While a batch is being built, if a measurement that's already present in the current batch arrives with a different timestamp(meaning it's a different measurement of the same type), a new batch must be started and this new measurement must be added to that batch.

## Assumptions

* Messages are delivered to the batcher either in real-time or in near-real-time. That is, a message received by the batcher at processing time `T` can only have an event timestamp `t` <= `T`. This is because there can always be some delay between the time at which the messages is generated(event time) and the time at which that message is delivered to the batcher(processing time) via the MQTT broker.
* Messages are generated and processed on the same `thin-edge.io` device using the same system clock. This provides us the extra guarantee that the message generation time can never be higher than the message processing time.

### MQTT Ordering Guarantees

The following statements on MQTT ordering guarantees are based on the MQTT 3.1 specification [here](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718105)

* Message ordering is guaranteed by MQTT at a topic level, meaning published messages will be delivered to subscribers in the same order as they were published.
* QOS 0 can lead to message drops but they will still be delivered in order. That is, if the messages `m1, m2, m3, m4, m5` are published in that order with QOS 0, a subscriber may not get all those messages and might only receive `m1, m3, m5` with m2 and m4 dropped on the way. But the overall order of the delivered messages is still guaranteed. And there won't be any duplicate messages with QOS 0.
* QOS 1 will prevent message drops but can lead to message duplications. That is, if the messages `m1, m2, m3, m4, m5` are published in that order with QOS 1, a subscriber is guaranteed to get all those messages but with some duplication like `m1, m2, m2, m3, m2, m3, m4, m5`. Even though the ordering is guaranteed even for the duplicate messages, there's a partial reordering happening for the overall flow.
* QOS 2 will prevent message drops as well as duplication and guarantees ordering as well. That is, if the messages m1, m2, m3, m4, m5 are published in that order with QOS 1, a subscriber is guaranteed to get all those messages in the exact same order with no message loss.
* Even if QOSes are mixed up, the ordering is guaranteed, but with duplication causing partially out of order messages because of QOS 1.

## Usecases

In the examples defined in the following use-case sections, the `batching window` is fixed as **50** and the `max message delay` is fixed as **20**. That is, if a message `m1` generated at `t100` is received by the batcher at `T110`, then the batcher will wait till `T170` to group any messages that has an embedded timestamp between `t100` and `t150`. Messages with event timestamp less than `t100` and greater than `t150` will be out of this batch even if received before the batching timeout at `T170`.

The representation `a5:t50@T60` represents the 5th instance of a measurement of type `a` generated with the event timestamp of `t50` received by the batcher at processing time `T60`.

The batch color schemes used in the images in the use-case sections are defined as follows:
* Yellow: Open batch
* Green: Closed batch
* Red: Rejected message(s)
* Blue: Indecisive

### UC1: Simple batching with batching window

Message flow:

![Batching window](./message-batching/UC1.svg)

Rationale:

* `a1:t110@T120` starts a new batch with the batching window `t110-t160` with the batching timeout at `T180`
* `b1` and `c1` gets added to the same batch as they are within the bounds of batch1's batching window.
* `d1:t152@T160` starts a new batch as its event timestamp is outside the first batch's batching window of `t110-160` even though it was received before the batch timeout of `T180`. The first batch is not closed at this point as messages from that batch can arrive till the batching timeout at `T180` and it will be kept open until then.
* `e1:t170@T190` gets added to the second batch itself as it belongs to the batching window of `t152-t202` started by `d2`.


### UC2: Simple batching with batching timeout

Message flow:

![Batching timeout](./message-batching/UC2.svg)

Rationale:

* `a1` `b1` and `c1` formed the first batch which got closed with the batching timeout at `T180`
* `d1` and `e1` are arrived after the batching timeout and hence a separate batch was started at `T210`. But since they're both within the same batching window, they're added together into a single batch.

### UC3: Multiple batches due to conflicting measurements

Message flow:

![Conflicting measurements](./message-batching/UC3.svg)

Rationale:

* `a1:t110@T120` starts a new batch with the batching window `t110-t160` with the batching timeout at `T180`
* Receipt of `a2:t140@T150` starts a new batch even though its event timestamp is within batch1's batching window as well as it was received before the batching timeout at `T180` because the measurement `a1` of type `a` already exists in the first batch. This second batch is started with a batching window of `t140-t190` with the batching timeout at `T210`. This also results in the first batch's batching window to be reduced from `t110-160` to `t110-t140` with the batching timeout also lowered from `T180 ` to `T160`.
* `a3:t155@T170` starts a third batch as it can't be added to the second batch either due to the conflict with `a2`. Its batching window would be `t155-t205` with the batch timeout at `T225`. It also results in the shorteing of batch2 with the batching window re-adjusted to `t140-155` with the batching timeout at `T175`.

### UC4: Receiving older already batched messages after starting a new batch

Message flow:

![Late duplicate messages](./message-batching/UC4.svg)

Rationale:

* `a1` `b1` and `c1` formed the first batch which got closed with the batching timeout at `T180`
* `d1:t190@T210` would trigger the creation of a new batch with the batching window `t190-t240` and a batching timeout at `T260`.
* When a duplicate of `c1:t130` is received at `T220` while building this second batch, it needs to be rejected because it is too old to be fit into the current batch's batching window.

### UC5: Receiving older unbatched messages after starting a new batch

Message flow:

![Late messages](./message-batching/UC5.svg)

Rationale:

* `a1` `b1` and `c1` formed the first batch which got closed with the batching timeout at `T180`
* `a2:t175@T190` starts a new batch with batching window `t175-t225` and a batching timeout at `T245`. `b2` is also added to the same batch.
* When `d1:t145` is received at `T210`, it can't be added to the first batch as it is already closed, even though its event timestamp `t145` falls within the first batch's batching window from `t110-t160`. It can't be added to the second batch either as the second batch's batching window is `t175 - t225`, unless its batching window is adjusted to `t145 - t195`. But then, `b2:t200` which is already in this second batch will have to be moved to a third batch as it doen't fall within the updated batching window of batch2 anymore. The other option is to start a new batch with `d1` for any message with event timestamp in the range `t145 - t175`.

## Design Specification

TBD