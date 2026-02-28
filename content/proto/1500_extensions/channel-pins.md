---
title: rsr.chat/channel-pins
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/channel-pins

This extension allows messages to be pinned within the channel they were originally sent
in. Pinned messages are surfaced via a structured list and represented as a reserved
channel metadata key. Each pin is identified by the message ID assigned by
`rsr.chat/message-link`. A message may only be pinned to the channel it was originally
sent in; cross-channel pinning is explicitly prohibited.

## Motivation

Pinned messages are a widely used affordance for surfacing important, persistent
information in a channel — rules, announcements, links, or decisions without relying
on the topic field or repeating the content manually. This extension provides a
server-authoritative pin list that is portable, auditable, and integrated with the
existing message identity and metadata infrastructure of this suite.

## Capability

This extension is advertised as the `rsr.chat/channel-pins` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/channel-pins
```

No capability value is used. Limits are communicated via ISUPPORT (see below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in RPL_ISUPPORT (005) when this capability
is active:

- `PINMAX=<n>` — the maximum number of pinned messages permitted per channel. A value
  of `0` indicates no server-imposed limit. The recommended default is `50`.

Example:

```
:server 005 nick PINMAX=50 :are supported by this server
```

If this value changes after a configuration reload, a fresh RPL_ISUPPORT MUST be sent
to all connected clients.

## Dependencies

This extension has two hard dependencies:

- `rsr.chat/message-link` — required to identify messages by their server-assigned
  `rsr.chat/msgid` tag. Servers MUST NOT advertise `rsr.chat/channel-pins` if
  `rsr.chat/message-link` is not active.
- `rsr.chat/channel-metadata` — required to store the pin list as a reserved metadata
  key and to surface pin count. Servers MUST NOT advertise `rsr.chat/channel-pins` if
  `rsr.chat/channel-metadata` is not active.

## Terminology

- **Pin**: an association between a channel and a message ID, asserting that the
  referenced message was sent in that channel and has been designated as notable.
- **Pin entry**: a record consisting of a message ID, the channel it was pinned in, the
  account name of the pinner, and the timestamp at which the pin was created.
- **Pin list**: the ordered sequence of pin entries for a channel, ordered from most
  recently pinned to least recently pinned.

## Channel Identity Constraint

A message MUST only be pinned to the channel in which it was originally sent. The server
MUST verify this by consulting its message store. If the message ID does not correspond
to a message in the target channel, the server MUST reject the pin attempt as if the message
did not exist.

## Reserved Metadata Keys

When both `rsr.chat/channel-metadata` and `rsr.chat/channel-pins` are active, the
following keys exist implicitly on every channel and MUST NOT be deleted.

### `pin-count`

Type: `uint`. The current number of pinned messages in this channel. Updated by the
server on each PIN ADD or PIN REMOVE. Read-only; attempts to set via CHANMETA SET MUST
result in `ERR_CHANMETAREADONLY`.

### `pin-messages`

Type: `TEXT`. A JSON-encoded list of message IDs that are pinned, in pin order.
Read-only; attempts to set via CHANMETA SET MUST result in `ERR_CHANMETAREADONLY`.

Both keys appear in CHANMETA LIST responses as normal metadata entries.

## Commands

### PIN ADD

```
PIN <channel> ADD <msgid>
```

Pins the message identified by `<msgid>` in `<channel>`. The server MUST verify:

1. The message ID is known to the server's message store.
2. The message was actually sent in `<channel>`.
3. The message is not already pinned in this channel.
4. The channel has not reached the `PINMAX` limit.

If all conditions are satisfied, the server adds the pin entry to the top of the pin list
and broadcasts a `PIN ADD` notification to all active channel members that have
negotiated this capability:

```
:pinner!user@host PIN #channel ADD <msgid> <pin-timestamp>
```

Where `<pin-timestamp>` is an ISO 8601 UTC timestamp indicating when the pin was created.

The server MUST also update the `pin-count` and `pin-messages` reserved metadata keys
atomically with the pin creation, and MUST broadcast the corresponding CHANMETA SET
notifications to members that have negotiated `rsr.chat/channel-metadata`.

### PIN REMOVE

```
PIN <channel> REMOVE <msgid>
```

Removes the pin for the message identified by `<msgid>` from `<channel>`. If the message
is not currently pinned the server MUST reply with ERR_PINNOTFOUND.

On success the server MUST broadcast a `PIN REMOVE` notification to all active channel
members that have negotiated this capability:

```
:remover!user@host PIN #channel REMOVE <msgid>
```

The server MUST update `pin-count` and `pin-messages` atomically.

### PIN LIST

```
PIN <channel> LIST
```

Requests the full pin list for the channel. If IRCv3 `batch` is negotiated the response
MUST be wrapped in a `rsr.chat/pinlist` batch. The server responds with zero or more
RPL_PINENTRY numerics followed by RPL_PINEND.

### PIN GET

```
PIN <channel> GET <msgid>
```

Requests the pin entry for a single message ID. The server replies with a single
RPL_PINENTRY followed by `RPL_PINEND`, or `ERR_PINNOTFOUND` if the message is not pinned.

## Server Numerics

| Symbolic name          | Description                                                      |
|------------------------|------------------------------------------------------------------|
| `RPL_PINENTRY`         | A single pin list entry                                          |
| `RPL_PINEND`           | Marks the end of a PIN LIST or PIN GET response                  |
| `ERR_PINNOTFOUND`      | The specified message ID is not pinned in this channel           |
| `ERR_PINALREADYPINNED` | The specified message ID is already pinned in this channel       |
| `ERR_PINWRONGCHANNEL`  | The message was not sent in the specified channel                |
| `ERR_PINUNKNOWNMSG`    | The message ID is not known to the server's message store        |
| `ERR_PINFULL`          | The channel has reached the PINMAX limit                         |
| `ERR_PINNOPERM`        | The client lacks permission to perform this pin operation        |

RPL_PINENTRY has the following format:

```
:server RPL_PINENTRY <nick> <channel> <msgid> <pinner-account> <pin-timestamp> * :<body-preview>
:server RPL_PINENTRY * :<body-preview-cont>
:server RPL_PINENTRY   :<original-msg-tags-if-applicable>
```

Where:

- `msgid` is the pinned message ID.
- `pinner-account` is the account name of the user who created the pin.
- `pin-timestamp` is an ISO 8601 UTC timestamp of when the pin was created.
- `body` is up to  bytes of the pinned message body, UTF-8, newlines replaced
  with U+2424, truncated with a trailing `…` if necessary. This preview is generated at
  pin creation time and stored with the pin entry; it is NOT re-fetched from message
  history on each LIST. If the referenced message has since expired from the message
  store, the preview MUST still be returned as stored at pin time.

`RPL_PINEND` has the following format:

```
:server `RPL_PINEND` <nick> <channel> :End of pin list
```

## Pin Entry Persistence

Pin entries are persistent records and MUST survive server restarts if the server
supports persistent storage. The body preview embedded in each `RPL_PINENTRY` is captured
at pin creation time precisely because the referenced message may eventually expire from
the message history store. The pin entry itself is not subject to message retention
limits.

If a pinned message's ID becomes unresolvable in the message store (due to expiry), the
pin entry remains. The body preview is the authoritative display text. Clients SHOULD
indicate to the user when a pinned message's full history context is no longer available,
which they can determine by attempting `CHATHISTORY` around the pin timestamp and finding
no matching message.

## Message Expiry and Pin Integrity

Servers MUST NOT automatically remove pin entries when the underlying message expires
from the history store. Pins outlive their source messages. If an operator or server
process forcibly deletes a message (e.g. via a moderation action), the server SHOULD also
remove the corresponding pin entry and broadcast a PIN REMOVE notification as if an
operator had issued the command.

## Default Permissions

Unless overridden by `rsr.chat/rbac`, the following default permission rules apply:

- Any channel member MAY call PIN LIST and PIN GET.
- Channel operators (mode `+o`) and above MAY call PIN ADD and PIN REMOVE.
- Setting and removing pins follows the same operator threshold as other channel
  management operations.
- Server operators MAY call PIN ADD and PIN REMOVE on any channel.

A pinner MAY remove their own pin regardless of their current role, provided they were
the original pinner. Servers MUST track the pinner account per entry to support this.

## Guild Integration

When `rsr.chat/guild` is negotiated, guild operators MAY add and remove pins in any
channel within the guild without holding a channel-level operator role. Guild-level pin
operations are broadcast and recorded identically to channel-level operations.

Guild operators MAY NOT pin messages from one guild channel into another; the channel
identity constraint (see Channel Identity Constraint) applies without exception regardless
of guild role.

## RBAC Integration

When `rsr.chat/rbac` is negotiated, the following permission identifiers are defined by
this extension:

| Permission identifier   | Description                                             |
|-------------------------|---------------------------------------------------------|
| `pins.list`             | View the pin list and individual pin entries            |
| `pins.add`              | Add a new pin to the channel                            |
| `pins.remove`           | Remove any existing pin from the channel                |
| `pins.remove.own`       | Remove a pin the requesting user originally created     |

RBAC rules for these permissions MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`.

Example (illustrative; normative RBAC syntax defined in rsr.chat/rbac):

```
RBACSET #engineering/ ROLE op PERMISSION pins.add
RBACSET #engineering/ ROLE op PERMISSION pins.remove
RBACSET #acmecorp/ ROLE admin PERMISSION pins.add
```

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, the following predicates are defined
for use in moderation rule conditions (normative syntax defined by that specification):

| Predicate                  | Matches when                                          |
|----------------------------|-------------------------------------------------------|
| `pins:is-pinned`           | The referenced message is currently pinned            |
| `pins:pin-count>=<n>`      | The channel has at least n pinned messages            |
| `pins:pin-count<=<n>`      | The channel has at most n pinned messages             |
| `pins:pinner=<account>`    | The pin was created by the specified account          |

These predicates may be used to gate actions on whether a channel is heavily pinned,
to restrict moderation operations on pinned messages, or to trigger alerts when the pin
list reaches a threshold.

## Interaction with Non-Negotiating Clients

Clients that have not negotiated `rsr.chat/channel-pins` receive no PIN notifications.
PIN commands sent by non-negotiating clients MUST be rejected with `ERR_UNKNOWNCOMMAND`.

Non-negotiating clients may observe pin activity indirectly via the `pin-count` and
`pin-latest-msgid` reserved metadata keys if they have negotiated
`rsr.chat/channel-metadata`, but receive no structural pin data.
