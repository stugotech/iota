# IoTa Protocol 1.0

## Licence

Copyright (c) 2016, Stugo Ltd

This work is licensed under the Creative Commons Attribution-ShareAlike 4.0 International License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/4.0/ or send a letter to Creative Commons, PO Box 1866, Mountain View, CA 94042, USA.


## Changes

| Version | Date | Author | Description |
|----|----|----|----|
| 1.0 | 6th June 2016 | Gordon Mackenzie-Leigh | Initial version |



## Introduction

IoTa is a session-layer protocol designed to enable reliable communications for the Internet of Things.  This specification does not define the message transport but the use envisioned is the transport of small amounts of data over SMS using embedded devices not endowed with a full TCP/IP stack.


## Protocol

### Overview

The IoTa protocol introduces the concept of *sessions*, or *conversations*.  A session allows both parties involved in a communication to keep track of messages that have been sent or received.  This gives the following guarantees:

1. Messages are processed **in-order**
2. All messages are **acknowledged**

These two properties allow both parties to reach a **consistent state**.


### Parameters

This protocol depends on the following parameters which should be defined by the implementor.

#### `INITIAL_ROUND_TRIP_TIME`

The initial guess of message round-trip latency used when sending retry messages.  A sensible value for this parameter depends on the underlying physical/transmission protocols so no suggested value is given.


### Message envelope

An IoTa message is comprised of an IoTa *header* (10 bytes) followed by zero or more *payload bytes* as follows:

| Offset | Field | Size |
|----|----|----|
| 0 bytes | `SESSION` | 4 bytes |
| 4 bytes | `TIMESTAMP` | 4 bytes |
| 8 bytes | `ACK_SEQUENCE` | 1 byte |
| 9 bytes | `SEND_SEQUENCE` | 1 byte |
| 10 bytes | `PAYLOAD ...` | *n* bytes |


#### SESSION field

The `SESSION` field is a non-zero random number chosen at the start of the conversation.  It ensures that only messages relevant to the current conversation are processed by the receiver.

#### TIMESTAMP

The `TIMESTAMP` field is an increasing integer related to the current date and time in the UTC timezone; e.g. a Unix timestamp.

#### SEND_SEQUENCE field

The `SEND_SEQUENCE` field is a monotonically increasing integer which starts at zero for the first message of a conversation and is incremented each time the peer sends a new message.

#### ACK_SEQUENCE field

The `ACK_SEQUENCE` field is the `SEND_SEQUENCE` value of the last message accepted by the peer.


#### PAYLOAD field

The `PAYLOAD` is an optional field containing application-defined data.


### Message size and integrity

This specification assumes that the transport-layer protocol (or other lower level protocols) define the concept of a datagram and provide guarantees about its integrity.  The maximum message size may be limited in these layers.


### Sessions

#### Initiation

A new session is established by sending a message with a randomly-chosen `SESSION` field and a `SEND_SEQUENCE` field of zero.  This session will continue until one of the parties starts a new session.  The `SEND_SEQUENCE` is incremented by one for each message sent.  When it reaches 255 (maximum value of an unsigned byte) it wraps round to 1.  It must not wrap to 0 as this would start a new session.

#### Transmission and retransmission

Each peer will attempt to send only one message at a time and will block until it receives acknowledgement.

The time in between sending a message and receiving a reply is stored in the peer state as `ROUND_TRIP_TIME` (starting at `INITIAL_ROUND_TRIP_TIME` seconds if no messages have been sent).  The sender will resend a message for which it has not received an acknowledgement after a period of `2×ROUND_TRIP_TIME` has elapsed.

In order to deal with network congestion, transmission issues, or receiver processing delays, `ROUND_TRIP_TIME` is doubled each time a message is retransmitted.


#### Acknowledgement

Acknowledgement occurs when the peer receives a message with an `ACK_SEQUENCE` field equal to the `SEND_SEQUENCE` of the message sent.


#### Invalid messages

An invalid message is any message with an unexpected `SEND_SEQUENCE`, unexpected `ACK_SEQUENCE` or `TIMESTAMP` less than the value stored in the peer state.  

Invalid messages are discarded with no response or further processing.


### Peer state

Each client stores the current `SESSION` id, the current `SEND_SEQUENCE` number, the last `TIMESTAMP` received from the other peer, the last `SEND_SEQUENCE` received from the other peer (i.e. the current `ACK_SEQUENCE`) and a value for `ROUND_TRIP_TIME` which is the time in between sending the last message and receiving an acknowledgement.  It also stores the current message if it has not received an acknowledgement.


### Message types

#### START

A `START` message is any message with a different `SESSION` value from the receiver's state and `SEND_SEQUENCE` and `ACK_SEQUENCE` set to zero.  The new `SESSION` value is a randomly-generated non-zero number.  Upon receiving a valid `START` message, the receiver will update its peer state so that `SESSION` matches the value in the message and the `SEND_SEQUENCE` and `ACK_SEQUENCE` are reset.  A START message may not contain a payload.


#### ACK+NOP

An `ACK` (acknowledge) message is any message with a valid `SESSION` value and `ACK_SEQUENCE` value corresponding to `SEND_SEQUENCE` of the message to acknowledge.  If no data is included then it is a `NOP` (no operation) message.

#### ACK+DATA

A `DATA` message is any valid message with a payload that contains one or more bytes of data.  The sender always sets the `ACK_SEQUENCE` field to the most recent `SEND_SEQUENCE` received, so every message sent is also an acknowledgement of the last message received.

#### RESET

A `RESET` message is any valid message with a zero `SESSION` value and zero `SEND_SEQUENCE` value.  No further messages will be accepted or transmitted by either peer until a `START` message is sent and acknowledged.



## Implementation


### Psuedo-code

The following methods are required for a successful implementation.  They are defined in psuedo-code with javascript syntax.


```javascript
function open(data) {
  state.SESSION = generate_nonzero_random();
  state.ACK_SEQUENCE = 0;
  state.SEND_SEQUENCE = 0;

  if (data) {
    state.PAYLOAD = data;
  }

  state.SEND_TIMESTAMP = get_timestamp();
  state.STATE = OPEN_WAITING;
  hardware_send();
}


function read() {
  var message = hardware_read();

  if (message && message.timestamp >= state.RECEIVED_TIMESTAMP) {
    // the received message is valid because it has a timestamp greater than last received.

    if (message.SESSION == 0) {
      // received RESET message - close the connection
      state.STATE = CLOSED;
      return CONNECTION_RESET;

    } else if (message.SEND_SEQUENCE == 0) {
      // received START message - start a new connection
      state.SESSION = message.SESSION;
      state.ACK_SEQUENCE = 0;
      state.SEND_SEQUENCE = 0;
      state.RECEIVED_TIMESTAMP = message.timestamp;

      return NEW_CONNECTION;

    } else if (message.SESSION == state.SESSION) {
      // the message is valid because it is for the current session

      if (message.ACK_SEQUENCE == state.SEND_SEQUENCE && state.STATE == OPEN_WAITING) {
        // receive ACK for sent message
        state.STATE = OPEN_READY;
      }

      if (message.SEND_SEQUENCE == state.ACK_SEQUENCE+1) {
        // new data received
        
      }
    }
  }

  // no message read or message was invalid
  return NO_MESSAGE;
}
```
