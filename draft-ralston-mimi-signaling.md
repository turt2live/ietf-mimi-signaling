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
layer, but described in another document [**TODO**: Link to policy I-D].

An overview of the architecture for MIMI is described by [I-D.barnes-mimi-arch [**TODO**: Use a real link]](https://bifurcation.github.io/mimi-arch/#go.draft-barnes-mimi-arch.html).
[**TODO**: Ensure definitions of this doc and arch match]

The signaling layer described by this document deliberately does not concern itself
with the specifics of the encryption/security layer placed next to it. This allows
existing messaging providers to insert their own external-to-MIMI encryption layer
for immediate interoperability while they transition onto MIMI's MLS layer [**TODO**:
Link to E2ES layer].

This document specifies a model where rooms are a virtual place where *users* send
events. Events can be application messages or policy configuration, and are extensible
in nature. The specific policy being used by a room is described during creation,
and enforced within context of that room. Other layers, such as the encryption & security
layer, may apply the policy further. For example, preventing a client from joining an
MLS group if they are not a member of the accompanying room.

Rooms and the events they contain can be persisted in a wide variety of ways, such as
within an MLS group itself through extensions. This document specifies abstract concepts
independent of their persistence.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Terms from [I-D.barnes-mimi-arch [**TODO**: Use a real link]](https://bifurcation.github.io/mimi-arch/#go.draft-barnes-mimi-arch.html)
and {{!I-D.ralston-mimi-terminology}} are used throughout this document.


# Room Model {#int-room-model}

[**TODO**: Check for overlap with arch]

Rooms are a virtual place where events are sent among users. Receiving events is largely
left as a transport responsibility. Events reference the event which was sent previous
to it. This linear singly linked list is known as the "room history". Sequencing of events
into the room history is done by the hub server for the room.

Events have two primary shapes:

* **State events**, containing metadata and policy configuration.
* **Message events**, or non-state events, containing application-specific data. For example,
  encrypted conversation activity.

State events which affect how the room's policy functions are called **auth events**. Message
events can never be an auth event. Each event, regardless of its shape, references the auth
events which permit it to be sent in the room. The **auth chain** for an event is retrieved
by recursing through the auth events of the event. That is, retrieving the auth events for the
event's own auth events, the auth events of those auth events, and so on.

The first event in the room is the "create event". This event defines what specific policy
and encryption/security descriptions are in use by the room, and is itself an auth event for
*all* other events. It MUST NOT have any preceeding events.

Rooms are identified using a **localpart** and the server name which originally created it.
The server name in a room ID is used to enforce a namespace but otherwise serves no meaning.
When another hub takes ownership of a room the room ID does not change. Room IDs are not
intended to be human readable or shown to users.

The ABNF {{!RFC5234}} grammar for a room ID is:

~~~
room_id = "!" room_id_localpart ":" server_name
room_id_localpart = 1*opaque
opaque = DIGIT / ALPHA / "-" / "." / "~" / "_"
~~~

`server_name` is assumed to be compliant with {{Section 2.1 of RFC1123}}.

Room IDs MUST be case sensitive and MUST NOT exceed 255 characters.

Example: `!cEXuEjziVcCzbxbqmN:example.org`


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

[**TODO**: Describe how pseudo IDs work]


# Auth Events

Mentioned in {{int-room-model}}, auth events are the state events which configure the policy
for the room. By name, they are:

* Membership events
* Role events
* The join rules event
* The create event

[**TODO**: The rest of this. Describe auth chain in more detail.]


# History Model

[**TODO**: This. Short version is servers only need the auth chain for all current members of
the room upon joining (we can discard people who joined and already left, and old message events).
Someone needs to keep a copy of each event somewhere though, otherwise the room breaks.]


# [TODO: Sections]

* Event schema/shape. Do not require JSON, through signing might need a canonical format.
  * Including hashes and event ID behaviours
* Join, leave, etc operations overview.
* `[User ->] Follower -> Hub` send path.
* Actual examples of an event?


# Security Considerations

[**TODO**: This]


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

[**TODO**: This]
