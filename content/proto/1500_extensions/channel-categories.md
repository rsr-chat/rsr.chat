---
title: rsr.chat/channel-categories
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/channel-categories

This extension allows IRC channels to be assigned hierarchical categories by embedding
forward-slash delimiters in the channel name. It provides structured namespace organisation
for servers that host many channels, and integrates with the `rsr.chat/guild`,
`rsr.chat/rbac`, and `rsr.chat/advanced-moderation` capabilities for scoped access control
and moderation.

## Motivation

Flat channel namespaces become difficult to navigate as server populations grow. Conventions
like `#project-general` and `#project-dev` exist only by social agreement and are not
machine-readable. This extension formalises a hierarchical naming scheme that clients and
servers can use to group, display, and govern channels structurally.

## Capability

This extension is advertised as the `rsr.chat/channel-categories` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/channel-categories
```

No capability value is used. The maximum permitted depth is communicated via ISUPPORT
(see below).

## Terminology

- **Segment**: a single slash-delimited component of a channel name.
- **Category**: any segment that is not the final (leaf) segment.
- **Leaf channel**: the final segment; the actual joinable channel.
- **Depth**: the total number of segments in a channel name. A plain channel such as
  `#general` has depth 1. `#engineering/general` has depth 2.
- **Guild prefix**: when `rsr.chat/guild` is negotiated, the outermost segment is
  interpreted as a guild identifier rather than a category (see Guild Integration).

## Channel Name Syntax

When this capability is in effect, channel names MAY contain forward-slash (`/`) characters
as segment delimiters. The following rules apply:

- The channel sigil (`#`, `&`, etc.) applies to the full name and is not repeated per
  segment.
- Each segment MUST be non-empty and MUST conform to the server's existing rules for
  channel name characters, excluding `/`.
- Consecutive slashes (`//`) are not permitted.
- A trailing slash is not permitted.
- Segment matching MUST follow the same case-folding rules as ordinary channel names.

Servers MUST reject channel names that violate these rules with `ERR_BADCHANMASK` (476).

## ISUPPORT Token

Servers MUST advertise the maximum permitted channel name depth via the `CHANLEVELS` token
in RPL_ISUPPORT (005):

```
:server 005 nick CHANLEVELS=2 :are supported by this server
```

The default value when only `rsr.chat/channel-categories` is negotiated (without
`rsr.chat/guild`) is `CHANLEVELS=2`, permitting one category level and one leaf channel
(e.g. `#category/channel`).

When `rsr.chat/guild` is additionally negotiated the value MUST be `CHANLEVELS=3`,
permitting a guild prefix, one category, and one leaf channel
(e.g. `#guild/category/channel`).

Servers MUST NOT advertise a `CHANLEVELS` value less than 2 when this capability is active,
and MUST NOT advertise a value greater than 3 unless a future extension explicitly permits
it.

If the server's depth limit changes (e.g. after a configuration reload), a fresh
`RPL_ISUPPORT` MUST be sent to all connected clients reflecting the updated value.

## Category Semantics

Categories are implicit namespaces derived from channel names. They are not themselves
joinable channels. A server hosting `#engineering/general` and `#engineering/deploys` has
an implicit `#engineering` category but does not thereby create a `#engineering` channel
unless that channel is independently created.

Clients SHOULD use category information to group channels in their UI. Servers MAY expose
a category listing command or numeric; such commands are outside the scope of this
specification.

## Client Behaviour

Clients that negotiate this capability MAY join and create channels with slash-delimited
names, subject to the depth limit advertised in CHANLEVELS.

Clients that have not negotiated `rsr.chat/channel-categories` MAY still join and
participate in channels whose names contain forward slashes, provided the channel already
exists. To such clients, a name like `#engineering/general` is an opaque string and is
treated identically to any other channel name. No hierarchy is implied or exposed on the
client side.

Servers MUST permit a JOIN from a non-negotiating client targeting an existing
slash-named channel and MUST NOT reply with `ERR_BADCHANMASK` (476) solely on the basis
that the name contains a slash. The client will receive the standard JOIN response,
`RPL_TOPIC`, `RPL_NAMREPLY`, and so on, as it would for any other channel.

### Example: non-negotiating client joins an existing hierarchical channel

(Server hosts #engineering/general, created by a client that negotiated the capability)

```
C: JOIN #engineering/general   (capability not negotiated)
S: :alice!alice@host JOIN #engineering/general
S: :server 332 alice #engineering/general :Engineering general chat
S: :server 353 alice = #engineering/general :alice bob
S: :server 366 alice #engineering/general :End of /NAMES list
```

### Example: non-negotiating client attempts to create a new hierarchical channel

```
C: JOIN #newcategory/newchannel   (capability not negotiated, channel does not exist)
S: :server 476 alice #newcategory/newchannel :Bad channel mask
```

## Server Behaviour

Servers MUST enforce the CHANLEVELS depth limit at channel creation and join time. Attempts
to join or create a channel whose depth exceeds the advertised limit MUST be rejected with
`ERR_BADCHANMASK` (476).

Servers MUST NOT silently truncate or rewrite channel names to fit within the depth limit.

## Guild Integration

When a client has negotiated both `rsr.chat/channel-categories` and `rsr.chat/guild`, the
outermost segment of a depth-3 channel name is treated as a guild identifier:

```
#<guild>/<category>/<leaf>
```

The guild segment identifies a top-level organisational boundary (analogous to a server
or workspace in other platforms). Guild membership, creation, and governance are defined
by the `rsr.chat/guild` specification. The categories extension does not define guild
semantics beyond the naming structure described here.

Depth-2 names (`#category/leaf`) remain valid when `rsr.chat/guild` is negotiated and are
interpreted as guild-unaffiliated channels.

## RBAC and Advanced Moderation Scoping

When a client has negotiated `rsr.chat/channel-categories` alongside `rsr.chat/rbac`,
`rsr.chat/advanced-moderation`, or both, rules defined by those capabilities MAY be scoped
to a category prefix rather than a single channel. A scoped rule applies to all leaf
channels whose name begins with the specified prefix.

A category scope is expressed as a channel sigil followed by the category segments and a
trailing slash:

```
#engineering/
```

This denotes the category scope covering all channels of the form `#engineering/<leaf>`.
When `rsr.chat/guild` is active, a guild-level scope MAY also be expressed:

```
#myguild/
```

This denotes all channels of the form `#myguild/<anything>`.

Scoped rules MUST be evaluated after any channel-specific rules and before server-wide
defaults. If a channel-specific rule and a category-scoped rule conflict, the
channel-specific rule takes precedence. If a category-scoped rule and a guild-scoped rule
conflict, the category-scoped rule takes precedence.

The specific syntax for defining and querying scoped rules is defined by the `rsr.chat/rbac`
and `rsr.chat/advanced-moderation` specifications respectively. This extension only defines
the scoping identifier format and precedence order.

## Examples

### Negotiation and joining a categorised channel

```
C: CAP LS 302
S: CAP * LS :rsr.chat/channel-categories rsr.chat/guild
C: CAP REQ :rsr.chat/channel-categories
S: CAP * ACK :rsr.chat/channel-categories
C: NICK alice
C: USER alice 0 * :Alice
S: 005 alice CHANLEVELS=2 :are supported by this server
S: 001 alice :Welcome to the network

C: JOIN #engineering/general
S: :alice!alice@host JOIN #engineering/general
```

### Depth limit exceeded

```
C: JOIN #engineering/backend/general
S: :server 476 alice #engineering/backend/general :Bad channel mask
```

### Guild-level naming with rsr.chat/guild negotiated

```
C: CAP REQ :rsr.chat/channel-categories rsr.chat/guild
S: CAP * ACK :rsr.chat/channel-categories rsr.chat/guild
S: 005 alice CHANLEVELS=3 :are supported by this server

C: JOIN #acmecorp/engineering/general
S: :alice!alice@host JOIN #acmecorp/engineering/general
```
