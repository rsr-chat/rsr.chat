---
title: rsr.chat/modern-ping
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/modern-ping

This extension provides a structured, server-validated inline mention syntax for
addressing other users within a channel. Unlike the informal convention of typing a
nick verbatim, this extension allows clients to embed typed mention tokens that the
server validates against channel membership, annotates with structured tags, and
delivers to the mentioned user with a dedicated notification tag. It integrates with
the RBAC and Guild extensions for scoped permission control.

## Motivation

IRC clients have long highlighted messages containing a user's nick by simple string
matching. This approach is fragile: it fires on partial matches, cannot distinguish
intentional pings from incidental occurrences of a string, and carries no semantic
structure that servers or clients can act on programmatically. This extension
formalizes the concept of an intentional, validated mention (a ping) as a first-class
protocol feature, while degrading cleanly for clients that have not negotiated the
capability.

## Capability

This extension is advertised as the `rsr.chat/modern-ping` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/modern-ping
```

No capability value is used. Supplementary configuration is communicated via ISUPPORT
(see below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `MAXPINGS=<n>` — the maximum number of ping tokens permitted in a single message.
  Servers MUST NOT enforce a value lower than 1.

Example:

```
:server 005 nick MAXPINGS=5 :are supported by this server
```

If this value changes after a configuration reload, a fresh `RPL_ISUPPORT` MUST be sent
to all connected clients.

## Terminology

- **Ping**: an intentional, validated reference to another user embedded in a message body.
- **Ping token**: the inline syntax used to express a ping (see Inline Ping Syntax).
- **Target**: the user being pinged.
- **Mention tag**: the `rsr.chat/mentioned` message tag attached to a delivered message
  to notify the target that they were pinged (see Notification Tag).
- **Special ping**: a ping that addresses a group rather than a single user
  (see Special Pings).

## Inline Ping Syntax

A ping token may appear anywhere in a `PRIVMSG` or `NOTICE` body. The syntax is:

```
@<nick>
```

Where `<nick>` is the IRC nick of the intended target. The `@` sigil introduces the
token. The token ends at the first character that is not a valid nick character as
defined by the server's NICKLEN and nick character rules, or at end of string. The
sigil and nick together form the ping token and are transmitted verbatim in the message
body.

Multiple ping tokens MAY appear in a single message up to the `MAXPINGS` limit.

Ping tokens in a NOTICE MUST NOT be validated, resolved, or annotated by the server.

### Example body with a ping

```
@alice can you take a look at this?
```

## Outbound Tags

For each resolved ping in a relayed PRIVMSG, the server MUST attach a
`ping.<n>` message tag to the outbound message, where `<n>` is the zero-based
index of the ping token in the order it appears in the body. The tag value is the
resolved nick of the target at the time of delivery, which MAY differ from the nick in
the body if the server normalises nick casing:

```
ping.0=alice
```

These tags are attached to the message delivered to all channel members that have
negotiated this capability, not only to the target. Clients SHOULD use these tags to
apply consistent mention highlighting regardless of their local nick-matching logic.

Clients that have not negotiated this capability receive messages with `@nick` tokens
as plain text and receive no ping tags. Servers MUST NOT strip the `@nick` tokens from
the body before relay, as they serve as the legible degraded form for non-negotiating
clients.

## Server Numerics

| Symbolic name        | Description                                                         |
|----------------------|---------------------------------------------------------------------|
| `ERR_PINGNOSUCHNICK` | The pinged nick is not present in the channel                       |
| `ERR_PINGNOPERM`     | The sender lacks permission to ping this user or group              |
| `ERR_TOOMANYPING`    | The message contains more ping tokens than MAXPINGS permits         |

`ERR_PINGNOSUCHNICK` format:

```
:server ERR_PINGNOSUCHNICK <sender> <channel> <nick> :No such nick in channel
```

`ERR_PINGNOPERM` format:

```
:server ERR_PINGNOPERM <sender> <channel> <nick> :Permission denied
```

`ERR_TOOMANYPING` format:

```
:server `ERR_TOOMANYPING` <sender> <channel> :Too many ping tokens in message
```

## Special Pings

In addition to individual nick pings, this extension defines the following reserved
special ping tokens. Special pings address groups of users and are subject to
stricter permission requirements (see Default Permissions).

| Token       | Addresses                                                               |
|-------------|-------------------------------------------------------------------------|
| `@everyone` | All users currently joined to the channel                               |
| `@here`     | All users joined to the channel whose away status is not set            |

Special ping tokens follow the same inline syntax as nick pings and are subject to the
same `MAXPINGS` limit. Each special ping token counts as one ping regardless of how
many users it resolves to.

The `ping.<n>` outbound tag value for a special ping is the token name itself 
rather than a nick:

```
ping.0=@here
```

Additional special tokens MAY be defined by future extensions. Servers MUST reject
unrecognised `@` tokens that do not match any joined member's nick with
`ERR_PINGNOSUCHNICK`.

## Default Permissions

Unless overridden by `rsr.chat/rbac` or `rsr.chat/advanced-moderation`, the following
rules apply:

- Any channel member MAY ping any other individual member of the same channel.
- Only channel operators (mode `+o`) MAY use `@here` or `@online`, UNLESS
- The channel has mode `+P` indicating anyone may send group pings.
- Server operators MAY use any ping token in any channel.
- No user MAY ping a user in a channel they are not joined to.

## Guild Integration

When `rsr.chat/guild` is negotiated, the following additional behaviour applies:

- Guild operators MAY use `@here` and `@online` in any channel within their guild
  regardless of their channel operator status.
- A guild-level special ping token `@guild` is defined, addressing all members of the
  guild across all channels they are joined to within that guild. `@guild` may only be
  used by guild owners or users granted the `ping.guild` RBAC permission (see RBAC
  Integration).
- Cross-guild pings are not permitted. A ping token naming a user who is not joined to
  the target channel is rejected with `ERR_PINGNOSUCHNICK` regardless of guild membership.

Servers MUST advertise support for the `@guild` token via the `MSGLINKGUILD=1` ISUPPORT
token when both `rsr.chat/modern-ping` and `rsr.chat/guild` are active.

## Category Integration

When `rsr.chat/channel-categories` is negotiated, RBAC ping permissions MAY be scoped
to a category prefix using the scope identifier syntax defined by that specification.
A category-scoped permission applies to all leaf channels within that category. No
category-level special ping token is defined by this extension.

## RBAC Integration

When `rsr.chat/rbac` is negotiated, the following permission identifiers are defined by
this extension:

| Permission identifier    | Description                                                  |
|--------------------------|--------------------------------------------------------------|
| `ping.user`              | May ping individual users by nick                            |
| `ping.here`              | May use the @here special ping                               |
| `ping.online`            | May use the @online special ping                             |
| `ping.guild`             | May use the @guild special ping (requires rsr.chat/guild)    |

RBAC rules for these permissions MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`.

Example (illustrative; normative RBAC syntax defined in rsr.chat/rbac):

```
RBACSET #engineering/ ROLE moderator PERMISSION ping.here
RBACSET #acmecorp/ ROLE owner PERMISSION ping.guild
RBACSET #engineering/general USER quietuser PERMISSION ping.exempt
```

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, the following predicates are defined
for use in moderation rule conditions (normative syntax defined by that specification):

| Predicate                     | Matches when                                           |
|-------------------------------|--------------------------------------------------------|
| `ping:has-ping`               | The message contains at least one ping token           |
| `ping:count>=<n>`             | The message contains at least n ping tokens            |
| `ping:has-special`            | The message contains at least one special ping token   |
| `ping:targets <nick>`         | The message pings the named nick                       |

These predicates may be used to rate-limit mass pings, suppress pings from users below
a trust threshold, or flag messages that ping operators.

## Participation by Non-Negotiating Clients

Clients that have not negotiated `rsr.chat/modern-ping` receive messages containing
`@nick` tokens as plain text. They will not receive `ping.<n>` tags. Their clients
may or may not highlight the message based on their own local nick-matching logic, 
which is outside the scope of this specification.

Non-negotiating clients MAY still be the *target* of a ping sent by a negotiating
client. The server MUST attach `ping.<n>` to the copy delivered to the non-negotiating
target. Clients that do not understand this tag will ignore it, but the tag is available
for future client upgrades.

Non-negotiating clients that send a message containing `@nick` patterns are not subject
to ping validation or MAXPINGS enforcement, as the server has no basis to interpret
their message bodies as containing intentional pings. Such messages are relayed
normally, without additional ping tags applied.

## Examples

### Capability negotiation

```
C: CAP LS 302
S: CAP * LS :rsr.chat/modern-ping
C: CAP REQ :rsr.chat/modern-ping
S: CAP * ACK :rsr.chat/modern-ping
C: NICK alice
C: USER alice 0 * :Alice
S: 005 alice MAXPINGS=5 :are supported by this server
S: 001 alice :Welcome to the network
```

### Successful individual ping

(bob is joined to #engineering/general)

```
C: PRIVMSG #engineering/general :@bob can you review this?

(All negotiating channel members receive:)
@ping.0=bob :alice!alice@host PRIVMSG #engineering/general :@bob can you review this?

(Non-negotiating members receive:)
:alice!alice@host PRIVMSG #engineering/general :@bob can you review this?
```

### Ping rejected — target not in channel

(charlie is not joined to #engineering/general)

```
C: PRIVMSG #engineering/general :@charlie are you around?
S: :server ERR_PINGNOSUCHNICK alice #engineering/general charlie :No such nick in channel
```

### Ping rejected — permission denied for special ping

(alice is not a channel operator)

```
C: PRIVMSG #engineering/general :@here please review the PR before EOD
S: :server ERR_PINGNOPERM alice #engineering/general @here :Permission denied
```

### Successful special ping by an operator

(alice is a channel operator)

```
C: PRIVMSG #engineering/general :@here standup in 5 minutes

(Every negotiating member receives:)
@ping.0=@here :alice!alice@host PRIVMSG #engineering/general :@here standup in 5 minutes
```

### Exceeding MAXPINGS

(Server has MAXPINGS=2)

```
C: PRIVMSG #engineering/general :@alice @bob @charlie thoughts?
S: :server ERR_TOOMANYPING alice #engineering/general :Too many ping tokens in message
```

### Multiple pings in one message

```
C: PRIVMSG #engineering/general :@bob @carol the build is failing, can one of you take a look?

(negotiating members receive:)
@ping.0=bob;ping.1=carol :alice!alice@host PRIVMSG #engineering/general :@bob @carol the build is failing, can one of you take a look?
```
