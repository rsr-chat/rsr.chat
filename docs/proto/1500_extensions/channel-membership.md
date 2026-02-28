---
title: rsr.chat/channel-membership
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/channel-membership

This extension separates a channel's persistent membership list from its transient list
of currently connected and joined users. A user may be a recognised member of a channel
without being present in an active IRC session, and may rejoin a channel to which they
hold membership without requiring an invitation or operator action. Membership entries
carry roles, metadata, and join timestamps that survive disconnection. The extension
integrates with IRCv3 account-based identity, channel history, and the access control
and organisational extensions in this suite.

## Motivation

Standard IRC channel membership is entirely session-scoped: the moment a user disconnects
or issues PART, all record of their association with a channel is lost. Operators must
re-invite returning users, ban lists are the only persistent per-user state, and there is
no way to distinguish a long-standing trusted member from an anonymous passer-by. This
extension provides a first-class persistent membership concept that co-exists cleanly
with standard IRC session mechanics and degrades gracefully for clients and servers that
do not support it.

## Capability

This extension is advertised as the `rsr.chat/channel-membership` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/channel-membership
```

No capability value is used. Limits and feature flags are communicated via ISUPPORT (see
below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `MEMBERMAX=<n>` — the maximum number of persistent members a single channel may hold.
  A value of `0` indicates no server-imposed limit.
- `MEMBERROLES=<r1>,<r2>,...` — a comma-separated ordered list of role names supported
  by this server, from highest to lowest precedence. Servers MUST include at least
  `owner,admin,op,voice,member` in this list. Servers MAY define additional roles between
  these. Role names are case-insensitive ASCII.
- `MEMBERPERSIST=1` — present if the server retains membership data across restarts.
  Absent if membership is in-memory only and may be lost on restart.

Example:

```
:server 005 nick MEMBERMAX=500 MEMBERROLES=owner,admin,op,voice,member MEMBERPERSIST=1 :are supported by this server
```

If any of these values change after a configuration reload, a fresh `RPL_ISUPPORT` MUST be
sent to all connected clients.

## Terminology

- **Persistent member**: an account that has been added to a channel's membership list.
  Membership survives session end.
- **Active member**: a persistent member who is also currently connected and has an active
  JOIN state in the channel.
- **Transient member**: a user currently joined to a channel who is not a persistent
  member. Standard IRC session mechanics apply to transient members without modification.
- **Role**: a named rank assigned to a persistent member, drawn from the server's
  `MEMBERROLES` list.
- **Account**: a registered services identity, as established by IRCv3 `account-tag` or
  equivalent mechanisms. Persistent membership is always bound to an account, never to a
  nick or hostmask alone.

## Account Requirement

Persistent membership is identity-based and requires that the target user hold a
recognised account. Servers MUST refuse CHMEMBER ADD for a nick that is not currently
identified to an account, returning ERR_NOTREGISTERED. Membership records store the
account name, not the nick. When an account holder connects and joins a channel, the
server resolves their current nick to their account for membership lookup.

Servers that do not support account services MUST NOT advertise this capability.

## Roles

Every persistent member is assigned exactly one role at the time they are added. The
default role for CHMEMBER ADD, if no role is specified, is `member`. Roles map to
standard IRC channel privileges as follows:

| Role     | IRC mode equivalent | Effective privilege                      |
|----------|---------------------|------------------------------------------|
| `owner`  | `+q` (where supported), else `+o` | Full channel control          |
| `admin`  | `+a` (where supported), else `+o` | Administrative privileges     |
| `op`     | `+o`                | Standard channel operator                |
| `voice`  | `+v`                | Ability to speak in moderated channels   |
| `member` | (none)              | Recognised member, no extra privilege    |

When a persistent member joins a channel as an active member, the server MUST
automatically apply the IRC mode corresponding to their role. Servers MUST do this
without requiring operator intervention and MUST complete the mode application before
sending the JOIN confirmation to the channel.

If `multi-prefix` is negotiated, the automatically applied mode MUST appear in NAMES and
WHO responses as normal.

## Commands

### CHMEMBER ADD

```
CHMEMBER <channel> ADD <nick> [<role>]
```

Adds the account currently identified by `<nick>` to the channel's persistent membership
list, optionally at the specified role. If `<role>` is omitted, `member` is assumed.

The requesting client MUST be an active member with a role of `op` or higher, or a server
operator, unless overridden by `rsr.chat/rbac`.

On success the server MUST broadcast a `CHMEMBER ADD` message to all active channel
members that have negotiated this capability:

```
:adder!user@host CHMEMBER #channel ADD <accountname> <role>
```

### CHMEMBER REMOVE

```
CHMEMBER <channel> REMOVE <nick> [:<reason>]
```

Removes the persistent membership of the account identified by `<nick>`. If the target is
currently active in the channel, they are not forced to part; their session continues as a
transient member until they disconnect or part.

On success the server MUST broadcast a `CHMEMBER REMOVE` message to all active channel
members that have negotiated this capability:

```
:remover!user@host CHMEMBER #channel REMOVE <accountname> [:<reason>]
```

### CHMEMBER SETROLE

```
CHMEMBER <channel> SETROLE <nick> <role>
```

Changes the role of an existing persistent member. The requesting client's role MUST be
higher in precedence than the target's current role and higher than or equal to the
requested new role. A client may not promote another member to a role equal to or higher
than their own.

On success the server MUST broadcast a `CHMEMBER SETROLE` message to all active channel
members that have negotiated this capability:

```
:setter!user@host CHMEMBER #channel SETROLE <accountname> <role>
```

If the target is currently active in the channel, the server MUST update their IRC
channel mode to reflect the new role immediately.

### CHMEMBER LIST

```
CHMEMBER <channel> LIST
```

Requests the full persistent membership list for the channel. The server responds with
one or more RPL_MEMBERENTRY numerics followed by RPL_MEMBEREND. If IRCv3 `batch` is
negotiated the entire list MUST be wrapped in a `rsr.chat/memberlist` batch.

### CHMEMBER INVITE

```
CHMEMBER <channel> INVITE <nick>
```

Grants a one-time session join permission to the specified nick for a channel that is
invite-only (+i), without adding them to the persistent membership list. This is
equivalent to a standard INVITE but issued via the membership command surface. If the
target is a persistent member, this command has no effect and the server MUST reply with
ERR_ALREADYMEMBER.

## Server Numerics

| Symbolic name          | Description                                                      |
|------------------------|------------------------------------------------------------------|
| `RPL_MEMBERENTRY`      | A single persistent membership entry                             |
| `RPL_MEMBEREND`        | Marks the end of a CHMEMBER LIST response                        |
| `ERR_NOTREGISTERED`    | Target nick is not identified to an account                      |
| `ERR_NOTAMEMBER`       | Target account is not a persistent member of this channel        |
| `ERR_ALREADYMEMBER`    | Target account is already a persistent member of this channel    |
| `ERR_MEMBERROLE`       | The requesting client's role is insufficient for this operation  |
| `ERR_MEMBERFULL`       | The channel has reached the MEMBERMAX limit                      |
| `ERR_MEMBERROLEINVAL`  | The specified role is not in the server's MEMBERROLES list       |

`RPL_MEMBERENTRY` has the following format:

```
:server RPL_MEMBERENTRY <nick> <channel> <accountname> <role> <joined-timestamp> [flags]
```

Where:

- `accountname` is the account name of the member.
- `role` is their current role.
- `joined-timestamp` is an ISO 8601 UTC timestamp recording when they were first added to
  the membership list.
- `flags` is an optional space-separated list of server-supplied annotations. Currently
  defined flags:
  - `active` — the member is currently connected and joined.
  - `away` — the member is currently active but marked away (see IRCv3 Integration).
  - `offline` — the member holds membership but is not currently connected.

`RPL_MEMBEREND` has the following format:

```
:server RPL_MEMBEREND <nick> <channel> :End of channel membership list
```

## Interaction with Standard JOIN and PART

### JOIN by a persistent member

When a persistent member joins a channel the server MUST:

1. Process the JOIN normally, broadcasting it to all channel members.
2. Automatically apply the mode corresponding to their role before or atomically with the
   JOIN broadcast, such that receiving clients observe the mode in NAMES immediately.
3. Broadcast a `CHMEMBER ACTIVE` event to all active members that have negotiated this
   capability, indicating the member has transitioned from offline to active:

```
:server CHMEMBER #channel ACTIVE <accountname>
```

### PART or QUIT by a persistent member

When a persistent member parts or quits, their persistent membership record is NOT
removed. The server MUST broadcast a standard PART or QUIT as normal, and MUST broadcast
a `CHMEMBER INACTIVE` event to all remaining active members that have negotiated this
capability:

```
    :server CHMEMBER #channel INACTIVE <accountname>
```

The member's role is preserved for their next session.

### KICK of a persistent member

A KICK removes the target from the active session but MUST NOT remove their persistent
membership by default. Servers MUST accept an extended KICK syntax to simultaneously
remove persistent membership:

```
KICK <channel> <nick> [--remove-membership] [:<reason>]
```

If `--remove-membership` is present and the kicker's role is sufficient to call
CHMEMBER REMOVE, the persistent membership is also removed. Servers that do not support
the flag MUST treat a KICK as session-only.

### Channel destruction

If a channel becomes empty of active members, the server MUST NOT destroy the channel if
it has a non-empty persistent membership list. The channel persists in an idle state
until a persistent member returns. Servers MAY destroy channels with empty persistent
membership lists on the normal empty-channel timeout.

## NAMES and WHO Integration

When responding to NAMES (353) or WHO for a channel, servers MUST include only currently
active members in the reply, consistent with standard IRC behaviour. Persistent but
currently offline members MUST NOT appear in NAMES or WHO.

Clients that wish to see the full membership list including offline members MUST use
CHMEMBER LIST.

If `userhost-in-names` is negotiated, active persistent members appear with their current
hostmask as normal.

If `multi-prefix` is negotiated, the automatically applied role mode prefix appears in
NAMES as normal.

## IRCv3 Integration

### account-tag

Persistent membership is bound to account names resolved via the `account-tag` mechanism.
When a client with a recognised account joins a channel the server resolves the account
name to check for existing membership before processing the JOIN. Servers MUST NOT add
persistent members based on nick or hostmask alone.

### away-notify

When `away-notify` is negotiated, transitions between active and away state for persistent
members who are currently offline MUST NOT generate spurious AWAY notifications. Only
active members (currently joined) generate `away-notify` events as normal.

When a persistent member becomes active (joins) they receive the current away status of
all other active members in the join burst if `away-notify` is negotiated, consistent with
existing `away-notify` semantics.

### batch

CHMEMBER LIST responses MUST be wrapped in a `rsr.chat/memberlist` batch when the
requesting client has negotiated IRCv3 `batch`. Servers MUST NOT send RPL_MEMBERENTRY
outside of a batch when `batch` is negotiated.

### chathistory

Persistent members MUST be permitted to request message history for channels in which
they hold membership, even when they are not currently joined. A persistent member
with role `member` or above MAY issue a CHATHISTORY command targeting a channel they
belong to while offline. The server MUST honour this request and return history up to the
channel's retention window.

Transient members (users who were previously joined but hold no persistent membership)
MUST NOT receive history for periods when they were not actively joined, consistent with
standard chathistory access rules.

### extended-join

When `extended-join` is negotiated, the JOIN broadcast for a persistent member MUST
include their account name in the extended format. Receiving clients can thereby
correlate the joining user with their persistent membership entry.

### invite-notify

When `invite-notify` is negotiated, CHMEMBER INVITE events are broadcast to all active
channel members that have negotiated both `invite-notify` and `rsr.chat/channel-membership`
using the standard INVITE notification format, consistent with `invite-notify` semantics.

### monitor

The MONITOR list of persistent members is independent of channel membership. If a
persistent member for whom another client has a MONITOR entry connects, the standard
RPL_MONONLINE notification is sent as normal. No MONITOR events are generated solely
because of membership list changes.

## Reserved Metadata Keys

When both `rsr.chat/channel-metadata` and `rsr.chat/channel-membership` are negotiated,
the following keys exist implicitly on every channel and cannot be deleted:

### `membership-count`

Type: `uint`. The current number of persistent members. Updated by the server on each
CHMEMBER ADD or REMOVE. Read-only; attempts to set via CHANMETA SET MUST result in
ERR_CHANMETAREADONLY.

### `membership-open`

Type: `bool`. When `true`, any authenticated user may add themselves to the membership
list via `CHMEMBER <channel> ADD <own-nick>` without requiring operator approval. When
`false`, only users with role `op` or higher may add new members. Default: `false`.

## Guild Integration

When `rsr.chat/guild` is negotiated, the guild operator role implicitly supersedes
`owner` for all channels within the guild. Guild operators MAY add, remove, or change the
role of any persistent member in any guild channel without holding a channel-level role
themselves.

Persistent membership records MUST be exported and visible within guild-scoped CHMEMBER
LIST queries if the guild specification defines a guild-wide member listing mechanism.
The specific syntax for guild-wide membership queries is defined by `rsr.chat/guild`.

## Category Integration

When `rsr.chat/channel-categories` is negotiated, CHMEMBER ADD MAY be performed at a
category scope, adding a user as a persistent member of all channels that currently exist
within that category:

```
CHMEMBER #engineering/ ADD <nick> <role>
```

This is a convenience operation equivalent to issuing CHMEMBER ADD individually for each
matching channel. The server MUST apply the operation atomically and MUST reply with
a success or error numeric for each channel individually in a `rsr.chat/memberlist` batch.
Channels created within the category after a scoped ADD are not automatically affected;
the scope applies only at the time of the command.

## RBAC Integration

When `rsr.chat/rbac` is negotiated, the following permission identifiers are defined by
this extension:

| Permission identifier       | Description                                             |
|-----------------------------|---------------------------------------------------------|
| `membership.list`           | View the persistent membership list                     |
| `membership.add`            | Add a new persistent member                             |
| `membership.remove`         | Remove a persistent member                              |
| `membership.setrole`        | Change a member's role                                  |
| `membership.invite`         | Issue a one-time session invite                         |
| `membership.self.add`       | Add oneself to a channel's membership list              |
| `membership.self.remove`    | Remove oneself from a channel's membership list         |

RBAC rules for these permissions MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`.

Furthermore, the `ISUPPORT` parameter `MEMBERROLES` MUST contain the value `rsr.chat/rbac`,
which MUST be understood by clients implementing both extensions.

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, the following predicates are defined
for use in moderation rule conditions (normative syntax defined by that specification):

| Predicate                       | Matches when                                         |
|---------------------------------|------------------------------------------------------|
| `membership:is-member`          | The acting user is a persistent member               |
| `membership:role=<role>`        | The acting user holds exactly the specified role     |
| `membership:role>=<role>`       | The acting user's role is equal to or higher than specified |
| `membership:is-active`          | The user is a persistent member and currently joined |
| `membership:is-offline`         | The user holds membership but is not currently joined|
| `membership:member-count>=<n>`  | The channel has at least n persistent members        |

These predicates may be used to restrict posting, joining, or other actions to persistent
members only, to gate on role level, or to allow offline members to perform out-of-session
actions such as chathistory requests.

## Interaction with Non-Negotiating Clients

Clients that have not negotiated `rsr.chat/channel-membership` observe no changes to
standard IRC mechanics. NAMES and WHO responses contain only active members as normal.
Automatic mode application when a persistent member joins is indistinguishable from a
standard operator applying modes manually. CHMEMBER commands sent by non-negotiating
clients MUST be rejected with ERR_UNKNOWNCOMMAND.

Non-negotiating clients that are persistent members receive the same automatic mode
application on join as negotiating clients, since that is expressed through standard IRC
mode messages they already understand.

## Examples

### Adding a persistent member

```
C: CHMEMBER #engineering/general ADD bob op
S: :alice!alice@host CHMEMBER #engineering/general ADD bob op
    (broadcast to all active members that negotiated the capability)
```

### Persistent member joins and receives automatic mode

(bob connects and joins #engineering/general, where he is a persistent member with role op)

```
S: :bob!bob@host JOIN #engineering/general
S: :server MODE #engineering/general +o bob
S: :server CHMEMBER #engineering/general ACTIVE bob
    (broadcast to active members that negotiated the capability)
```

### CHMEMBER LIST with batch

```
C: CHMEMBER #engineering/general LIST
S: :server BATCH +ml rsr.chat/memberlist #engineering/general
S: @batch=ml :server RPL_MEMBERENTRY alice #engineering/general alice_acct owner 2023-11-01T09:00:00.000Z active
S: @batch=ml :server RPL_MEMBERENTRY alice #engineering/general bob_acct op 2023-11-03T14:22:10.000Z active away
S: @batch=ml :server RPL_MEMBERENTRY alice #engineering/general carol_acct member 2024-01-15T08:45:00.000Z offline
S: :server BATCH -ml
S: :server RPL_MEMBEREND alice #engineering/general :End of channel membership list
```

### Persistent member parts; membership retained

```
C: PART #engineering/general
S: :bob!bob@host PART #engineering/general
S: :server CHMEMBER #engineering/general INACTIVE bob
    (broadcast to remaining active members that negotiated the capability)
(bob's membership record and role are preserved)
```

### Kick without membership removal

```
C: KICK #engineering/general bob :Please rejoin when available
S: :alice!alice@host KICK #engineering/general bob :Please rejoin when available
(bob is removed from the active session; persistent membership is unaffected)
```

### Kick with membership removal

```
C: KICK #engineering/general bob --remove-membership :Conduct violation
S: :alice!alice@host KICK #engineering/general bob :Conduct violation
S: :alice!alice@host CHMEMBER #engineering/general REMOVE bob :Conduct violation
    (broadcast to active members that negotiated the capability)
```

### Role change while member is active

```
C: CHMEMBER #engineering/general SETROLE bob voice
S: :alice!alice@host CHMEMBER #engineering/general SETROLE bob voice
S: :server MODE #engineering/general -o+v bob
    (mode updated immediately since bob is currently active)
```

### Offline member requesting history via chathistory

```
(carol is a persistent member but is not currently joined)
C: CHATHISTORY LATEST #engineering/general * 50
S: (history returned for #engineering/general up to retention window)
```

### Insufficient role error

(bob has role voice and attempts to add a new member)

```
C: CHMEMBER #engineering/general ADD carol member
S: :server ERR_MEMBERROLE bob #engineering/general :Insufficient role to add members
```

### Category-scoped ADD

```
C: CHMEMBER #engineering/ ADD carol member
S: :server BATCH +cm rsr.chat/memberlist #engineering/
S: @batch=cm :server RPL_MEMBERENTRY alice #engineering/general carol_acct member 2024-03-15T10:00:00.000Z offline
S: @batch=cm :server RPL_MEMBERENTRY alice #engineering/deploys carol_acct member 2024-03-15T10:00:00.000Z offline
S: :server BATCH -cm
```