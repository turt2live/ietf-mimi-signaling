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

informative:
  RFC1123:


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

# Room Model {#int-room-model}

**TODO**: Check for overlap with arch

A room is a conceptual place where events are sent among users. These events carry information
such as the new policy configuration, application messages, and other room state information
as needed. Receiving events is left as a responsibility for the transport.

Rooms have two primary event definitions:

* **State events**, containing metadata and policy configuration.
* **Message events**, containing application-specific data. For example, encrypted conversation
  activity.

Events reference the event preceeding themselves linearly. The hub server is expected to enforce
this linear relationship, and therefore events have two representations. Events not yet processed
by a hub server are known as "limited events", and are lacking authorization and ordering information.
They are not sent until they become "fully-formed events" by a hub server, where the missing
auth and ordering fields are added. A hub server MUST only emit fully-formed events over its
transport, though it does *receive* limited events from other follower servers.

Throughout this document, the word "event" means "fully-formed event" unless otherwise qualified
as a "limited event". This implies that the event has been accepted by the hub into the room.

Each room has exactly one set of policies it follows. Once the policy is set, it cannot have
rules added or removed. The policy can only be configured using state events. The ability to send
any particular event is determined by policy. If a room wishes to use a new set of policies, it
must replace itself using a new room identifier.

The state events which affect how the policy operates are known as "auth events". Message events
can never be an auth event. Both message and state events reference the auth events which permit
it to be sent to the room, authorizing its existence in that room. The "auth chain" for an event
is the recursive collection of auth events. That is, the auth events for the event and the auth
events of those events and so on.

The first event in the room is the "create event". This event defines the specific set of policies
the room is using. The create event MUST NOT have any preceeding events, and MUST be an auth event
for all future events. The create event additionally specifies the specific encryption algorithm
and configuration for the room, enabling the room's choice of encryption & security layers. For
MIMI, this is expected to be the algorithm specified by **TODO**: Link to E2ES I-D.

The create event is described in more detail in {{int-ev-create}}.

Rooms are identified under the creating server's domain, creating a namespace. Room IDs are not
intended to be parsed or human readable. The unique portion of the room ID, the localpart, is
deliberately an implementation detail. Servers can use any pattern they wish for localparts, so
long as the resulting room ID hasn't been used before.

The server name in the room ID is only used to validate the create event and declare the initial
hub server in the room. Otherwise, the server name serves no purpose. The server might not even
be participating in the room anymore.

The ABNF {{!RFC5234}} grammar for a room ID is:

~~~
room_id = "!" room_id_localpart ":" server_name
room_id_localpart = 1*opaque
opaque = DIGIT / ALPHA / "-" / "." / "~" / "_"
~~~

`server_name` is as described by [I-D.barnes-mimi-arch TODO: Use a real link](https://bifurcation.github.io/mimi-arch/#go.draft-barnes-mimi-arch.html).

Room IDs MUST be case sensitive and MUST NOT exceed 255 characters.

Example: `!cEXuEjziVcCzbxbqmN:example.org`

## Room History

Consider the following series of events:

~~~aasvg
A <-- B <-- C <-- D
~~~
{: #overview title="A four event room." }

The first event, A, is the create event ({{int-ev-create}}) for the room. B is the creator's
join event ({{int-ev-user}}), C could be an encrypted "hello world" text message, and D an
invite ({{int-ev-user}}) for another user. Collectively, this linear set of events creates
the history for the room.

**TODO**: Does history visibility go here?

## User and Server Participation

**TODO**: This?

# User Model

Users belong to a server, and are represented by a user ID. Each user ID has a **localpart**
and references the server which is responsible for that user.

The ABNF {{!RFC5234}} grammar for a user ID is:

~~~
user_id = "@" user_id_localpart ":" server_name
user_id_localpart = 1*user_id_char
user_id_char = DIGIT
             / %x61-7A                   ; a-z
             / "-" / "." / "="
             / "_" / "/" / "+"
~~~

Like room IDs, `server_name` is assumed to be compliant with {{Section 2.1 of RFC1123}}.

Room IDs MUST be case sensitive and MUST NOT exceed 255 characters.

Example: `@watch/for/slashes:example.org`

**TODO**: Describe how pseudo IDs work

# Auth Events

Mentioned in {{int-room-model}}, auth events are the state events which configure the policy
for the room. By name, they are:

* Membership events
* Role events
* The join rules event
* The create event

**TODO**: The rest of this. Describe auth chain in more detail.

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
