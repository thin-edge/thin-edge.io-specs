# Message Batching

A deterministic message bacthing algorithm that builds batches based on the timestamp at which the message is generated rather than the timestamp at which the message was received by the batcher.

## Terminology

* **Event time:** The time at which a measurement/event is *generated*
* **Processing time:** The time at which that measurement/event is *received/processed*
* **Batching window:** A `batching window` is defined as the maximum allowed gap between the event timestamps of the first message in a batch and the last message in a batch. It is important to note that the `batching window` is based on the *event timestamps*.
* **Batching timeout:** The `batching timeout` is the maximum time that the batcher will wait for the last message after the batch has started. It is important to note that the `batching timeout` is based on the *processing time*.

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

In the following examples, the representation `m5:t50@T60` represents a measurement from its 5th batch generated with the event timestamp of 50 received by the batcher at processing time 60.

The `batching window` is fixed as **50** and the `batching timeout` is fixed as **100** for these examples. That is, if a message `m1` generated at `t105` is received at `T110`, then the batcher will wait till `T210` to group any messages that has an embedded timestamp between `t105` and `t155`. Messages with event timestamp less than `t105` and greater than `t150` will be out of this batch even if received before the batching timeout at `T210`.

The message sequences in the following examples are ordered as per the message `processing time` and NOT event time.

### UC1: Simple batching with batching window

Message sequence:
```
a1:t100@T110, b1:t115@T120, c1:t145@T150, d1:t155@T160, e1:t165@T170
```

Desired batches: 2

```
[a1:t100@T110, b1:t115@T120, c1:t145@T150],
[d1:t155@T160, e1:t165@T170]
```

Rationale:

Even though `d1` was received before the batching timeout at T200, it's in the second batch because its embedded timestamp of `t160` is outside the batching window of t150 (t100 of a1 + batching window 50). `e1` also needs to be in the same batch as `d1` as it's within the batching window of `d1`.

### UC2: Simple batching with batching timeout

Message sequence:
```
a1:t100@T110, b1:t115@T120, c1:t145@T150, d1:t155@T210, e1:t165@T225
```

Desired batches: 2

```
[a1:t100@T110, b1:t115@T120, c1:t145@T150],
[d1:t155@T210, e1:t165@T2250]
```

Rationale:

`d1` and `e1` are arrived after the batching timeout and hence a separate batch was started at `T210`. But since they're both within the same batching window, they're added togethee into a single batch.

### UC3: Multiple batches due to conflicting measurements

Message sequence:
```
a1:t100@T110, b1:t115@T120, c1:t125@T150, a2:t135@T160, a3:t145@T170
```

Desired batches:
```
[a1:t100@T110, b1:t115@T120, c1:t145@T150], 
[a2:t155@T160], 
[a3:t165@T170]
```

Rationale:
Even though `a2` and `a3` were received before the batching timeout at `T200` and within the batching window of `t150`, they're not included in the first batch itself because the measurement `a1` of type `a` already exists in the first batch. But they wouldn't even be added to another single batch but have to be put into two different batches they are of the same type `a` between themseleves as well.

### UC4: Receiving older already batched messages after starting a new batch

Message sequence:

```
a1:t100@T110, b1:t115@T120, c1:t145@T150, d1:t155@T210, c1:t145@T150, e1:t165@T225
```

Desired batches: 2

```
[a1:t100@T110, b1:t115@T120, c1:t145@T150],
[d1:t155@T210, e1:t165@T2250]
```

Rationale:

Here `d1` would trigger the creation of a new batch which can include any message that has an event timestamp greater than `t155` and less than `t205` due to the batching window of 50. Now, when `c1:t145` is received while building this second batch, it needs to be rejected because `c1` is already included in the first batch.

### UC5: Receiving older unbatched messages after starting a new batch

Message sequence:

```
a1:t100@T110, b1:t115@T120, c1:t145@T150, a2:t155@T210, b2:t200@T220, d1:t149@T225
```

Desired batches: 3

```
[a1:t100@T110, b1:t115@T120, c1:t145@T150],
[a2:t155@T225, b2:t200@T220]
[d1:t149@T225]
```

Rationale:

At `T200`, the first batch would be closed and `a2` received at `T210` would trigger the creation of a new batch that can include any message that has an event timestamp greater than `t155` and less than `t205` due to the batching window of 50. Now, when `d1:t149` is received while building this new batch, it can't be added to the first batch as it is already closed, even though its event timestamp `t149` falls within the first batch's batching window from `t100 - t150`. It can't be added to the second batch either as the second batch's batching window is `t155 - t205`, unless its batching window is adjusted to `t149 - t199`. But then, `b2:t200` which is already in this second batch will have to be moved to a third batch as it doen't fall within the updated batching window of batch2 anymore. The other option is to start a new batch with `d1` for any message with event timestamp in the range `t149 - t154`. 

## Design Specification

TBD