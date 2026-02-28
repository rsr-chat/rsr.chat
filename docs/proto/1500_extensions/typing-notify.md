---
title: rsr.chat/typing-notify
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/typing-notify

This extension allows clients to broadcast lightweight typing indicators to other members
of a channel or private conversation. Indicators are purely advisory, are rate-limited by
the server, and are never stored, replayed in history, or delivered to clients that have
not opted in. The design prioritises minimal wire overhead and avoids implementation
patterns that produce excessive or misleading indicators.

## Motivation

Typing indicators are a common affordance in modern messaging applications. Without a
standard mechanism IRC clients either implement proprietary solutions incompatible with
other clients or omit the feature entirely. This extension provides a minimal, interoperable
typing notification mechanism that respects the lightweight nature of IRC and does not
impose meaningful overhead on servers or clients that choose not to use it.

## Capability

This extension is advertised as the `rsr.chat/typing-notify` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/typing-notify
```

No capability value is used. Rate-limit parameters are communicated via ISUPPORT (see
below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `TYPINGINTERVAL=<ms>` — the minimum interval in milliseconds that the server will
  accept between TYPING notifications from a single client for the same target. Clients
  MUST NOT send TYPING for a given target more frequently than this interval. The
  recommended server default is `3000` (3 seconds).
- `TYPINGTIMEOUT=<ms>` — the duration in milliseconds after which, if no further TYPING
  notification is received, receiving clients MUST automatically clear the typing
  indicator for that user. The recommended server default is `6000` (6 seconds). This
  value MUST always be greater than `TYPINGINTERVAL`.

Example:

```
:server 005 nick TYPINGINTERVAL=3000 TYPINGTIMEOUT=6000 :are supported by this server
```

If these values change after a configuration reload, a fresh `RPL_ISUPPORT` MUST be sent
to all connected clients.

## The TYPING Command

```
TYPING <target> <state>
```

`<target>` is a channel name or nick (for private conversations). `<state>` is one of:

| State     | Meaning                                                              |
|-----------|----------------------------------------------------------------------|
| `active`  | The user is currently composing a message                            |
| `done`    | The user has stopped composing without sending (e.g. cleared input)  |

Clients SHOULD send `active` at most once per `TYPINGINTERVAL` while the user is
actively typing. Clients MUST send `done` when the user explicitly clears their input
buffer without sending. Clients MUST NOT send `done` after successfully sending a message;
the act of sending is itself an implicit termination of the typing state and the server
handles clearance automatically (see Implicit Clearance).

Clients MUST NOT send `active` when the input buffer is empty.

## Server Behaviour

### Relay

When the server receives a valid TYPING command it MUST relay it as a `TYPING` message
tag notification to all members of the target channel or conversation that have negotiated
this capability, excluding the sender:

```
:alice!alice@host TYPING #engineering/general active
```

For private conversations the server relays the notification to the target nick only if
that nick has negotiated this capability.

The server MUST NOT relay TYPING notifications to clients that have not negotiated this
capability under any circumstance.

### Rate limiting

The server MUST enforce the `TYPINGINTERVAL` limit per sending client per target. If a
client sends TYPING for a given target within the interval since their last accepted
TYPING for that target, the server MUST silently discard the excess notification without
relaying it and without sending an error to the client.

Silent discard is required rather than an error numeric to avoid generating additional
round-trip traffic that would itself defeat the purpose of rate limiting.

### Implicit clearance

When a client in a typing state sends a PRIVMSG or NOTICE to a target, the server MUST
synthesise and relay a `done` TYPING notification to the relevant recipients on behalf of
the sender:

```
:alice!alice@host TYPING #engineering/general done
```

This ensures that indicators are cleared promptly on message delivery without requiring
clients to send an explicit `done` after every sent message. If the client also sends an
explicit `done` immediately after a message, the server MUST rate-limit the explicit
notification as normal; the synthesised clearance takes precedence.

### Session end

When a client disconnects or quits, the server MUST synthesise and relay a `done` TYPING
notification for every target for which that client currently has an active typing state,
before processing the QUIT:

```
:alice!alice@host TYPING #engineering/general done
```

Servers MUST track which targets a client has a live `active` state for in order to
perform this cleanup. The per-target state is established on receipt of an `active`
notification and cleared on receipt of a `done`, on implicit clearance, or on session end.

### History and storage

TYPING notifications MUST NOT be stored in message history. TYPING notifications MUST
NOT appear in CHATHISTORY replies. Servers MUST ensure that TYPING state is entirely
transient and leaves no persistent record.

## Client Behaviour

### Sending

Clients SHOULD implement a timer-based send strategy:

1. When the user begins typing, send `active` immediately.
2. Restart a local timer of `TYPINGINTERVAL` milliseconds.
3. If the user is still typing when the timer fires, send another `active` and restart
   the timer.
4. If the user stops typing (input idle, not cleared), allow the timer to expire without
   sending; the server-side `TYPINGTIMEOUT` will cause receiving clients to clear the
   indicator automatically.
5. If the user explicitly clears their input buffer, send `done` immediately and cancel
   the timer.

This strategy means that under continuous typing a client sends at most one notification
per `TYPINGINTERVAL`, which at the default of 3 seconds is at most 20 messages per minute
per target — a negligible overhead.

Clients MUST NOT send TYPING for a target the client is not currently joined to or
conversing with.

### Receiving

Receiving clients MUST maintain a per-sender per-target timer initialised to
`TYPINGTIMEOUT` milliseconds on receipt of an `active` notification. If the timer expires
before a subsequent `active` or a `done` is received, the client MUST clear the indicator
as if `done` had been received. This ensures that indicators do not persist indefinitely
if a `done` is lost or a client disconnects without the server synthesising a clearance.

On receipt of `done` for a sender, the client MUST immediately clear the indicator and
cancel the timeout timer for that sender-target pair.

Clients SHOULD render indicators in a non-intrusive fashion — for example a small
animated ellipsis in a status area below the message list — rather than inserting
transient messages into the message log.

Clients SHOULD aggregate multiple simultaneous typers gracefully, e.g. "alice and bob
are typing..." or "several people are typing..." above a configurable threshold, to avoid
an indicator area that becomes noisy in active channels.

Clients MUST provide a user-facing option to disable sending typing notifications.
Clients SHOULD provide a separate user-facing option to disable receiving and displaying
them.

## Privacy Considerations

Typing notifications reveal behavioural metadata: that a user is present, active, and
composing a message. This is intentional but has privacy implications in some contexts.

Clients that allow users to disable sending notifications MUST do so cleanly — they MUST
NOT send `active` when the feature is disabled, and MUST send `done` if they were
previously in an `active` state at the moment the user disables the feature.

Servers MAY allow users to suppress outbound relay of their own TYPING notifications
globally via a user mode or preference mechanism outside the scope of this specification.
If a server suppresses a user's TYPING notifications it MUST do so silently.

## Interaction with Non-Negotiating Clients

Non-negotiating clients receive no TYPING messages. They are entirely unaware of the
mechanism. TYPING commands sent by non-negotiating clients MUST be rejected with
`ERR_UNKNOWNCOMMAND`. This is not an error condition; it simply means typing indicators
are invisible to those clients, which is the correct degraded behaviour.

## Channel Membership Integration

When `rsr.chat/channel-membership` is negotiated, persistent members who are not currently
active (offline members) MUST NOT receive or generate typing notifications. TYPING relay
targets only currently active joined members, consistent with the transient nature of
typing state.

## Guild and Category Integration

When `rsr.chat/guild` or `rsr.chat/channel-categories` are negotiated, TYPING relay
follows normal channel membership boundaries. There is no guild-wide or category-scoped
typing notification; notifications are scoped to individual channels as normal.

Guild or category operators do not receive any special typing notification visibility
beyond that of a normal channel member.

## RBAC Integration

When `rsr.chat/rbac` is negotiated, the following permission identifier is defined by
this extension:

| Permission identifier  | Description                                               |
|------------------------|-----------------------------------------------------------|
| `typing.send`          | Client may send TYPING notifications to this target       |
| `typing.receive`       | Client receives TYPING notifications from this target     |

Servers MUST NOT relay TYPING notifications from clients that lack `typing.send` for the
relevant target. Such notifications are silently discarded.

RBAC rules for these permissions MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`.

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, the following predicate is defined
for use in moderation rule conditions (normative syntax defined by that specification):

| Predicate            | Matches when                                                 |
|----------------------|--------------------------------------------------------------|
| `typing:active`      | The user currently has an active typing state in this channel|

This predicate may be used in unusual moderation configurations, such as preventing
typing notifications in announcement-only channels regardless of capability negotiation.

## Examples

### Capability negotiation

```
C: CAP LS 302
S: CAP * LS :rsr.chat/typing-notify
C: CAP REQ :rsr.chat/typing-notify
S: CAP * ACK :rsr.chat/typing-notify
C: NICK alice
C: USER alice 0 * :Alice
S: 005 alice TYPINGINTERVAL=3000 TYPINGTIMEOUT=6000 :are supported by this server
S: 001 alice :Welcome to the network
```

### Typing indicator sent and received

```
(alice begins typing in #engineering/general)
C: TYPING #engineering/general active
S: (relayed to all other active members that negotiated the capability:)
:alice!alice@host TYPING #engineering/general active

(3 seconds later, alice is still typing)
C: TYPING #engineering/general active
:alice!alice@host TYPING #engineering/general active

(alice sends her message; server synthesises done)
C: PRIVMSG #engineering/general :Here is my thoughts on the deploy issue.
S: (relayed to channel:)
:alice!alice@host PRIVMSG #engineering/general :Here is my thoughts on the deploy issue.
:alice!alice@host TYPING #engineering/general done
```

### Explicit clearance

```
(alice starts typing then deletes her input)
C: TYPING #engineering/general active
C: TYPING #engineering/general done
S: :alice!alice@host TYPING #engineering/general done
    (relayed immediately; receiving clients clear the indicator)
```

### Rate limit enforcement

```
(client sends active notifications too quickly)
C: TYPING #engineering/general active     (accepted, relayed)
C: TYPING #engineering/general active     (sent 500ms later; discarded silently)
C: TYPING #engineering/general active     (sent 1000ms later; discarded silently)
C: TYPING #engineering/general active     (sent 3000ms after first; accepted, relayed)
```

### Session end clearance

```
(alice disconnects while in active typing state in two channels)
S: :alice!alice@host TYPING #engineering/general done
    (synthesised and relayed to #engineering/general members)
S: :alice!alice@host TYPING #random done
    (synthesised and relayed to #random members)
S: :alice!alice@host QUIT :Connection closed
```

### Private conversation typing indicator

```
(alice is composing a private message to bob, who has negotiated the capability)
C: TYPING bob active
S: (relayed to bob only:)
:alice!alice@host TYPING alice active

(alice sends the message)
C: PRIVMSG bob :Hey, are you free to review my PR?
S: :alice!alice@host TYPING alice done
    (synthesised clearance relayed to bob)
```

### Non-negotiating client

```
(carol has not negotiated rsr.chat/typing-notify)
C: TYPING #engineering/general active
S: :server 421 carol TYPING :Unknown command
```
