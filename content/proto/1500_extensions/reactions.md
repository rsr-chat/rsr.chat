---
title: rsr.chat/reactions
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/reactions

This extension allows clients to attach emoji or custom emote reactions to individual
messages identified by their `rsr.chat/message-link` message ID. Reactions are
aggregated per message per emote by the server and delivered as structured tag
notifications. The full reaction state for a channel is replayable via message history.
Access to reactions may be governed by `rsr.chat/rbac` and `rsr.chat/advanced-moderation`.

## Motivation

Message reactions are a standard affordance in modern messaging platforms that allow
users to respond to a message without producing a new message in the channel log. Without
a standard mechanism, IRC clients cannot interoperate on reaction state. This extension
defines a server-authoritative, ID-anchored reaction store that integrates cleanly with
the existing message identity and emote systems and degrades gracefully for clients that
have not negotiated the capability.

## Capability

This extension is advertised as the `rsr.chat/reactions` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/reactions
```

No capability value is used. Limits are communicated via ISUPPORT (see below).

## Dependencies

### rsr.chat/message-link

This extension requires that `rsr.chat/message-link` be negotiated. Reactions are
addressed by the `rsr.chat/msgid` tag value assigned to each message by that extension.
A reaction cannot be applied to a message that has no resolvable ID. All message ID
semantics, retention, and unresolvability rules defined by `rsr.chat/message-link` apply
equally to reaction targets.

### rsr.chat/emote

This extension requires that `rsr.chat/emote` be negotiated. Emote identifiers used in
reactions MUST be valid under the emote specification. Two emote classes are recognised
by this extension:

- **Unicode emoji**: a single Unicode emoji scalar or fully-qualified emoji sequence as
  defined by the Unicode Emoji standard. Represented as the literal character sequence,
  e.g. `👍` or `❤️`.
- **Custom emote**: a custom emote identifier as defined by `rsr.chat/emote`, expressed
  using that specification's identifier syntax, e.g. `:blobwave:` or
  `:acmecorp/blobwave:`.

Servers MUST reject reaction attempts that reference an emote identifier not valid under
`rsr.chat/emote` with `ERR_REACTIONINVALIDEMOTE`.

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `REACTIONMAX=<n>` — the maximum number of distinct emotes that may be reacted to a
  single message. A value of `0` indicates no server-imposed limit.
- `REACTIONUSERMAX=<n>` — the maximum number of users whose reactions are tracked per
  emote per message. A value of `0` indicates no limit. When this limit is reached the
  server MUST continue to accept reactions but MAY stop tracking individual reactor
  identities for that emote, incrementing only the aggregate count.

Example:

```
    :server 005 nick REACTIONMAX=20 REACTIONUSERMAX=100 :are supported by this server
```

If any of these values change after a configuration reload, a fresh `RPL_ISUPPORT` MUST be
sent to all connected clients.

## Terminology

- **Reaction**: an association between a message ID, an emote, and the account or nick of
  the user who placed it.
- **Aggregate**: the server-maintained count of distinct users who have placed a given
  emote on a given message.
- **Reactor**: a user who has placed a reaction.
- **Toggle semantics**: placing a reaction that the user has already placed removes it;
  placing one they have not placed adds it.

## The REACT Command

```
    REACT <msgid> <emote>
```

Places or removes a reaction on the message identified by `<msgid>`. Reactions follow
toggle semantics: if the calling user has already placed `<emote>` on this message, the
reaction is removed; otherwise it is added.

`<msgid>` MUST be a valid message ID of exactly `MSGIDLEN` characters as defined by
`rsr.chat/message-link`. The server MUST resolve the target message before accepting the
reaction. If the message ID cannot be resolved the server MUST reply with
`ERR_REACTIONUNKNOWNMSG`.

`<emote>` MUST be a valid emote identifier under `rsr.chat/emote`. The server MUST
validate the emote before accepting the reaction. If the emote is not valid the server
MUST reply with `ERR_REACTIONINVALIDEMOTE`.

On success the server MUST broadcast a `REACT` notification to all members of the
channel or conversation in which the target message resides that have negotiated this
capability:

```
:alice!alice@host REACT <msgid> <emote> <action> <count>
```

Where:

- `<action>` is `add` if the reaction was placed or `remove` if it was withdrawn.
- `<count>` is the updated aggregate count of users who have placed `<emote>` on this
  message after applying this change.

Non-negotiating clients receive no notification.

## The REACTLIST Command

```
REACTLIST <msgid>
```

Requests the full reaction state for the message identified by `<msgid>`. The server
responds with zero or more `RPL_REACTENTRY` numerics followed by `RPL_REACTEND`. If IRCv3
`batch` is negotiated the entire response MUST be wrapped in a `rsr.chat/reactlist` batch.

`REACTLIST` may be issued by any client that is currently joined to the channel containing
the target message, or that holds persistent membership in it (see Channel Membership
Integration). Clients that lack visibility of the target channel MUST receive
`ERR_REACTIONUNKNOWNMSG` as if the message did not exist.

## Server Numerics

| Symbolic name                | Description                                                   |
|------------------------------|---------------------------------------------------------------|
| `RPL_REACTENTRY`             | A single emote aggregate entry for a message                  |
| `RPL_REACTEND`               | Marks the end of a REACTLIST response                         |
| `ERR_REACTIONUNKNOWNMSG`     | The target message ID could not be resolved                   |
| `ERR_REACTIONINVALIDEMOTE`   | The emote identifier is not valid under rsr.chat/emote        |
| `ERR_REACTIONFULL`           | The message has reached the REACTIONMAX distinct-emote limit  |
| `ERR_REACTIONNOPERM`         | The client lacks permission to react in this context          |

`RPL_REACTENTRY` has the following format:

```
:server RPL_REACTENTRY <nick> <msgid> <emote> <count> :<reactors>
```

Where:

- `<emote>` is the emote identifier.
- `<count>` is the total number of users who have placed this reaction.
- `<reactors>` is a space-separated list of account names or nicks of up to
  `REACTIONUSERMAX` users who have placed this reaction. If `REACTIONUSERMAX` has been
  exceeded for this emote the list MUST be truncated and the server MUST append the
  token `+more` as the final element to signal that the list is incomplete.

`RPL_REACTEND` has the following format:

```
:server RPL_REACTEND <nick> <msgid> :End of reactions
```

## Reaction State in Message Tags

When a message is delivered to a negotiating client and that message already has one or
more reactions attached (e.g. during history replay), the server MUST include a
`reaction-summary` tag on the message. The tag value is a
pipe-delimited list of `<emote>:<count>` pairs, one per distinct emote that has at
least one reaction, ordered by descending count:

    rsr.chat/reaction-summary=👍:12|❤️:7|:blobwave::3

This allows clients to render reaction counts inline with messages without requiring a
separate REACTLIST round-trip on initial load.

If a message has no reactions, the `reaction-summary` tag MUST be omitted.

## Reaction Storage and Retention

Reactions MUST be stored persistently by the server for as long as the referenced message
is within the channel's history retention window. When a message expires from history,
all reactions attached to it MUST also be discarded.

Reactions MUST NOT be stored for messages that are not within the history retention window
at the time the reaction is placed. If a client attempts to react to a message whose ID
is valid in format but is no longer in the retention window, the server MUST reply with
`ERR_REACTIONUNKNOWNMSG`.

## Message History Integration

When a client requests message history via `CHATHISTORY` or an equivalent mechanism,
relayed messages MUST include the `reaction-summary` tag reflecting the reaction
state at the time of the history request, not the state at original delivery time.
Reaction counts are live aggregates and their value in history replies MUST be current.

Reaction add and remove events themselves MUST NOT appear as entries in the `CHATHISTORY`
message log. They are metadata about messages, not messages in their own right.

## Default Permissions

Unless overridden by `rsr.chat/rbac` or `rsr.chat/advanced-moderation`, the following
default permission rules apply:

- Any client currently joined to a channel MAY use REACT and REACTLIST for messages in
  that channel.
- Persistent members who are not currently active (offline) MAY use REACTLIST but MUST
  NOT use REACT.
- Removing another user's reaction is not permitted under any default permission. Only
  server operators and, when `rsr.chat/rbac` is negotiated, users with the
  `reaction.remove.any` permission MAY do so.
- Server operators MAY use REACT and REACTLIST on any channel.

## Guild and Category Integration

When `rsr.chat/guild` or `rsr.chat/channel-categories` are negotiated, reaction
visibility follows normal channel membership boundaries. There is no guild-wide or
category-scoped reaction feed; reactions are scoped to the individual channel containing
the target message.

When `rsr.chat/channel-categories` is negotiated, RBAC rules for reaction permissions
MAY be scoped to a category or guild prefix using the scope syntax defined by that
specification. See RBAC Integration.

## Channel Metadata Integration

When `rsr.chat/channel-metadata` is negotiated, the following key exists implicitly on
every channel and cannot be deleted:

### `reactions-enabled`

Type: `bool`. When `false`, no reactions may be placed on messages in this channel. REACT
commands targeting this channel MUST be rejected with ERR_REACTIONNOPERM. Default: `true`.

This key may be set by channel operators or by users with the `reaction.configure` RBAC
permission. It is not read-only and may be set via CHANMETA SET.

## Channel Membership Integration

When `rsr.chat/channel-membership` is negotiated, persistent members who are not
currently active (offline) MAY call REACTLIST to inspect reaction state for channels in
which they hold membership. This is consistent with the offline CHATHISTORY access
defined by `rsr.chat/channel-membership`.

Offline persistent members MUST NOT be permitted to place or remove reactions via REACT.
Reactions imply active presence. If an offline member attempts REACT the server MUST
reply with ERR_REACTIONNOPERM.

## RBAC Integration

When `rsr.chat/rbac` is negotiated, the following permission identifiers are defined by
this extension:

| Permission identifier       | Description                                              |
|-----------------------------|----------------------------------------------------------|
| `reaction.add`              | Place a reaction on a message                            |
| `reaction.remove.own`       | Remove one's own reaction from a message                 |
| `reaction.remove.any`       | Remove any user's reaction from a message                |
| `reaction.list`             | Call REACTLIST for messages in this channel              |
| `reaction.configure`        | Set the `reactions-enabled` channel metadata key         |
| `reaction.custom`           | Place reactions using custom emotes (not Unicode emoji)  |

Servers MUST reject REACT from clients that lack `reaction.add` for the target channel
with ERR_REACTIONNOPERM.

Servers MUST reject removal of another user's reaction from a client that lacks
`reaction.remove.any` with ERR_REACTIONNOPERM.

Servers MAY restrict custom emote reactions to clients that hold `reaction.custom`. If a
client without `reaction.custom` attempts to place a custom emote reaction the server
MUST reply with ERR_REACTIONNOPERM.

RBAC rules for reaction permissions MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`.

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, the following predicates are defined
for use in moderation rule conditions (normative syntax defined by that specification):

| Predicate                          | Matches when                                          |
|------------------------------------|-------------------------------------------------------|
| `reaction:has-reactions`           | The target message has at least one reaction          |
| `reaction:count>=<n>`              | The total reaction count across all emotes is >= n    |
| `reaction:emote=<emote>`           | The specified emote has at least one reaction         |
| `reaction:emote-count>=<n>`        | The number of distinct emotes with reactions is >= n  |
| `reaction:is-custom`               | The emote being placed is a custom emote              |
| `reaction:reactor=<account>`       | The specified account has reacted to this message     |

These predicates may be used to enforce policies such as disabling custom emote reactions
in certain channels, alerting on messages that accumulate a large number of reactions, or
restricting reaction-heavy threads.

## Interaction with Non-Negotiating Clients

Non-negotiating clients receive no REACT notifications and cannot issue REACT or
REACTLIST commands. REACT and REACTLIST sent by non-negotiating clients MUST be rejected
with ERR_UNKNOWNCOMMAND. Reaction state is entirely invisible to non-negotiating clients.
Messages delivered to non-negotiating clients MUST NOT include the
`rsr.chat/reaction-summary` tag.

## Examples

### Capability negotiation

```
C: CAP LS 302
S: CAP * LS :rsr.chat/reactions rsr.chat/message-link rsr.chat/emote
C: CAP REQ :rsr.chat/reactions rsr.chat/message-link rsr.chat/emote
S: CAP * ACK :rsr.chat/reactions rsr.chat/message-link rsr.chat/emote
C: NICK alice
C: USER alice 0 * :Alice
S: 005 alice REACTIONMAX=20 REACTIONUSERMAX=100 REACTIONSELF=1 MSGLINKMAX=5 MSGIDLEN=26 :are supported by this server
S: 001 alice :Welcome to the network
```

### Placing a reaction

```
(bob sent a message that alice received as:)
@MSGID=01HQ7K2N4R8VXJD3MCWBPZT56Y :bob!bob@host PRIVMSG #engineering/general :The deploy is fixed.

C: REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍
S: (broadcast to all negotiating channel members:)
:alice!alice@host REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍 add 1
```

### A second user reacts with the same emote

```
:carol!carol@host REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍 add 2
```

### Removing a reaction via toggle

```
    C: REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍
    S: :alice!alice@host REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍 remove 1
```

### Using a custom emote

```
C: REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y :blobwave:
S: :alice!alice@host REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y :blobwave: add 1
```

### REACTLIST with batch

```
C: REACTLIST 01HQ7K2N4R8VXJD3MCWBPZT56Y
S: :server BATCH +rl rsr.chat/reactlist 01HQ7K2N4R8VXJD3MCWBPZT56Y
S: @batch=rl :server RPL_REACTENTRY alice 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍 2 :carol_acct alice_acct
S: @batch=rl :server RPL_REACTENTRY alice 01HQ7K2N4R8VXJD3MCWBPZT56Y :blobwave: 1 :alice_acct
S: :server BATCH -rl
S: :server RPL_REACTEND alice 01HQ7K2N4R8VXJD3MCWBPZT56Y :End of reactions
```

### History replay with reaction summary tag

```
(client requests history for #engineering/general)
S: :server BATCH +hist chathistory #engineering/general
S: @batch=hist;rsr.chat/msgid=01HQ7K2N4R8VXJD3MCWBPZT56Y;rsr.chat/reaction-summary=👍:2|:blobwave::1 :bob!bob@host PRIVMSG #engineering/general :The deploy is fixed.
S: :server BATCH -hist
```

### Unresolvable message ID

```
C: REACT 01HQXXXXXXXXXXXXXXXXXXXXXXXX 👍
S: :server ERR_REACTIONUNKNOWNMSG alice :Message ID could not be resolved
```

### Reactions disabled on channel

```
(reactions-enabled metadata key is false for #announcements)
C: REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍
S: :server ERR_REACTIONNOPERM alice #announcements :Reactions are disabled in this channel
```

### REACTIONMAX exceeded

```
(message already has 20 distinct emotes)
C: REACT 01HQ7K2N4R8VXJD3MCWBPZT56Y 🎉
S: :server ERR_REACTIONFULL alice 01HQ7K2N4R8VXJD3MCWBPZT56Y :Reaction emote limit reached
```

### REACTIONUSERMAX exceeded; truncated reactor list

```
(101 users have reacted with 👍 and REACTIONUSERMAX=100)
C: REACTLIST 01HQ7K2N4R8VXJD3MCWBPZT56Y
S: @batch=rl :server RPL_REACTENTRY alice 01HQ7K2N4R8VXJD3MCWBPZT56Y 👍 101 :user1_acct user2_acct ... user100_acct +more
```

### Self-reaction rejected

```
(REACTIONSELF is absent from ISUPPORT; alice tries to react to her own message)
C: REACT 01HQAB3T2F7PPXC4NK8RWLM09S 👍
S: :server ERR_REACTIONSELF alice :Self-reactions are not permitted on this server
```