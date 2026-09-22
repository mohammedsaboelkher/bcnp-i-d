---
title: "Brain Control Network Protocol (BCNP)"
docname: draft-salaheldin-bcnp-latest
category: std
ipr: trust200902
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: art
workgroup: "Individual submission"
keyword:
 - brain
 - control
 - heterogeneous devices network
 - dynamic configuration
 - application layer
 - binary protocol
venue:
  github: "mohammedsaboelkher/bcnp-i-d"
  latest: "https://mohammedsaboelkher.github.io/bcnp-i-d/draft-salaheldin-bcnp.html"

author:
 -
    fullname: Mohamed Salaheldin Abouelkhir Attia
    email: mohammeds.aboelkher@gmail.com

normative:
  RFC2119:
  RFC8174:
  RFC8259:

informative:
  RFC9293:
...

--- abstract

The Brain Control Network Protocol (BCNP) is a binary, TCP-based application-layer protocol for discovering, configuring, and commanding networked devices from a single controller using a small, fixed vocabulary of discrete control inputs, such as those produced by a brain-computer interface. A Controller establishes a private wireless network that Devices join, discovers Devices and assigns each a persistent address, discovers and registers the Actions a Device can perform, and subsequently issues short, stateless commands, expressed through that same small input vocabulary, to elicit those Actions. This document specifies BCNP's binary message formats, its Discovery, Pairing, and Command phases, and the reconciliation mechanism used to recover from lost acknowledgments and Devices that change network address.


--- middle

# Introduction {#intro}

Brain-computer interfaces (BCIs) allow a user to control external systems using neural signals, most commonly electroencephalography (EEG), without requiring physical movement. This makes BCIs a valuable input modality for assistive technology, benefiting users with severe motor impairments, as well as an emerging general-purpose input method. A defining constraint of BCI-based input, however, is that only a small number of distinct mental commands can typically be classified reliably; each additional command that a user must learn to produce, and a classifier must learn to distinguish, increases training burden and reduces accuracy.

The Brain Control Network Protocol (BCNP) is designed around this constraint. Its Controller (a software program with no dedicated user interface assumed, and expected in practice to be driven by an EEG headset's signal classifier) interacts with the protocol through a small, fixed set of "probes": seven control probes (Discover, Hello, Register, Forget, End, Collection, Device) and a configurable number, K, of key probes, where K corresponds to however many distinct classes a given classifier can reliably distinguish. Each probe corresponds to one distinguishable mental command a user can produce.

BCNP's central design goal is to let this same small vocabulary of K probes control an arbitrary number of Devices and Actions, without requiring dedicated software or a retrained classifier for each new device. This is achieved by making the mapping between probes and real Devices and Actions fully configurable at runtime: a Controller assigns Devices to Collections and Device Keys during a Device Configuration Phase ("Discovery"), and assigns a Device's Actions to Action Keys during an Action Configuration Phase ("Pairing"), after which the same K key probes can elicit different Actions depending on which Device is currently addressed. A user who has learned to reliably produce K distinguishable mental commands can, in principle, control any number of BCNP Devices.

BCNP is designed for a controller-hosted network: the Controller is responsible for establishing the wireless network Devices join, and BCNP's security properties ([Security Considerations](#security)) depend on that network being the trust boundary. Joining a pre-existing, general-purpose network is out of scope for this document.

This document specifies:

* the BCNP binary header and message formats ([Message Format](#msgformat)),
* the Device Configuration Phase ("Discovery"), Action Configuration Phase ("Pairing"), and Command phase, including the state machines governing Controller and Device behavior ([Protocol Operation](#protoop)),
* a reconciliation mechanism for recovering from lost acknowledgments and Devices that have changed network address ([Protocol Operation](#protoop)), and
* the security properties and residual risks of BCNP's intended deployment model ([Security Considerations](#security)).


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology {#terminology}

Control Network:
: A network comprising one Controller and a set of Devices, in which the Controller configures and commands the Devices.

Controller:
: A software program that configures and commands a set of Devices via the Probes defined in this document. This document assumes no specific Controller user interface or input mechanism; see [Introduction](#intro) for the brain-computer interface use case that motivates BCNP's design.

Device:
: A software program that listens for configuration and commands from a Controller.

Probe:
: A single, discrete unit of Controller input, corresponding to one command a Controller's operator can produce. A Controller has seven Control Probes (Discover, Hello, Register, Forget, End, Collection, Device) and K Key Probes, numbered 0 to K-1.

K:
: The number of Key Probes available to a Controller. K is fixed for a given Controller and is expected to correspond to however many distinct inputs its underlying input mechanism can reliably distinguish. K bounds the range of Collection Key, Device Key, Action Key, and Temporary Device Key; it does not bound Root Action Key or Sequence ID. Because these bounded fields are each carried in one byte on the wire (see Message Format), K MUST be an integer from 1 to 256 inclusive.

Collection:
: A group of Devices, identified by a unique Collection Key, known to a Controller.

Collection Key:
: An unsigned integer in the range 0 to K-1, unique among the Collections known to a Controller, used to identify a Collection.

Device Key:
: An unsigned integer in the range 0 to K-1, unique among the Devices within a single Collection, used to identify a Device. Two Devices in different Collections may share the same Device Key.

Action:
: A capability of a Device that a Controller can elicit, for example turning on an LED.

Action Key:
: An unsigned integer in the range 0 to K-1, unique among the Actions registered for a single Device, used to identify an Action once a Controller has Paired with that Device.

Root Action Key:
: An unsigned integer in the range 0 to 255, assigned by a Device to identify one of its own Actions. Unlike Action Key, Root Action Key is not bounded by K: a Device's Root Action Keys need not be contiguous, and a Device may have more Actions than a given Controller's K. See [Protocol Operation](#protoop) for how a Controller addresses a Device's Actions when they outnumber K.

Temporary Device Key:
: An unsigned integer in the range 0 to K-1, assigned by a Controller during the Device Configuration Phase to temporarily identify a Device that has not yet been assigned a Device Key.

Device Configuration Phase:
: Also referred to as "Discovery". The process by which a Controller discovers Devices and assigns each a Collection Key and a Device Key.

Action Configuration Phase:
: Also referred to as "Pairing". The process by which a Controller, having opened a persistent connection to a single Device, discovers that Device's Actions and assigns each an Action Key.

Commanding:
: The process by which a Controller elicits a Device to perform an Action, outside of the Device Configuration Phase or Action Configuration Phase.

Temporary Device Map:
: A construct held in a Controller's volatile memory, mapping each currently assigned Temporary Device Key to the Network ID of the Device holding it, for the duration of a single Device Configuration Phase.

Device Map:
: A construct held in a Controller's persistent storage, mapping each Device Key, within a Collection, to the last known Network ID of the Device holding it. A Device's Network ID is not assumed to remain stable; see [Protocol Operation](#protoop) for the reconciliation mechanism used to recover a Device's current Network ID.

Action Map:
: A construct, held independently in the persistent storage of both a Controller and a Device, mapping each Action Key to the corresponding Root Action Key.

Network ID:
: The address of a Device on the Control Network, for example an IP address.

Status Code:
: A single-byte value, carried in most response messages, indicating the outcome of the request being acknowledged. See [Message Format](#msgformat).

Reconciliation:
: The process by which a Controller recovers from an operation that failed or timed out, by querying whether a Device holding a specific Collection Key and Device Key pair can still be located. See [Protocol Operation](#protoop).


# Protocol Overview {#overview}

BCNP defines three sequential phases of interaction between a Controller and a Device: a Device Configuration Phase ("Discovery"), an Action Configuration Phase ("Pairing"), and Commanding.

During Discovery, a Controller broadcasts a request for any Device on its network to announce itself, assigns each responding Device a Temporary Device Key, and then assigns each a permanent Collection Key and Device Key, recording the mapping in its Device Map. A Device transitions from DEV_STATE_UNASSIGNED to DEV_STATE_ASSIGNED as a result.

During Pairing, a Controller opens a single persistent connection to one already-assigned Device, discovers the Actions that Device can perform, and assigns each an Action Key. The Device transitions to DEV_STATE_PAIRED for the duration of this connection, returning to DEV_STATE_ASSIGNED once it ends.

During Commanding, a Controller, addressing a Device by Collection Key, Device Key, and Action Key, elicits a single Action. Commanding requires no persistent connection, and no prior phase beyond Discovery, plus Pairing if the desired Action has not already been assigned an Action Key.

A Controller has a corresponding state machine: CTRL_STATE_IDLE, from which Discovery or Pairing may be entered and Commands may be issued; CTRL_STATE_DISCOVERING, active for the duration of a Device Configuration Phase; and CTRL_STATE_PAIRED, active for the duration of an Action Configuration Phase. [Protocol Operation](#protoop) specifies the complete state machines for both Controller and Device, and the procedures governing every transition.

## Deployment Model

BCNP assumes a Controller establishes its own wireless network, and that Devices join that network through some out-of-band mechanism before engaging in BCNP; the specifics of that mechanism are out of scope for this document. This document further assumes a single Controller per Control Network: a Device holds at most one Collection Key and Device Key at a time, and this document does not define behavior for a second Controller attempting to configure a Device already claimed by another. Both of these are scoping decisions rather than protocol requirements as such; see [Security Considerations](#security) for the trust model this deployment assumption enables.

Because Collection Key, Device Key, and Action Key are each bounded to the range 0 to K-1 (see "K" in [Terminology](#terminology)), a single Controller can, per Collection, address at most K Devices, each with at most K Actions available for direct addressing, for a system-wide ceiling of K x K addressable Devices. This is a deliberate consequence of BCNP's design (see [Introduction](#intro)), not an oversight.


# Message Format {#msgformat}

BCNP depends on a reliable, connection-oriented transport; this document assumes TCP {{RFC9293}}. All BCNP messages consist of a fixed 6-byte header, optionally followed by a payload whose shape is determined entirely by the Message Type in that header.

## Header

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Magic     |    Version    | Message Type  |  Sequence ID  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Payload Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

Magic:
: Fixed to 0xBC. A receiver MUST discard, without further processing, any message whose Magic byte is not 0xBC.

Version:
: The protocol version in use. A receiver MUST reject a message carrying a Version it does not support, responding where applicable with Status Code ERR_VERSION.

Message Type:
: Identifies the message; see "Message Types" below. A receiver MUST discard, without further processing, any message carrying a Message Type this document does not assign.

Sequence ID:
: Set by the sender of a request. A response MUST echo the same value as the request it answers. BCNP is synchronous: an implementation MUST NOT have more than one request outstanding at a time on a single connection. Under this constraint, wraparound of this one-byte field cannot cause a response to be mismatched to the wrong request.

Payload Length:
: The number of bytes following the header, encoded in network byte order (big-endian), 0 to 65535.

## Notation

The tables below describe each message's payload as an ordered sequence of fields. `[field_name:N]` denotes a field occupying exactly N bytes at that position; fields are concatenated with no delimiters, since both sides already know the expected shape from the Message Type. `+ JSON tail` denotes that all remaining bytes, up to Payload Length, are one UTF-8 encoded JSON value (see "JSON Schema" below). `count x [...]` denotes that the bracketed group repeats `count` times.

## Status Codes

Most response messages carry a one-byte Status Code as the first byte of their payload. Codes of local or synthesized origin are never serialized; they are included here to keep the code space in one place, and are cross-referenced from [Protocol Operation](#protoop), where the controller behavior that produces them is specified.

| Code | Name | Origin | Meaning |
|---|---|---|---|
| 0x00 | OK | wire | Success |
| 0x01 | ERR_LOCAL_NOT_FOUND | local | Never serialized; see [Protocol Operation](#protoop) |
| 0x02 | ERR_LOCAL_COLLISION | local | Never serialized; see [Protocol Operation](#protoop) |
| 0x03 | ERR_LOCAL_INVALID_STATE | local | Never serialized; see [Protocol Operation](#protoop) |
| 0x04 | ERR_TIMEOUT | synthesized | Never serialized; see [Protocol Operation](#protoop) |
| 0x05 | ERR_MALFORMED | wire | The message's payload does not parse for its Message Type |
| 0x06 | ERR_VERSION | wire | The Version in the request is not supported |
| 0x07 | ERR_IDENTITY_MISMATCH | wire | The responding Device's Collection Key/Device Key does not match the request |
| 0x08 | ERR_REGISTER_VERIFY_FAILED | synthesized | Never serialized; see [Protocol Operation](#protoop) |
| 0x09 | ERR_ACTION_TIMEOUT | synthesized | Never serialized; see [Protocol Operation](#protoop) |
| 0x0A | ERR_ACTION_FAILED | wire | The Device attempted the Action and it failed |

## Message Types

Message Type is one byte, giving 256 possible values. This document assigns 0x01 through 0x1B; the remainder are unassigned, and, per [IANA Considerations](#iana), this document establishes no registry for allocating them.

### Device Configuration Phase

| Type | Message | Direction | Payload |
|---|---|---|---|
| 0x01 | BCNP_DISCOVER | Controller to Devices (broadcast) | empty |
| 0x02 | BCNP_ANNOUNCE | Device to Controller | `[has_keys:1]` + if has_keys=1: `[collection_key:1][device_key:1]` + JSON tail |
| 0x03 | BCNP_DISCOVER_ASSIGN | Controller to Device | `[temp_device_key:1]` |
| 0x04 | BCNP_DISCOVER_ACK | Device to Controller | `[status:1]` |
| 0x05 | BCNP_REGISTER_DEVICE | Controller to Device | `[collection_key:1][temp_device_key:1][new_device_key:1]` |
| 0x06 | BCNP_REGISTER_DEVICE_ACK | Device to Controller | `[status:1]` |
| 0x07 | BCNP_FORGET_DEVICE | Controller to Device | `[collection_key:1][device_key:1]` |
| 0x08 | BCNP_FORGET_DEVICE_ACK | Device to Controller | `[status:1]` |
| 0x09 | BCNP_END_DISCOVER | Controller to Device | empty |
| 0x0A | BCNP_END_DISCOVER_ACK | Device to Controller | `[status:1]` |

### Action Configuration Phase

| Type | Message | Direction | Payload |
|---|---|---|---|
| 0x0B | BCNP_PAIR | Controller to Device | `[collection_key:1][device_key:1]` |
| 0x0C | BCNP_PAIR_ACK | Device to Controller | `[status:1]` |
| 0x0D | BCNP_DISCOVER_ACTION | Controller to Device | `[root_action_key:1]` |
| 0x0E | BCNP_ANNOUNCE_ACTION | Device to Controller | `[root_action_key:1]` + JSON tail |
| 0x0F | BCNP_DISCOVER_ALL_ACTIONS | Controller to Device | empty |
| 0x10 | BCNP_ANNOUNCE_ALL_ACTIONS | Device to Controller | JSON array, no fixed prefix |
| 0x11 | BCNP_REGISTER_ACTION | Controller to Device | `[root_action_key:1][new_action_key:1]` |
| 0x12 | BCNP_REGISTER_ACTION_ACK | Device to Controller | `[status:1]` |
| 0x13 | BCNP_FORGET_ACTION | Controller to Device | `[action_key:1]` |
| 0x14 | BCNP_FORGET_ACTION_ACK | Device to Controller | `[status:1]` |
| 0x15 | BCNP_END_PAIR | Controller to Device | empty |
| 0x16 | BCNP_END_PAIR_ACK | Device to Controller | `[status:1]` |

Messages 0x0D through 0x16 ride the single persistent connection opened by BCNP_PAIR; none carry Collection Key or Device Key, since the connection itself identifies the Device unambiguously.

### Command Mode

| Type | Message | Direction | Payload |
|---|---|---|---|
| 0x17 | BCNP_COMMAND | Controller to Device | `[collection_key:1][device_key:1][action_key:1]` |
| 0x18 | BCNP_COMMAND_ACK | Device to Controller | `[status:1]` (final result: OK or ERR_ACTION_FAILED) |
| 0x19 | BCNP_COMMAND_ACCEPT | Device to Controller | `[status:1]` (immediate receipt confirmation) |

BCNP_COMMAND_ACCEPT (0x19) is sent before BCNP_COMMAND_ACK (0x18) in every exchange, despite its higher Type value; 0x18 retained the Type value assigned to the single acknowledgment in an earlier version of this protocol, and 0x19 was appended rather than renumbering it. Both responses to a single BCNP_COMMAND MUST echo that request's Sequence ID.

### Reconciliation Mode

| Type | Message | Direction | Payload |
|---|---|---|---|
| 0x1A | BCNP_IDENTITY_CHECK | Controller to Devices (broadcast) | `[count:2]` + count x `[collection_key:1][device_key:1]` |
| 0x1B | BCNP_IDENTITY_CHECK_ACK | Device to Controller | `[collection_key:1][device_key:1]` |

`count` is encoded in network byte order. A Device receiving BCNP_IDENTITY_CHECK MUST validate that `count * 2 + 2` equals Payload Length before iterating the list; it MUST discard the message without further processing if this check fails. A Device sends BCNP_IDENTITY_CHECK_ACK only if one of the pairs in the list matches its own Collection Key and Device Key; a Device that finds no match MUST NOT respond.

## JSON Schema

BCNP_ANNOUNCE, BCNP_ANNOUNCE_ACTION, and BCNP_ANNOUNCE_ALL_ACTIONS each carry a JSON tail, encoded in UTF-8 per {{RFC8259}}. Two distinct shapes are used.

### Device Descriptor

Used as the JSON tail of BCNP_ANNOUNCE.

| Field | Type | Presence |
|---|---|---|
| name | string | REQUIRED |
| description | string | OPTIONAL |

### Action Descriptor

Used as the JSON tail of BCNP_ANNOUNCE_ACTION, and as each element of the JSON array carried by BCNP_ANNOUNCE_ALL_ACTIONS.

| Field | Type | Presence |
|---|---|---|
| name | string | REQUIRED |
| description | string | OPTIONAL |
| duration | integer | OPTIONAL |
| root_action_key | integer, 0-255 | REQUIRED in BCNP_ANNOUNCE_ALL_ACTIONS array elements only |

`duration` is expressed in milliseconds. If absent, a Controller MUST fall back to a default timeout when awaiting the Action's completion, rather than treating its absence as an error (see [Protocol Operation](#protoop)). `root_action_key` MUST NOT be present in BCNP_ANNOUNCE_ACTION's JSON tail, where it is instead carried by the fixed `[root_action_key:1]` field preceding it; its presence is REQUIRED in each element of BCNP_ANNOUNCE_ALL_ACTIONS's array, which has no such fixed prefix.

Field names use snake_case throughout, matching the naming convention of the fixed binary fields defined elsewhere in this section.


# Protocol Operation {#protoop}

## Controller States

A Controller is, at any time, in exactly one of three states.

CTRL_STATE_IDLE:
: The Controller's resting state. A Controller in this state MAY issue a Command, entering no other state as a result once the exchange completes. A Controller in this state MAY enter CTRL_STATE_DISCOVERING by initiating a Device Configuration Phase, or CTRL_STATE_PAIRED by initiating an Action Configuration Phase.

CTRL_STATE_DISCOVERING:
: Active for the duration of a Device Configuration Phase (see "Discovery Procedure" below). A Controller in this state MUST NOT issue a Command, and MUST send and accept only Device Configuration Phase messages. A Controller returns to CTRL_STATE_IDLE once the Device Configuration Phase ends.

CTRL_STATE_PAIRED:
: Active for the duration of an Action Configuration Phase, over the single persistent connection opened by BCNP_PAIR (see "Pairing Procedure" below). A Controller in this state MUST NOT issue a Command, and MUST send and accept only Action Configuration Phase messages on that connection. A Controller returns to CTRL_STATE_IDLE once the persistent connection closes.

## Device States

A Device is, at any time, in exactly one of three states, persisted across restarts except where noted.

DEV_STATE_UNASSIGNED:
: The Device's initial state, and the state a Device returns to after being forgotten. A Device in this state listens for BCNP_DISCOVER and MUST respond with BCNP_ANNOUNCE. A Device in this state MUST evaluate any BCNP_IDENTITY_CHECK it receives (it will never match, holding no Collection Key or Device Key) and MUST silently discard a BCNP_COMMAND. A Device transitions to DEV_STATE_ASSIGNED on receiving a BCNP_REGISTER_DEVICE addressed to its currently held Temporary Device Key.

DEV_STATE_ASSIGNED:
: A Device holding a Collection Key and Device Key. A Device in this state MUST respond to BCNP_COMMAND (see "Command Procedure" below), MUST respond to BCNP_DISCOVER with a BCNP_ANNOUNCE carrying its current Collection Key and Device Key, and MUST evaluate BCNP_IDENTITY_CHECK, responding with BCNP_IDENTITY_CHECK_ACK if the message includes its own Collection Key and Device Key pair. A Device in this state transitions to DEV_STATE_PAIRED on receiving a BCNP_PAIR that matches its held Collection Key and Device Key, and to DEV_STATE_UNASSIGNED on receiving a matching BCNP_FORGET_DEVICE. A Device's handling of a BCNP_REGISTER_DEVICE that does not exactly match its already-held keys is specified under "Idempotency and Retry Safety" below.

DEV_STATE_PAIRED:
: A Device with a single persistent connection open to a Controller, entered by way of a successful BCNP_PAIR. A Device in this state MUST NOT act upon any message other than an Action Configuration Phase message received via its open persistent connection; in particular, it MUST NOT act upon a BCNP_COMMAND, regardless of source. A Device in this state MUST close the connection and return to DEV_STATE_ASSIGNED after receiving BCNP_END_PAIR, or after an idle period with no message received from the Controller exceeding a locally configured threshold; the idle period MUST reset on any message received from the Controller, not merely on connection establishment. A Device MUST NOT persist DEV_STATE_PAIRED across a restart: a restarted Device always begins in DEV_STATE_UNASSIGNED or DEV_STATE_ASSIGNED depending on the Collection Key and Device Key recorded in its persistent storage. A Device's handling of a BCNP_PAIR received on the already-open connection is specified under "Idempotency and Retry Safety" below.

## Unrecognized Messages

Except where a Controller or Device state's description above specifies otherwise, an implementation MUST silently discard any message that does not apply to its current state.

## Local and Synthesized Errors

Six Status Codes are never carried on the wire: a Controller produces each internally, as its own conclusion, without any corresponding message from a Device. Four are explained here; the remaining two, ERR_REGISTER_VERIFY_FAILED and ERR_ACTION_TIMEOUT, are explained where they naturally arise, under "Register Device Verification" and "Command Procedure" respectively.

ERR_LOCAL_NOT_FOUND:
: Produced when an operation references a Collection Key, Device Key, or Action Key that is not present in the Controller's Device Map or Action Map. Because these maps are authoritative for a Controller operating under the single-Controller assumption in [Protocol Overview](#overview), this check MUST be performed, and MUST fail, before any message is sent; no Device is contacted.

ERR_LOCAL_COLLISION:
: Produced when a Register operation's target, the new Device Key in BCNP_REGISTER_DEVICE or the new Action Key in BCNP_REGISTER_ACTION, is already occupied in the Controller's Device Map or Action Map. As with ERR_LOCAL_NOT_FOUND, this check MUST be performed locally before any message is sent.

ERR_LOCAL_INVALID_STATE:
: Produced when a Probe is invoked that does not apply to the Controller's current state (for example, a Register Probe invoked while CTRL_STATE_IDLE; see "Controller States" above). No message is sent.

ERR_TIMEOUT:
: Produced when a Controller does not receive an expected response to a message it sent, within its configured timeout for that exchange. Unlike the three codes above, this follows an attempt that did reach the network; see "Reconciliation" below for how a Controller acts on it.


## Discovery Procedure

A Controller enters CTRL_STATE_DISCOVERING by broadcasting BCNP_DISCOVER to its network, using a broadcast mechanism appropriate to its network layer rather than a connection to each address individually. Every Device in DEV_STATE_UNASSIGNED or DEV_STATE_ASSIGNED that receives this broadcast responds with BCNP_ANNOUNCE, opening its own short-lived connection to do so; a Device in DEV_STATE_PAIRED does not respond. Every message in this phase is sent over its own short-lived connection, opened for that one exchange and closed afterward; a Controller MUST NOT hold a connection open across multiple Discovery-phase exchanges with the same Device.

For each responding Device, the Controller assigns a Temporary Device Key by sending BCNP_DISCOVER_ASSIGN, recording the Device's Network ID against that key in its Temporary Device Map on receiving BCNP_DISCOVER_ACK.

Once addressed by its Temporary Device Key, a Controller assigns a Device a permanent Collection Key and Device Key by sending BCNP_REGISTER_DEVICE. On receiving BCNP_REGISTER_DEVICE_ACK with Status Code OK, a Controller MUST verify the registration before recording it in its Device Map, using the mechanism described under "Idempotency and Retry Safety" below, rather than committing on the acknowledgment alone.

A Controller removes a Device's registration by sending BCNP_FORGET_DEVICE, addressed by that Device's Collection Key and Device Key as currently recorded in its Device Map, removing the corresponding entry on receiving BCNP_FORGET_DEVICE_ACK.

A Controller ends a Device Configuration Phase by sending BCNP_END_DISCOVER individually to every Device remaining in its Temporary Device Map. Once every such Device has responded with BCNP_END_DISCOVER_ACK, the Controller discards its Temporary Device Map and returns to CTRL_STATE_IDLE.

### Key Addressing and Pagination

Temporary Device Key here, and Root Action Key during Pairing (see "Pairing Procedure" below), may both need to address more candidates than K. Both use the same mechanism: a Controller maintains an ordered list of candidates wider than K, of which only a window of K (one page) is addressable by a Key Probe at a time.

The Device probe, which selects the current Device context in CTRL_STATE_IDLE, takes on a different meaning in CTRL_STATE_DISCOVERING and during Action discovery within CTRL_STATE_PAIRED: invoked without a Key Probe, it advances to the next page. Paging wraps: invoking it past the last page, whether full or partial, returns to the first page.

Here, advancing the page reassigns Temporary Device Key values 0 to K-1 to the next page's Devices by sending each a fresh BCNP_DISCOVER_ASSIGN, using the Network ID already learned from that Device's BCNP_ANNOUNCE; a Controller MUST NOT need to re-broadcast BCNP_DISCOVER to advance a page. Devices outside the current page are simply not addressed until the Controller pages back to them.

## Pairing Procedure

A Controller enters CTRL_STATE_PAIRED by sending BCNP_PAIR to a Device already recorded in its Device Map, opening the single persistent connection that remains open for the duration of this phase. BCNP_PAIR carries the Collection Key and Device Key the Controller believes it is addressing; a Device receiving it MUST compare these against its own held Collection Key and Device Key and reject with ERR_IDENTITY_MISMATCH on any mismatch, rather than pairing with whichever Controller happened to reach it.

Once paired, a Controller queries a single Action by sending BCNP_DISCOVER_ACTION with the Action's Root Action Key, resolved from the current page if pagination is in use (see "Key Addressing and Pagination" above), receiving its metadata back in BCNP_ANNOUNCE_ACTION. A Controller queries every Action a Device offers at once by sending BCNP_DISCOVER_ALL_ACTIONS, receiving the full set back in a single BCNP_ANNOUNCE_ALL_ACTIONS; this exchange is not subject to pagination, since it addresses no single Action Key.

A Controller assigns an Action Key to a Device's Action by sending BCNP_REGISTER_ACTION, and removes an assignment by sending BCNP_FORGET_ACTION, addressed by the already-assigned Action Key. As with BCNP_REGISTER_DEVICE, a Controller MUST verify a BCNP_REGISTER_ACTION_ACK carrying Status Code OK using the mechanism described under "Idempotency and Retry Safety" below before recording it.

A Controller ends an Action Configuration Phase by sending BCNP_END_PAIR over the open connection, closing the connection on receiving BCNP_END_PAIR_ACK and returning to CTRL_STATE_IDLE. A Device's own timeout and connection-loss behavior while paired is specified under "Device States" above.

## Command Procedure

A Controller in CTRL_STATE_IDLE elicits an Action by sending BCNP_COMMAND, addressed by Collection Key, Device Key, and Action Key, over its own short-lived connection. A Controller MUST NOT issue a further Command, or enter CTRL_STATE_DISCOVERING or CTRL_STATE_PAIRED, until the exchange completes or fails.

A Device receiving BCNP_COMMAND responds immediately with BCNP_COMMAND_ACCEPT, before performing the Action, to confirm receipt; Status Code OK indicates the Device will attempt the Action, and any other code indicates outright rejection, in which case no further message follows. Once the Action completes, the Device sends BCNP_COMMAND_ACK carrying the outcome: Status Code OK on success, or ERR_ACTION_FAILED if the Action was attempted and did not succeed. Both messages MUST echo the Sequence ID of the BCNP_COMMAND they answer.

A Controller awaits BCNP_COMMAND_ACCEPT for a short, fixed timeout; failing to receive it is handled as specified under "Idempotency and Retry Safety" below. Once BCNP_COMMAND_ACCEPT has been received, a Controller awaits BCNP_COMMAND_ACK for a duration derived from the Action's declared duration (see "JSON Schema" in Message Format) plus a grace period, or for a locally configured default duration if the Action declared none; failing to receive it within that window is a distinct condition, ERR_ACTION_TIMEOUT, also specified under "Idempotency and Retry Safety" below.

## Idempotency and Retry Safety

### Idempotent Operations

A repeated BCNP_REGISTER_DEVICE, BCNP_REGISTER_ACTION, BCNP_FORGET_DEVICE, BCNP_FORGET_ACTION, or BCNP_PAIR that exactly matches state a Device already holds MUST be treated as successful and re-acknowledged with Status Code OK, not dropped or treated as an error; a Controller MAY therefore retry any of these operations without first determining whether an earlier attempt succeeded. A request that instead conflicts with state a Device already holds (for example, a BCNP_REGISTER_DEVICE referencing a different Collection Key or new Device Key than one already assigned to the addressed Temporary Device Key) MUST be rejected with ERR_IDENTITY_MISMATCH.

BCNP_COMMAND is not idempotent: performing an Action twice is not, in general, equivalent to performing it once. A Controller MUST NOT automatically retry a BCNP_COMMAND that has timed out or failed; the decision to reissue it is left to whatever invoked the Controller.

### Reconciliation

A Controller recovers from a failed or ambiguous operation by sending BCNP_IDENTITY_CHECK, querying whether a Device holding a specific Collection Key and Device Key pair, or several such pairs batched into a single message, can still be located, regardless of Network ID. A Controller triggers Reconciliation:

* on ERR_TIMEOUT or ERR_IDENTITY_MISMATCH while issuing a Command or during Pairing, for the specific Collection Key and Device Key pair involved;
* on ERR_TIMEOUT while removing a Device's registration with BCNP_FORGET_DEVICE, for the Collection Key and Device Key being removed;
* on startup, proactively, for every entry in its Device Map at once.

A Controller does not attempt a Device's last-known Network ID again before broadcasting BCNP_IDENTITY_CHECK: the operation that triggered Reconciliation has already served that purpose. A Controller SHOULD retry BCNP_IDENTITY_CHECK up to three times, approximately two seconds apart; the exact retry count and interval are implementation-defined. If BCNP_IDENTITY_CHECK_ACK is received, confirming the Device is still reachable, a Controller MAY retry the operation that triggered Reconciliation, since Register, Forget, and Pair are all idempotent (see "Idempotent Operations" above). If no BCNP_IDENTITY_CHECK_ACK is received after these retries, the Controller removes the corresponding entry from its Device Map, freeing the Device Key for reuse. A subsequent reference to a removed entry is rejected locally with ERR_LOCAL_NOT_FOUND, without generating any further network traffic.

Reconciliation applies only to operations addressed by Collection Key and Device Key. BCNP_DISCOVER_ASSIGN and BCNP_END_DISCOVER are addressed by Temporary Device Key or by neither, and their timeout is handled without Reconciliation: a Device that does not acknowledge BCNP_DISCOVER_ASSIGN is simply not added to the Controller's Temporary Device Map for this Device Configuration Phase, and a Controller MAY proceed without receiving BCNP_END_DISCOVER_ACK, since no permanent Collection Key or Device Key registration depends on it.

### Register Device Verification

Because Device Key is drawn from a range of only K values, shared across an entire Collection, a registration a Controller believes succeeded but that did not actually persist on the Device (for example, because the Device restarted between sending its acknowledgment and completing the write to its own persistent storage) would otherwise occupy a scarce key indefinitely, undetected. The same uncertainty arises if BCNP_REGISTER_DEVICE_ACK is never received at all: the registration may have succeeded regardless, with only the acknowledgment lost. To guard against both cases, a Controller MUST, before recording a registration in its Device Map, send BCNP_IDENTITY_CHECK for the Collection Key and Device Key just attempted: immediately on receiving BCNP_REGISTER_DEVICE_ACK carrying Status Code OK, or on ERR_TIMEOUT if no acknowledgment arrives at all. If BCNP_IDENTITY_CHECK_ACK confirms them, the Controller records the registration; otherwise, it MUST NOT record it, and MUST report ERR_REGISTER_VERIFY_FAILED. A Controller MAY safely retry BCNP_REGISTER_DEVICE after such a failure, since the operation is idempotent.

This verification is scoped to BCNP_REGISTER_DEVICE only. BCNP_IDENTITY_CHECK can confirm a Collection Key and Device Key pair, but has no equivalent for an Action Key: there is no message in this document capable of confirming that a Device's Action Map actually holds a given Action Key to Root Action Key mapping. A Controller therefore has no way to verify a BCNP_REGISTER_ACTION_ACK the way it verifies BCNP_REGISTER_DEVICE_ACK; the same phantom-acknowledgment risk described above applies to Action Key, unguarded, and is instead discovered the way a stale BCNP_FORGET_DEVICE entry is: a subsequent Command referencing the affected Action Key fails, triggering the error handling specified in "Command Procedure" above, at which point the Controller and its operator learn the registration did not hold. This is a deliberate asymmetry rather than an oversight: Device Key is shared across an entire Collection and is worth the stronger guarantee; Action Key affects only the single Device it belongs to.

A BCNP_FORGET_DEVICE or BCNP_FORGET_ACTION whose acknowledgment is lost requires no equivalent verification step: a Device that has genuinely forgotten an entry will not answer a later BCNP_IDENTITY_CHECK for it, so ordinary Reconciliation above already removes the resulting stale Device Map entry on next use.


# Security Considerations {#security}

### Trust Model and Scope

BCNP's security properties depend entirely on the deployment model described under "Deployment Model" in [Protocol Overview](#overview): a single Controller establishes its own wireless network, and Devices join that network through an out-of-band mechanism outside the scope of this document. This document treats that network, not any individual BCNP message, as the trust boundary. Every message defined in [Message Format](#msgformat) is unauthenticated and unencrypted at the BCNP layer; whatever confidentiality and access control exist are provided entirely by the wireless network's own link-layer security (for example, WPA2 or WPA3), not by BCNP itself.

This scoping has a direct consequence: BCNP provides no protection whatsoever against a party who is not on the Controller's wireless network. It relies entirely on that network correctly excluding anyone who has not been given credentials to join it. BCNP also provides no protection against a device that is a legitimate, credentialed member of that network but is compromised or malicious; this residual risk is discussed below.

BCNP is not designed for, and this document makes no claims about, operation on a network the Controller does not establish and control: for example, a household's existing general-purpose WiFi network shared with other, unrelated devices and users.

### Residual Risk Within the Trust Boundary

No message defined in this document carries any credential, and a Device accepts BCNP_PAIR, BCNP_REGISTER_DEVICE, BCNP_REGISTER_ACTION, and BCNP_COMMAND from any sender reachable on the Controller's network, without verifying that sender's identity beyond the checks against Collection Key and Device Key described in [Protocol Operation](#protoop). Consequently, any device that has joined the Controller's wireless network (including one that is compromised, whether through a firmware vulnerability, a supply-chain issue, or any other means) can impersonate the Controller to another Device, or impersonate a Device to the Controller. This is an accepted risk for the version of BCNP specified in this document, not an oversight: WPA2/WPA3 network membership demonstrates only that a station knows the network's credentials, not that it is specifically the Controller or specifically a particular Device, and this document does not attempt to close that gap at the BCNP layer.

A future extension to this protocol may add a per-Device shared secret, established during BCNP_REGISTER_DEVICE and used to authenticate subsequent messages to and from that Device (for example, via a keyed hash over each message), closing this gap without requiring a full public-key infrastructure. Such a mechanism would need to be paired with replay protection (for example, a Device rejecting any Sequence ID at or below the last one it accepted from its Controller), since message authentication alone does not prevent a captured, valid message from being replayed verbatim. This document does not specify either mechanism.

### Denial of Service

Two denial-of-service concerns are worth addressing explicitly, closed by the idle timeout and pagination mechanisms specified in [Protocol Operation](#protoop) rather than by any dedicated defense.

A Device in DEV_STATE_PAIRED rejects Action Configuration Phase messages, and Commands, from any connection other than the one already open. Because BCNP_PAIR carries no credential (see "Residual Risk Within the Trust Boundary" above), any device on the Controller's network can open this connection by impersonating the Controller, and could hold it open indefinitely, denying the legitimate Controller access to a Device it should otherwise be able to reach. The idle timeout specified under "Device States" bounds this exposure: a Device MUST end an Action Configuration Phase after an idle period with no message received, regardless of why that silence occurred, without needing to distinguish a hostile connection from a merely idle legitimate one.

A Controller's Temporary Device Key space is limited to K entries at a time. Without the pagination mechanism specified under "Key Addressing and Pagination," a Device Configuration Phase in which more than K Devices respond to a single BCNP_DISCOVER would leave the excess Devices permanently unable to obtain a Temporary Device Key. Pagination resolves this by design, not as a security-specific addition: every responding Device eventually becomes addressable as the Controller pages through its full candidate list.


# IANA Considerations {#iana}

BCNP does not request a port number assignment from IANA. TCP port 49400 is used as a suggested default; implementations and deployments MAY configure a different port as needed. BCNP is designed to operate on a closed, controller-hosted local network rather than requiring a globally coordinated well-known port.

This document defines a fixed set of Message Type values and a fixed set of status/error codes. No IANA registry is established for either; both remain closed and are considered part of this specification rather than open to extension via IANA registration. Any future extension to either set requires revising this document directly.


--- back

# Example Input Mechanisms

Any input mechanism capable of producing 7 + K reliably distinguishable signals can drive a BCNP Controller. For a brain-computer interface, this may be a set of K + 7 distinct motor imagery classes (for example, imagining movement of different limbs), or a set of K + 7 distinct imagined-speech classes; nothing in this document assumes one over the other, or assumes a brain-computer interface at all.

# Example Device Onboarding

This document assumes Devices join the Controller's wireless network before engaging in BCNP (see "Deployment Model"), but specifies no mechanism for doing so. A Device capable of joining any WPA2 or WPA3 network by ordinary means (for example, WPS, a QR code, or manual credential entry) can join the Controller's network the same way, since it is an ordinary WPA2/WPA3 network from the Device's perspective.

# Acknowledgments
{:numbered="false"}

I thank Professor Abeer T. Khalil, of the Electronics and Communication Department, Faculty of Engineering, Mansoura University, who supervised this work as a graduation project and reviewed the protocol design.
