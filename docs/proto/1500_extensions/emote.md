---
title: rsr.chat/emote
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/emote

This extension defines a server-managed registry of named custom emotes that may be
referenced inline in messages using a standard token syntax. Emotes may be scoped to the
server, to a guild, or to an individual channel. Each emote is associated with an image
asset whose URL is advertised to clients.

## Motivation

Custom emotes are a first-class feature of modern messaging platforms. Without a standard
mechanism, IRC clients cannot interoperate on emote identity, availability, or rendering.
Ad-hoc conventions such as `:name:` text substitutions exist only by client-side agreement
and carry no server-authoritative asset information. This extension defines a typed,
namespaced, server-authoritative emote registry with a negotiated inline token syntax,
structured asset delivery, and access controls.

## Capability

This extension is advertised as the `rsr.chat/emote` capability during CAP negotiation:

```
CAP * LS :rsr.chat/emote
```

No capability value is used. Limits and feature flags are communicated via ISUPPORT (see
below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `EMOTEMAX=<n>` — the maximum number of emotes permitted in the server-global registry.
  A value of `0` indicates no server-imposed limit.
- `EMOTEGUILDMAX=<n>` — the maximum number of emotes permitted per guild registry when
  `rsr.chat/guild` is negotiated. A value of `0` indicates no limit.
- `EMOTECHANMAX=<n>` — the maximum number of emotes permitted per channel registry.
  A value of `0` indicates no limit.
- `EMOTELEN=<n>` — the maximum byte length of an emote name, excluding the surrounding
  colons. Servers MUST NOT enforce a value lower than 2 or higher than 64.
- `EMOTEFORMATS=<f1>,<f2>,...` — a comma-separated list of image formats accepted for
  emote asset uploads, in server preference order. Servers MUST support at least `png`
  and `webp`. Servers MAY additionally support `gif` and `avif`.
- `EMOTEANIM=1` — present if the server supports animated emote assets (e.g. animated
  `webp` or `gif`). Absent if only static assets are accepted.

Example:

```
:server 005 nick EMOTEMAX=500 EMOTEGUILDMAX=100 EMOTECHANMAX=20 EMOTELEN=32 EMOTEFORMATS=webp,png,gif EMOTEANIM=1 :are supported by this server
```

If any of these values change after a configuration reload, a fresh RPL_ISUPPORT MUST be
sent to all connected clients.

## Terminology

- **Emote**: a named image asset registered with the server and referenceable inline in
  messages using the token syntax defined by this extension.
- **Emote name**: a case-insensitive ASCII identifier matching `[A-Za-z0-9][A-Za-z0-9_-]*`
  of at most `EMOTELEN` bytes.
- **Emote identifier**: the fully-qualified reference to an emote, consisting of an
  optional namespace prefix and the emote name enclosed in colons:
  `:name:` (global), `:guildname/name:` (guild-scoped), `:#channel/name:` or
  `:#category//name:` (channel-scoped). See Emote Identifiers.
- **Emote scope**: the registry in which an emote is defined — server-global, guild, or
  channel.
- **Emote pack**: a named, ordered collection of emotes within a scope that may be
  enabled or disabled as a unit. See Emote Packs.
- **Asset URL**: an HTTPS URL at which the emote image asset may be retrieved by clients.
  Servers are responsible for hosting and serving these assets.
- **Animated emote**: an emote whose asset is an animated image. Only supported when
  `EMOTEANIM=1` is advertised.
- **Unicode emoji**: a Unicode emoji scalar or fully-qualified emoji sequence. Unicode
  emoji are valid emote references in any context that accepts emote identifiers but are
  not managed by this extension and have no entry in the server registry.

## Emote Identifiers

An emote identifier is a token that uniquely addresses an emote within the context of a
message. The syntax is:

```
:<namespace/><name>:
```

Where `<namespace/>` is an optional scope prefix:

| Identifier form              | Scope           | Example                   |
|------------------------------|-----------------|---------------------------|
| `:name:`                     | Server-global   | `:thumbsup:`              |
| `:guildname/name:`           | Guild           | `:acmecorp/blobwave:`     |
| `:#channelname/name:`        | Channel         | `:#engineering/approved:` |

Emote names within a namespace MUST be unique. An emote in a guild namespace does not
conflict with a server-global emote of the same name; when both exist the guild-scoped
emote takes precedence for users who are members of that guild. Channel-scoped emotes
take precedence over both.

Precedence order (highest to lowest):

1. Channel-scoped emote matching the channel of the current message.
2. Guild-scoped emote matching the guild of the current channel (if any).
3. Server-global emote.

When a client sends a bare `:name:` token, the server MUST resolve it according to this
precedence order for the target channel. The server MUST rewrite the outgoing message tag
(see Inline Emote Tags) to include the fully-qualified identifier so that receiving
clients do not need to perform their own resolution.

## Asset Requirements

Each emote asset MUST satisfy the following constraints:

- Format: one of the formats listed in `EMOTEFORMATS`.
- Dimensions: square, between 16×16 and 256×256 pixels inclusive.
- File size: at most 256 KiB for static assets; at most 512 KiB for animated assets.
- Transport: served over HTTPS. Self-signed certificates are not permitted for
  production deployments.

Servers MAY transcode uploaded assets to their preferred format. If transcoding is
performed, the asset URL advertised to clients MUST reference the transcoded asset, not
the original upload.

## Inline Emote Token Syntax

Within a PRIVMSG or NOTICE body, any sequence matching the emote identifier syntax is
treated as an inline emote token. Tokens MAY appear at any position in the body.

Clients MUST NOT rely on the server to transform or replace the token text within the
message body itself. The message body is transmitted verbatim. Emote resolution is
communicated exclusively through message tags (see Inline Emote Tags), allowing
non-negotiating clients to read the raw token text as plain text.

## Inline Emote Tags

When a server delivers a PRIVMSG or NOTICE that contains one or more inline emote tokens
to a client that has negotiated this capability, it MUST attach a
`rsr.chat/emotes` message tag. The tag value is a semicolon-delimited list of resolved
emote entries, one per token occurrence in body order:

```
emotes=<token-index>:<qualified-id>:<asset-url>[:<flags>]
```

Fields:

| Field            | Description                                                          |
|------------------|----------------------------------------------------------------------|
| `token-index`    | Zero-based index of the token within the message body, counting from the start of the body string |
| `qualified-id`   | The fully-qualified emote identifier after server-side resolution    |
| `asset-url`      | HTTPS URL of the emote image asset                                   |
| `flags`          | Optional comma-separated flags. Currently defined: `animated`        |

Multiple entries are separated by semicolons. Example:

```
rsr.chat/emotes=0::acmecorp/blobwave:https://cdn.example.com/emotes/blobwave.webp:animated;14::thumbsup:https://cdn.example.com/emotes/thumbsup.webp
```

If a token cannot be resolved (the emote does not exist in any applicable registry or the
user lacks access), the server MUST omit that index from the tag. The unresolvable token
remains in the message body as plain text. The server MUST NOT reject a message solely
because it contains an unresolvable emote token.

Clients that have not negotiated this capability receive messages without the
`rsr.chat/emotes` tag and see token text as plain text, which is the correct degraded
behaviour.

## Commands

### EMOTE ADD

```
EMOTE ADD <scope> <name> <format> <asset-url>
```

Registers a new emote. `<scope>` is one of:

- `global` — server-global registry (server operators only).
- `guild:<guildname>` — guild registry (guild operators or users with `emote.add` RBAC
  permission for the guild).
- `chan:<channel>` — channel registry (channel operators with role `op` or higher, or
  users with `emote.add` RBAC permission for the channel).

`<format>` MUST be one of the formats in `EMOTEFORMATS`. `<asset-url>` MUST be an HTTPS
URL. The server MUST fetch and validate the asset at registration time, checking format,
dimensions, and file size. If validation fails the server MUST reply with
ERR_EMOTEINVALIDASSET and the emote MUST NOT be registered.

On success the server MUST broadcast an `EMOTE ADD` notification to all clients that have
negotiated this capability and have visibility of the relevant scope:

```
:adder!user@host EMOTE ADD <scope> <name> <format> <asset-url>
```

### EMOTE REMOVE

```
EMOTE REMOVE <scope> <name>
```

Removes a registered emote. The requesting client MUST hold the same permissions as
required for EMOTE ADD in the relevant scope.

Removing an emote does not retroactively alter delivered messages. Existing `rsr.chat/emotes`
tags in history reference the asset URL directly; clients that cache assets will continue
to render them. Servers MAY expire asset hosting for removed emotes after a grace period
of at least 30 days.

On success the server MUST broadcast an `EMOTE REMOVE` notification to all clients that
have negotiated this capability and have visibility of the scope:

```
:remover!user@host EMOTE REMOVE <scope> <name>
```

### EMOTE RENAME

```
EMOTE RENAME <scope> <name> <newname>
```

Renames an existing emote. The emote's asset and all other properties are unchanged. The
server MUST broadcast an `EMOTE RENAME` notification to all clients that have negotiated
this capability and have visibility of the scope:

```
:renamer!user@host EMOTE RENAME <scope> <name> <newname>
```

### EMOTE LIST

```
    EMOTE LIST [<scope>]
```

Requests the full emote registry for a given scope, or all scopes visible to the calling
client if `<scope>` is omitted. The server responds with zero or more RPL_EMOTEENTRY
numerics followed by RPL_EMOTEEND. If IRCv3 `batch` is negotiated the entire response
MUST be wrapped in a `rsr.chat/emotelist` batch.

### EMOTE INFO

```
EMOTE INFO <identifier>
```

Requests metadata for a single emote by its fully-qualified identifier. The server
responds with a single RPL_EMOTEENTRY followed by RPL_EMOTEEND.

## Emote Packs

An emote pack is a named, ordered collection of emotes within a single scope. Packs allow
administrators to group thematically related emotes and allow users to enable or disable
sets of emotes as a unit for display purposes. Pack management is performed via the
following commands.

### EMOTEPAK CREATE

```
EMOTEPAK CREATE <scope> <packname>
```

Creates an empty emote pack within the given scope.

### EMOTEPAK ADD

```
EMOTEPAK ADD <scope> <packname> <emotename>
```

Adds an existing emote within the same scope to a pack. An emote may belong to multiple
packs.

### EMOTEPAK REMOVE

```
EMOTEPAK REMOVE <scope> <packname> <emotename>
```

Removes an emote from a pack without deleting the emote itself.

### EMOTEPAK DELETE

```
EMOTEPAK DELETE <scope> <packname>
```

Deletes an emote pack. The emotes within the pack are not deleted.

### EMOTEPAK LIST

```
EMOTEPAK LIST [<scope>]
```

Lists all packs and their member emotes within the given scope, or all visible scopes if
omitted. If IRCv3 `batch` is negotiated the response MUST be wrapped in a
`rsr.chat/emotepklist` batch.

## Server Numerics

| Symbolic name             s| Description                                                     |
|---------------------------|-----------------------------------------------------------------|
| `RPL_EMOTEENTRY`          | A single emote registry entry                                   |
| `RPL_EMOTEEND`            | Marks the end of an EMOTE LIST or EMOTE INFO response           |
| `RPL_EMOTEPKENTRY`        | A single emote pack entry                                       |
| `RPL_EMOTEPKEND`          | Marks the end of an EMOTEPAK LIST response                      |
| `ERR_EMOTEUNKNOWN`        | The requested emote does not exist                              |
| `ERR_EMOTENAMEEXISTS`     | An emote with this name already exists in the target scope      |
| `ERR_EMOTENAMEINVAL`      | The emote name does not match the required pattern              |
| `ERR_EMOTEINVALIDASSET`   | The asset URL failed format, dimension, or size validation      |
| `ERR_EMOTENOPERM`         | The client lacks permission for this emote operation            |
| `ERR_EMOTEFULL`           | The target scope has reached its emote limit                    |
| `ERR_EMOTEPACKUNKNOWN`    | The specified emote pack does not exist                         |
| `ERR_EMOTEPACKEXISTS`     | An emote pack with this name already exists in the target scope |

`RPL_EMOTEENTRY` has the following format:

```
:server RPL_EMOTEENTRY <nick> <scope> <name> <qualified-id> <format> <asset-url> <added-timestamp> <added-by> [flags]
```

Where `flags` is an optional space-separated list of server-supplied annotations.
Currently defined flags:

- `animated` — the asset is an animated image.
- `pack:<packname>` — the emote belongs to the named pack (may appear multiple times).

`RPL_EMOTEPKENTRY` has the following format:

```
:server RPL_EMOTEPKENTRY <nick> <scope> <packname> <emotename> <qualified-id>
```

## Emote Availability by Scope

### Server-global emotes

Server-global emotes are available to all connected users on the server regardless of
channel or guild membership.

### Guild-scoped emotes

Guild-scoped emotes are available to members of the guild in which they are registered.
When `rsr.chat/guild` is negotiated, guild membership is determined by the guild
specification. Guild emotes are not available to users who are not members of the guild.

### Channel-scoped emotes

Channel-scoped emotes are available to all users currently joined to the channel in which
they are registered. When `rsr.chat/channel-membership` is negotiated, persistent members
who are offline may still reference channel emotes in CHATHISTORY replies but MUST NOT
use them in new messages.

## Reserved Channel Metadata Key

When both `rsr.chat/channel-metadata` and `rsr.chat/emote` are negotiated, the following
key exists implicitly on every channel and cannot be deleted:

### `emotes-enabled`

Type: `bool`. When `false`, inline emote tokens in messages sent to this channel are not
resolved and no `rsr.chat/emotes` tag is attached. Custom emote references pass through
as plain text for all clients. Default: `true`.

This key may be set by channel operators or by users with the `emote.configure` RBAC
permission.

## History Integration

When a client requests message history via CHATHISTORY or an equivalent mechanism,
replayed messages MUST include the `rsr.chat/emotes` tag as originally computed at
delivery time. Servers MUST NOT re-resolve emote tokens during history replay. Asset URLs
embedded in history tags are not guaranteed to remain valid indefinitely; see the grace
period note under EMOTE REMOVE.

## Guild Integration

When `rsr.chat/guild` is negotiated:

- Guild operators MAY manage the guild emote registry via EMOTE ADD, EMOTE REMOVE, and
  EMOTE RENAME targeting `guild:<guildname>`.
- Guild-scoped emotes are available to all guild members across all channels within the
  guild.
- When resolving a bare `:name:` token in a guild channel, the guild registry is searched
  after the channel registry but before the server-global registry, as per the precedence
  order defined in Emote Identifiers.
- Guild emote packs MAY be defined via EMOTEPAK commands targeting the guild scope.

## RBAC Integration

When `rsr.chat/rbac` is negotiated, the following permission identifiers are defined by
this extension:

| Permission identifier     | Description                                               |
|---------------------------|-----------------------------------------------------------|
| `emote.add`               | Register a new emote in this scope                        |
| `emote.remove`            | Remove an emote from this scope                           |
| `emote.rename`            | Rename an emote in this scope                             |
| `emote.list`              | List emotes in this scope                                 |
| `emote.use`               | Use emotes from this scope in messages                    |
| `emote.use.animated`      | Use animated emotes from this scope                       |
| `emote.pack.manage`       | Create, delete, and modify emote packs in this scope      |
| `emote.configure`         | Set the `emotes-enabled` channel metadata key             |

Servers MAY restrict use of animated emotes separately from static emotes via
`emote.use.animated`. If a client without `emote.use.animated` uses an animated emote
token, the server MUST resolve it to the static fallback asset if one exists, or treat
it as unresolvable if none does.

RBAC rules for emote permissions MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`.

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, the following predicates are defined
for use in moderation rule conditions (normative syntax defined by that specification):

| Predicate                          | Matches when                                         |
|------------------------------------|------------------------------------------------------|
| `emote:has-emote`                  | The message body contains at least one emote token   |
| `emote:count>=<n>`                 | The message contains at least n emote tokens         |
| `emote:is-animated`                | At least one resolved emote in the message is animated |
| `emote:scope=global`               | At least one resolved emote is from the global scope |
| `emote:scope=guild`                | At least one resolved emote is from a guild scope    |
| `emote:scope=channel`              | At least one resolved emote is from a channel scope  |
| `emote:id=<qualified-id>`          | The specific emote appears in the message            |
| `emote:unresolved`                 | At least one emote token could not be resolved       |

These predicates may be used to restrict emote-heavy messages in low-bandwidth channels,
prevent animated emotes in accessibility-sensitive contexts, or flag messages containing
specific emotes for moderation review.

## Interaction with rsr.chat/reactions

`rsr.chat/reactions` depends on this extension for emote validation. All emote identifier
syntax, namespacing, precedence, and validation rules defined here apply equally to
reaction emotes. The following integration points apply:

- Unicode emoji are valid reaction emotes without a registry entry.
- Custom emote reaction identifiers MUST reference a registered emote in the server,
  guild, or channel registry visible to both the reactor and the target channel.
- The server MUST validate reaction emote identifiers using the same resolution and
  precedence rules as inline message emotes.
- The `rsr.chat/reaction-summary` tag defined by `rsr.chat/reactions` uses the same
  emote identifier syntax defined here.

## Interaction with Non-Negotiating Clients

Non-negotiating clients receive messages with emote tokens as plain text and without the
`rsr.chat/emotes` tag. They interact with emote tokens solely as the string `:name:` or
similar. EMOTE and EMOTEPAK commands sent by non-negotiating clients MUST be rejected with
ERR_UNKNOWNCOMMAND. This is the correct degraded behaviour and requires no special server
handling beyond standard tag omission.

## Examples

### Capability negotiation

```
C: CAP LS 302
S: CAP * LS :rsr.chat/emote
C: CAP REQ :rsr.chat/emote
S: CAP * ACK :rsr.chat/emote
C: NICK alice
C: USER alice 0 * :Alice
S: 005 alice EMOTEMAX=500 EMOTEGUILDMAX=100 EMOTECHANMAX=20 EMOTELEN=32 EMOTEFORMATS=webp,png,gif EMOTEANIM=1 :are supported by this server
S: 001 alice :Welcome to the network
```

### Registering a server-global emote

```
C: EMOTE ADD global thumbsup webp https://cdn.example.com/emotes/thumbsup.webp
S: :serverop!serverop@host EMOTE ADD global thumbsup webp https://cdn.example.com/emotes/thumbsup.webp
    (broadcast to all negotiating clients)
```

### Registering a guild-scoped emote

```
C: EMOTE ADD guild:acmecorp blobwave webp https://cdn.acmecorp.example/emotes/blobwave.webp
S: :alice!alice@host EMOTE ADD guild:acmecorp blobwave webp https://cdn.acmecorp.example/emotes/blobwave.webp
    (broadcast to acmecorp guild members that negotiated the capability)
```

### Registering a channel-scoped emote

```
C: EMOTE ADD chan:#engineering/general approved webp https://cdn.example.com/emotes/approved.webp
S: :alice!alice@host EMOTE ADD chan:#engineering/general approved webp https://cdn.example.com/emotes/approved.webp
    (broadcast to #engineering/general members that negotiated the capability)
```

### Message delivery with inline emote tags

```
C: PRIVMSG #engineering/general :Great work :blobwave: :thumbsup:

(Negotiating recipients receive:)
@MSGID=01HQ7K2N4R8VXJD3MCWBPZT56Y;rsr.chat/emotes=12::acmecorp/blobwave:https://cdn.acmecorp.example/emotes/blobwave.webp:animated;23::thumbsup:https://cdn.example.com/emotes/thumbsup.webp :alice!alice@host PRIVMSG #engineering/general :Great work :blobwave: :thumbsup:

(Non-negotiating recipients receive:)
:alice!alice@host PRIVMSG #engineering/general :Great work :blobwave: :thumbsup:
```

### Bare name resolved with guild precedence

```
(alice is in guild acmecorp which has a guild-scoped emote named "wave";
    the server also has a global emote named "wave")
C: PRIVMSG #engineering/general :See you tomorrow :wave:
S: (rsr.chat/emotes tag carries :acmecorp/wave: not :wave:, guild takes precedence)
```

### Unresolvable emote token

```
C: PRIVMSG #engineering/general :Check this out :nonexistentemote:
S: (rsr.chat/emotes tag omits the unresolvable token; :nonexistentemote: is plain text)
```

### EMOTE LIST with batch

```
C: EMOTE LIST global
S: :server BATCH +el rsr.chat/emotelist global
S: @batch=el :server RPL_EMOTEENTRY alice global thumbsup :thumbsup: webp https://cdn.example.com/emotes/thumbsup.webp 2024-01-10T09:00:00.000Z serverop
S: @batch=el :server RPL_EMOTEENTRY alice global wave :wave: webp https://cdn.example.com/emotes/wave.webp 2024-01-10T09:01:00.000Z serverop pack:basics
S: :server BATCH -el
S: :server RPL_EMOTEEND alice global :End of emote list
```

### Creating and populating an emote pack

```
C: EMOTEPAK CREATE guild:acmecorp reactions
C: EMOTEPAK ADD guild:acmecorp reactions blobwave
C: EMOTEPAK ADD guild:acmecorp reactions blobapprove
C: EMOTEPAK ADD guild:acmecorp reactions blobsad
```

### Asset validation failure

```
C: EMOTE ADD chan:#engineering/general toobig webp https://cdn.example.com/emotes/toobig.webp
S: :server ERR_EMOTEINVALIDASSET alice :Asset exceeds maximum file size of 256 KiB
```

### Emote limit reached

```
C: EMOTE ADD chan:#engineering/general newemote webp https://cdn.example.com/emotes/newemote.webp
S: :server ERR_EMOTEFULL alice chan:#engineering/general :Channel emote limit reached
```

### Category-scoped emote registration

```
C: EMOTE ADD chan:#engineering/ shipit webp https://cdn.example.com/emotes/shipit.webp
S: :server BATCH +ce rsr.chat/emotelist #engineering/
S: @batch=ce :server RPL_EMOTEENTRY alice chan:#engineering/general shipit :#engineering/general/shipit: webp https://cdn.example.com/emotes/shipit.webp 2024-03-15T10:00:00.000Z alice
S: @batch=ce :server RPL_EMOTEENTRY alice chan:#engineering/deploys shipit :#engineering/deploys/shipit: webp https://cdn.example.com/emotes/shipit.webp 2024-03-15T10:00:00.000Z alice
S: :server BATCH -ce
```

### Removing an emote

```
C: EMOTE REMOVE guild:acmecorp blobwave
S: :alice!alice@host EMOTE REMOVE guild:acmecorp blobwave
    (broadcast to acmecorp guild members that negotiated the capability)
```