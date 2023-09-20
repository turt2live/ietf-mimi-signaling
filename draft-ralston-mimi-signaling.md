---
title: "MIMI Signaling Protocol"
abbrev: "MIMI Signaling Protocol"
category: std

docname: draft-ralston-mimi-signaling-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
keyword:
 - mimi
 - signaling
 - protocol
 - layer
 - messaging
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "turt2live/ietf-mimi-signaling"
  latest: "https://turt2live.github.io/ietf-mimi-signaling/draft-ralston-mimi-signaling.html"

author:
 -
    fullname: Travis Ralston
    organization: The Matrix.org Foundation C.I.C.
    email: travisr@matrix.org
 -
    fullname: Matthew Hodgson
    organization: The Matrix.org Foundation C.I.C.
    email: matthew@matrix.org

normative:
  RFC4648:

informative:
  RFC8446:


--- abstract

The MIMI signaling protocol describes a framework for user-level interactions in
the overall MIMI protocol stack. The event structure can be used by control protocols
described by {{!I-D.barnes-mimi-arch}}.

--- middle

# Introduction

The More Instant Messaging Interoperability (MIMI) working group is responsible for
specifying a protocol to achieve interoperability among modern messaging providers.
Most providers do not currently support Messaging Layer Security (MLS) {{?RFC9420}},
a chartered requirement of MIMI, but do support other forms of encryption alongside
their existing signaling.

Within the MIMI stack of protocols, signaling is responsible for coordinating user-level
actions and participation. This document describes such a protocol, encompassing
a framework for control protocols to operate on top of.

An overview of the architecture is described by {{!I-D.barnes-mimi-arch}}. Specific
implementations of policy, participation, and security control protocols are *not*
described by this document.

MIMI has a chartered requirement to use MLS in the encryption/security protocol, however
most existing messaging providers do not currently use MLS in their encryption. This
document describes an approach for enabling a given encryption/security protocol without
concerning itself with the specifics of such a protocol. This allows existing messaging
providers to insert their own encryption/security protocols external to the MIMI working
group.

As described by {{!I-D.barnes-mimi-arch}}, events are sent in rooms to accomplish state
changes and messaging. This document defines how events are copied between servers, and
where control protocols become involved.

This document describes an extensible approach to room policy, similar to how different
encryption/security layers can be used within a room. The policy cannot change once set,
though it can be configured using state events.

A create state event is used to set the policy and encryption/security protocol. This
create event is the very first event in the room - all other events are linearly placed
after it.

Abstract concepts are used for transport and persistence of rooms/events. For example,
events can be persisted within an MLS group or serialized in a different format than
described in this document.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms from {{!I-D.barnes-mimi-arch}} and {{!I-D.ralston-mimi-terminology}} are used
throughout this document. {{!I-D.barnes-mimi-arch}} takes precedence where there's conflict.

Other terms include:

*Unpadded Base64*: Serialization as described by {{Section 4 of RFC4648}} with padding (`=`
characters) removed.

*URL-Safe Base64*: As described by {{Section 5 of RFC4648}}. May also be unpadded, as above.

*Redaction*: An algorithm to remove fields irrelevant to the overall protocol's operation.
This should not be confused with message deletion/removal, where a user wishes to delete
content they previously sent in the room. Redaction is reserved for protocol operation
exclusively. See {{int-redactions}}.

*Content Hash*: A SHA256 {{!RFC6234}} hash covering the unredacted event. See {{int-content-hash}}.

*Reference Hash*: A SHA256 {{!RFC6234}} hash covering the essential fields of an event,
including the content hash. See {{int-reference-hash}}.

# Room Structure

Rooms consist of an identifier and linear linked-list of events. The first event MUST be
a create event ({{int-ev-create}}) to establish the policy and encryption/security protocol
the room uses. The second event MUST be the user participation event ({{int-ev-user}}) for
the creator. All other events follow, and are sequenced into the linked-list by the hub server.

The linked-list represents the set of accepted events in the room. Rejected or illegal events
are excluded. The hub server MUST enforce the room's policy upon the room. Follower servers
SHOULD additionally enforce the policy, preventing the hub from accepting illegal events.

Events point to their parent event, as sequenced by the hub, and the policy-configuring state
events which prove the event is allowed in the room (the event's "auth events"). The create
event MUST be included in the auth events, unless the event is the create event itself. Other
examples of auth events include the sender's participation event and current permissions event.

This structure allows a server to discard events it no longer needs, but can easily be found
later. For example, a follower server might only persist the create event and a dozen most
recent events. If the server then has a need to "backfill" then it can simply use the `prevEvent`
pointer off the oldest (non-create) event it has until eventually hitting the intended mark
or create event.

A hub server, however, MUST persist the entire chain of events from `m.room.create` onwards.
This is to guarantee that at least one server in the room has the full history for the room
available.

In diagram form, a room looks as such:

~~~aasvg
+---------------+   +-------------+   +------------------+
| m.room.create |<--| m.room.user |<--| m.room.encrypted |
+---------------+   +-------------+   +------------------+
       ^                    ^                   |
       +--------------------+---auth events-----+
~~~
{: #overview title="A room made up of events." }

The `m.room.encrypted` event (not specified in this document) has the `m.room.user` event
as its only previous event, but points at both the `m.room.user` and `m.room.create` events
as auth events. In practice, a room would likely have more `m.room.user` events to represent
other users in the room, rather than this example user conversing with themselves.

**TODO(TR): Should we replace room IDs with the create event's ID?
[[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/1)]**

# Events {#int-events}

The concept of events is described by Section 5.1 of {{!I-D.barnes-mimi-arch}}. An event is
authenticated against its TLS presentation language format ({{Section 3 of RFC8446}}):

~~~
// Consists of a server's signatures using their signing key. The transport
// specification defines which specific signing key algorithm is used. This
// document describes *what* a signature covers.
opaque Signatures;
struct {
  // where the event is sent to
  opaque roomId;

  // the type of event. Defines format for content
  // See "Event Types" IANA registry.
  opaque type;

  // if present, the event is a state event
  opaque [[stateKey]];

  // who or what sent this event
  opaque sender;

  // binary type-specific event content (TODO(TR): type??)
  opaque content;

  // the event IDs of the auth events
  opaque authEvents[];

  // the parent event ID
  opaque prevEvent;

  // the origin server's content hash of the event
  uint8 originContentHash[32];

  Signatures hubSignature;
  Signatures originSignature;
} Event;
~~~

**TODO(TR): Should we bring over origin_server_ts?
[[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/2)]**

**TODO(TR): Maximum lengths? (or is this a transport/not-us problem?)
[[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/3)]**

Note that an "event ID" is not specified on the object. The event ID for an event is the
sigil `$` followed by the URL-Safe Unpadded Base64-encoded reference hash ({{int-reference-hash}})
of the event.

The "origin server" of an event is the server denoted/implied by the `sender`.

**TODO(TR): Do we need to describe how events are extensible? ie: being able to add things
to the m.room.create event content.
[[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/4)]**

## Reference Hash {#int-reference-hash}

Events are referenced by ID in relation to each other, forming the room history and auth
chain. If the event ID was a sender-generated value, any server along the send or receive
path could "replace" that event with another perfectly legal event, both using the same ID.

By using a calculated value, namely the reference hash, if a server does try to replace the
event then it would result in a completely different event ID. That event ID becomes impossible
to reference as it wouldn't be part of the proper room history.

To calculate a reference hash, the event is first redacted ({{int-redactions}}) alongside
`hubSignature` and `originSignature` fields being removed. The resulting binary is then
hashed using SHA256 {{!RFC6234}}.

To further create an event ID, the resulting hash is encoded using URL-Safe Unpadded Base64
and prefixed with the `$` sigil.

## Content Hash {#int-content-hash}

An event's content hash prevents servers from modifying details of the event not covered
by the reference hash itself. For example, a room name state event doesn't have the name
itself covered by a reference hash because it's redacted, so it's instead covered by the
content hash, which is in turn covered by the reference hash. This allows the event to
later be redacted without affecting the event ID of that event.

To calculate a content hash, the following fields are removed from the event first:

* `originContentHash`
* `hubSignature`
* `originSignature`
* `prevEvent`
* `authEvents`

`authEvents` and `prevEvent` are removed because they are populated by the hub server. The
content hash is to preserve the origin server's event, not the hub server's.

The resulting binary is then hashed using SHA256 {{!RFC6234}}.

## Redaction {#int-redactions}

The process of removing fields from an event which aren't important for the overall protocol
to operate is known as "redaction". This document protects fields of events required for
signaling to work while a policy document or encryption/security protocol will define their
own fields. For example, the MLS ciphertext for a proposal or commit might be preserved through
redaction to avoid breaking the MLS group for future participants.

The default behaviour for any event type is to remove all fields *not* specified by {{int-events}},
excluding `content`. Individual event types MAY specify their own redaction behaviour for `content`.

# Room Metadata

Information about the room such as its name, topic (description), avatar, etc is described by state
events. State events are also used for policy control configuration, as specified elsewhere in this
document.

This document describes the following event types for room metadata purposes.

## `m.room.name` {#int-ev-name}

**State key**: Empty string.

**Content**:

~~~
struct {
  // human-readable name for the room. May be empty or missing to denote
  // no name.
  opaque name;
} MRoomNameEventContent;
~~~

**Redaction considerations**: None. It is considered a feature that the room name can be redacted
to remove unwanted room names. A redacted `m.room.name` event is treated the same as a room without
an `m.room.name` event present.

## `m.room.avatar` {#int-ev-avatar}

**State key**: Empty string.

**Content**:

~~~
struct {
  // human-usable image to differentiate the room visually.
  // TODO(TR): Do we know how images are being sent in MIMI?
  opaque mediaUri;
} MRoomAvatarEventContent;
~~~

**Redaction considerations**: None. It is considered a feature that the room avatar can be redacted
to remove unwanted room avatars. A redacted `m.room.avatar` event is treated the same as a room without
an `m.room.avatar` event present.

## `m.room.topic` {#int-ev-topic}

**State key**: Empty string.

**Content**:

~~~
struct {
  // human-readable description for the room.
  opaque topic;
} MRoomTopicEventContent;
~~~

**Redaction considerations**: None. It is considered a feature that the room topic can be redacted
to remove unwanted room topics. A redacted `m.room.topic` event is treated the same as a room without
an `m.room.topic` event present.

# Protocol Events

## `m.room.create` {#int-ev-create}

**State key**: Empty string.

**Content**:

~~~
struct {
  // The policy envelope the room is using.
  // See the "Policy IDs" IANA registry.
  opaque policyId;

  // The encryption/security algorithm name in use for the room.
  // TODO(TR): Does this also need an IANA registry? Currently we expect
  // this to be `m.mls` or something, maybe with ciphersuite encoded or
  // defined adjacent?
  opaque encryptionAlgorithm;
} MRoomCreateEventContent;
~~~

**Redaction considerations**: All `content` is *not* redacted.

## `m.room.user` {#int-ev-user}

**State key**: The user ID the participation event affects.

**Content**:

~~~
struct {
  // The participation state the user is in. Legal values are described by
  // the policy envelope for the room.
  opaque state;
} MRoomUserEventContent;
~~~

**Redaction considerations**: `state` under `content` is protected from redaction.

# Transport

**TODO(TR): Link to I-D.kohbrok-mimi-transport** describes a series of REST endpoints
for communicating between servers. The following transaction types are defined with
respect to signaling.

## Sending Events

**TODO(TR): Specifics. [[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/5)]**

* One event at a time
* Mark whether it's hub->follower, or follower->hub
* When follower->hub, mention that some fields are excluded on the event
  * Hub then validates the event (proper shape, legal fields, enforce policy)
  * If valid, hub appends its signatures and other fields, and adds it to the room
  * Added events are then fanned out to all relevant servers
* Unordered sequencing, re-assembled from `prevEvent`

## Retrieving Events

If a server notices that it missed an event, or simply wishes to re-request a
particular event, it can use the following operations.

**TODO(TR): Specifics. [[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/5)]**

* Get event by ID
* Get state event by type & state key
* Get batch of previous events?

# IANA Considerations

IANA creates the following registries:

* Event Types
* Policy IDs

## Event Types Registry

**TODO(TR): Is this what IANA actually wants? [[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/6)]**

An event type is used to determine the type of `content` being carried by an event. The type MAY
further influence policy, participation, or other aspects of the overall MIMI stack. For example,
an event type for sending an encrypted MLS application message may be specified.

Event types MUST be prefixed with a reverse domain namespace (i.e.: `org.example.my_event`). Event
types starting with `m.` are reserved for use within the MIMI working group. Event types are issued
on a first-come first-serve basis.

Each event type registration MUST be accompanied by a `content` definition. Registrations MUST also
indicate whether the event type is for a state event, and if so what describes a legal `stateKey`.
An event type MUST NOT be used for both a state event and non-state event.

The following event types and their `content` definitions are reserved as the first entries in this
registry:

* `m.room.create` ({{int-ev-create}})
* `m.room.user` ({{int-ev-user}})
* `m.room.name` ({{int-ev-name}})
* `m.room.avatar` ({{int-ev-avatar}})
* `m.room.topic` ({{int-ev-topic}})

### MIMI Namespace Conventions

Events in the `m.` namespace SHOULD use `m.room.` as a prefix to account for future flexibility and
expansion.

## Policy IDs

**TODO(TR): Is this what IANA actually wants?[[GH issue](https://github.com/turt2live/ietf-mimi-signaling/issues/6)]**

A policy ID is the identifier to describe the policy envelope the room is using. Policy IDs MUST be
prefixed with a reverse domain namespace (i.e.: `org.example.my_event`). Policy IDs starting with
`m.` are reserved for use within the MIMI working group. Policy IDs are issued on a first-come
first-serve basis.

This document does not describe any initial entries for this registry.

### MIMI Namespace Conventions

Policy IDs in the `m.` namespace SHOULD be suffixed with an integer to denote relative ordering/stability.
For example, `m.0` for an initial "version" of the policy envelope.

--- back
