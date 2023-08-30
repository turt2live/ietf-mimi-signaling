---
title: "MIMI Signaling"
abbrev: "MIMI Signaling"
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
 - next generation
 - unicorn
 - sparkling distributed ledger
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

The MIMI signaling layer is responsible for user-level interactions in the overall
messaging stack. It is aware of encryption, and more specifically MLS, but does not
have a dependency on any particular encryption or security layer.

--- middle

# Introduction

The More Instant Messaging Interoperability (MIMI) working group is responsible for
specifying a protocol to achieve interoperability among modern messaging providers.
Most providers do not currently support Messaging Layer Security (MLS) {{?RFC9420}},
a chartered requirement of MIMI, but do support other forms of encryption alongside
their existing signaling.

Signaling in the context of MIMI is the layer responsible for user-level operation of
a chat, such as joining, parting, banning, etc. These user operations are validated
through use of a defined policy envelope. The policy is enforced by the signaling
layer, but described in another document **TODO**: Link to policy I-D.

An overview of the architecture for MIMI is described by [I-D.barnes-mimi-arch TODO: Use a real link](https://bifurcation.github.io/mimi-arch/#go.draft-barnes-mimi-arch.html).
**TODO**: Ensure definitions of this doc and arch match

The signaling layer described by this document deliberately does not concern itself
with the specifics of the encryption/security layer placed next to it. This allows
existing messaging providers to insert their own external-to-MIMI encryption layer
for immediate interoperability while they transition onto MIMI's MLS layer **TODO**:
Link to E2ES layer.

This document specifies a model where rooms are a virtual place where *users* send
events. Events can be application messages or policy configuration, and are extensible
in nature. The specific policy being used by a room is described during creation,
and enforced within context of that room. Other layers, such as the encryption & security
layer, may apply the policy further. For example, preventing a client from joining an
MLS group if they are not a member of the accompanying room.

Rooms and the events they contain can be persisted in a wide variety of ways, such as
within an MLS group itself through extensions. This document specifies abstract concepts
independent of their persistence.

Finally, this signaling layer is agnostic of transport. Its events can be re-framed for
sending between servers without affecting signaling itself.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms from [I-D.barnes-mimi-arch TODO: Use a real link](https://bifurcation.github.io/mimi-arch/#go.draft-barnes-mimi-arch.html)
and {{!I-D.ralston-mimi-terminology}} are used throughout this document.

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
