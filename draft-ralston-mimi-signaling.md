---
title: "MIMI Signaling Protocol"
abbrev: "MIMI Signaling Protocol"
category: info

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

The MIMI signaling protocol is responsible for user-level interactions in the overall
MIMI protocol stack. It implements a *membership control protocol* and methods for a
*policy control protocol* to operate, as described by {{!I-D.barnes-mimi-arch}}.

--- middle

# Introduction

The More Instant Messaging Interoperability (MIMI) working group is responsible for
specifying a protocol to achieve interoperability among modern messaging providers.
Most providers do not currently support Messaging Layer Security (MLS) {{?RFC9420}},
a chartered requirement of MIMI, but do support other forms of encryption alongside
their existing signaling.

Within the MIMI stack of protocols, signaling is responsible for coordinating user-level
actions and membership. This document describes such a protocol, encompassing a membership
control protocol and a framework for a policy control protocol.

An overview of the architecture, including the control protocols mentioned, is described
by {{!I-D.barnes-mimi-arch}}. A proposed policy control protocol running over this
signaling protocol is described by **TODO(TR): Policy draft**.

MIMI has a chartered requirement to use MLS in the encryption/security protocol, however
most existing messaging providers do not currently use MLS in their encryption. This
document describes an approach for enabling a given encryption/security protocol without
concerning itself with the specifics of such a protocol. This allows existing messaging
providers to insert their own encryption/security protocols external to the MIMI working
group.

As described by {{!I-D.barnes-mimi-arch}}, events are sent in rooms to accomplish state
changes and messaging. This document defines how events become "accepted" into the room
by enforcing policy, and how state events can configure the policy envelope for future
enforcement.

This document describes an extensible approach to room policy, similar to how different
encryption/security layers can be used within a room. The policy cannot change once set,
though it can be configured using state events. This document describes some baseline
policy aspects, but otherwise defers the majority of policy to other documents.

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

*Redaction*: An algorithm to remove fields irrelevant to the signaling protocol's operation.
This should not be confused with message deletion/removal, where a user wishes to delete
content they previously sent in the room. Redaction is reserved for protocol operation
exclusively. See {{int-redactions}}.

*Content Hash*: A SHA256 {{!RFC6234}} hash covering the unredacted event. See {{int-content-hash}}.

*Reference Hash*: A SHA256 {{!RFC6234}} hash covering the essential fields of an event,
including the content hash. See {{int-reference-hash}}.

# Room Structure

Rooms consist of an identifier and linear linked-list of events. The first event MUST be
a create event ({{int-ev-create}}) to establish the policy and encryption/security protocol
the room uses. The second event MUST be the user membership event ({{int-ev-member}}) for
the creator. All other events follow, and are sequenced into the linked-list by the hub server.

The linked-list represents the set of accepted events in the room. Rejected or illegal events
are excluded. The hub server MUST enforce the room's policy upon the room. Follower servers
SHOULD additionally enforce the policy, preventing the hub from accepting illegal events.

Events point to their parent event, as sequenced by the hub, and the policy-configuring state
events which prove the event is allowed in the room (the event's "auth events"). The create
event MUST be included in the auth events, unless the event is the create event itself. Other
examples of auth events include the sender's membership event and current permissions event.

This structure allows a server to discard events it no longer needs, but can easily be found
later. For example, a follower server might only persist the create event and a dozen most
recent events. If the server then has a need to "backfill" then it can simply use the `prevEvent`
pointer off the oldest (non-create) event it has until eventually hitting the intended mark
or create event.

In diagram form, a room looks as such:

~~~aasvg
+---------------+   +---------------+   +------------------+
| m.room.create |<--| m.room.member |<--| m.room.encrypted |
+---------------+   +---------------+   +------------------+
       ^                    ^                     |
       +--------------------+---------------------+
~~~
{: #overview title="A room made up of events." }

The `m.room.encrypted` event (not specified in this document) has the `m.room.member` event
as its only previous event, but points at both the `m.room.member` and `m.room.create` events
as auth events. In practice, a room would likely have more `m.room.member` events to represent
other users in the room, rather than this example user conversing with themselves.

**TODO(TR): Should we replace room IDs with the create event's ID?**

# Events {#int-events}

The concept of events is described by Section 5.1 of {{!I-D.barnes-mimi-arch}}. An event is
authenticated against its TLS presentation language format ({{Section 3 of RFC8446}}):

~~~
// Consists of a server's signatures using their signing key. The transport
// specification defines which specific signing key algorithm is used. This
// document describes *what* a signature covers.
opaque Signatures;
struct {
  opaque roomId;               // where the event is sent to
  opaque type;                 // the type of event. Defines format for content
  opaque [[stateKey]];         // if present, the event is a state event
  opaque sender;               // who or what sent this event
  opaque content;              // binary event content (TODO(TR): type??)
  opaque authEvents[];         // the event IDs of the auth events
  opaque prevEvent;            // the parent event ID
  uint8 originContentHash[32]; // the origin server's content hash of the event
  Signatures hubSignature;
  Signatures originSignature;
} Event;
~~~

**TODO(TR): Should we bring over origin_server_ts?**

**TODO(TR): Maximum lengths? (or is this a transport problem?)**

Note that an "event ID" is not specified on the object. The event ID for an event is the
sigil `$` followed by the URL-Safe Unpadded Base64-encoded reference hash ({{int-reference-hash}})
of the event.

The "origin server" of an event is the server denoted/implied by the `sender`.

## Reference Hash {#int-reference-hash}

Events are referenced by ID in relation to each other, forming the room history and auth
chain. If the event ID was a sender-generated value, any server along the send or receive
path could "replace" that event with another perfectly legal event, both using the same ID.

By using a calculated value, namely the reference hash, if a server does try to replace the
event then it would result in a completely different event ID. That event ID becomes impossible
to reference as it wouldn't be part of the proper room history.

**TODO(TR): There might be a bug in here. Even with a reference hash, the malicious event
is only made illegal upon discovery of a later event. This means we end up having to evict
an event or two, which breaks append-only semantics.**

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

The default behaviour for any event type is to remove all fields *not* specified by {{int-events}}
plus `content`. Individual event types MAY specify their own redaction behaviour for `content`.

## State Events

**TODO**: This. What are they?

## `m.room.create` {#int-ev-create}

**TODO**: This.

## `m.room.user` {#int-ev-user}

**TODO**: This.

# History Model

**TODO**: This. Short version is servers only need the auth chain for all current members of
the room upon joining (we can discard people who joined and already left, and old message events).
Someone needs to keep a copy of each event somewhere though, otherwise the room breaks.

# [TODO: Sections]

* Event schema/shape. Do not require JSON, through signing might need a canonical format.
  * Including hashes and event ID behaviours
* Join, leave, etc operations overview.
* `[User ->] Follower -> Hub` send path.
* Actual examples of an event?

# Security Considerations

**TODO**: This

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

**TODO**: This
