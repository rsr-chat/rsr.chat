---
title: rsr.chat/channel-meta
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/channel-meta

This extension allows arbitrary typed key-value metadata to be attached to channels.
Metadata entries may be short-form (single-line) or long-form (multi-line or extended).
The channel topic is defined as an implicit metadata key and may be managed through either
the standard TOPIC command or the metadata interface defined here.

## Motivation

IRC channels have historically carried only a small set of fixed attributes: a name, a
topic, a mode string, and a creation timestamp. Modern applications require richer
structured metadata — descriptions, icons, language tags, custom configuration values —
without polluting the mode string or abusing the topic field. This extension provides a
general-purpose, typed, access-controlled metadata store per channel.

## Capability

This extension is advertised as the `rsr.chat/channel-meta` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/channel-metadata
```

No capability value is used. Limits are communicated via ISUPPORT (see below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in RPL_ISUPPORT (005) when this capability is
active:

- `CHANMETAKEYS=<n>` — the maximum number of metadata keys permitted per channel.
- `CHANMETALEN=<n>` — the maximum byte length of a short-form metadata value.
- `CHANMETALONGLEN=<n>` — the maximum total byte length of a long-form metadata value.
  If the server does not support long-form values, this token MUST be omitted.

Example:

```
:server 005 nick CHANMETAKEYS=64 CHANMETALEN=390 CHANMETALONGLEN=8192 :are supported by this server
```

If any of these limits change (e.g. after a configuration reload), a fresh RPL_ISUPPORT
MUST be sent to all connected clients.

## Terminology

- **Key**: a metadata identifier. Keys are case-insensitive ASCII strings matching the
  pattern `[A-Za-z0-9][A-Za-z0-9_-]*`, with a maximum length of 64 characters.
- **Type**: a declared value type associated with a key at the time of its creation. The
  type is immutable for the lifetime of the key.
- **Short-form value**: a value that fits within a single IRC message line, up to
  `CHANMETALEN` bytes.
- **Long-form value**: a value that may span multiple lines or exceed `CHANMETALEN` bytes,
  up to `CHANMETALONGLEN` bytes. Long-form values require `rsr.chat/extended-line-length`
  or the multiline batch mechanism defined in this specification (see Long-Form Values).
- **Reserved key**: a key whose name and semantics are defined by this or another
  specification. Reserved keys MUST NOT be created or deleted by clients; they exist
  implicitly (see Reserved Keys).

## Value Types

Every metadata key carries a declared type. The following types are defined:

| Type identifier | Description                                          | Form       |
|-----------------|------------------------------------------------------|------------|
| `string`        | A UTF-8 string with no embedded newlines             | Short      |
| `int`           | A signed 64-bit decimal integer                      | Short      |
| `uint`          | An unsigned 64-bit decimal integer                   | Short      |
| `bool`          | Exactly the string `true` or `false`                 | Short      |
| `url`           | An absolute URL; scheme MUST be `http` or `https`    | Short      |
| `color`         | A CSS-style hex color code, e.g. `#ff8800`           | Short      |
| `text`          | A UTF-8 string that may contain newlines or exceed   | Long       |
|                 | `CHANMETALEN` bytes                                  |            |

Servers MUST reject values that do not conform to the declared type with
`ERR_CHANMETABADVALUE`.

## Reserved Keys

The following keys exist implicitly on every channel and do not need to be created. They
cannot be deleted.

### `topic`

The `topic` key mirrors the channel's topic. Its type is `text`.

- When a client sets the topic via the standard TOPIC command, the server MUST update the
  `topic` metadata key as if `CHANMETA SET` had been called.
- When a client sets the `topic` metadata key via `CHANMETA SET`, the server MUST broadcast
  a standard TOPIC notification to all channel members as if the TOPIC command had been
  used.
- The permission required to set `topic` via `CHANMETA SET` is identical to the permission
  required to change the topic via the TOPIC command under the channel's current mode.

No other keys are reserved by this specification. Future extensions MAY define additional
reserved keys.

## Commands

### CHANMETA SET

```
CHANMETA <channel> SET <key> <type> :<value>
```

Creates or updates a metadata key on the specified channel. If the key does not yet exist,
it is created with the given type. If the key already exists, the type argument MUST match
the previously declared type; a mismatch MUST result in ERR_CHANMETABADTYPE.

The value is provided as a trailing parameter. For short-form types this MUST be a single
trailing parameter. For long-form (`text`) values, see Long-Form Values.

On success the server MUST broadcast a `CHANMETA SET` message to all channel members that
have negotiated this capability:

```
:setter!user@host CHANMETA #channel SET <key> <type> :<value>
```

Members that have not negotiated the capability receive no notification, with the exception
of the `topic` key (see Reserved Keys).

### CHANMETA GET

```
CHANMETA <channel> GET <key>
```

Requests the current value of a single metadata key. The server replies with
`RPL_CHANMETAENTRY`, followed by `RPL_CHANMETAEND`.

### CHANMETA LIST

```
CHANMETA <channel> LIST
```

Requests all metadata keys on the channel. The server replies with zero or more
`RPL_CHANMETAENTRY` numerics, followed by `RPL_CHANMETAEND`.

### CHANMETA DEL

```
CHANMETA <channel> DEL <key>
```

Deletes a metadata key. Reserved keys MUST NOT be deleted; attempts MUST result in
`ERR_CHANMETAREADONLY`.

On success the server MUST broadcast a `CHANMETA DEL` message to all channel members that
have negotiated this capability:

```
:setter!user@host CHANMETA #channel DEL <key>
```

## Server Numerics

| Symbolic name          | Description                                              |
|------------------------|----------------------------------------------------------|
| RPL_CHANMETAENTRY      | A single key-value metadata entry (see below)            |
| RPL_CHANMETAEND        | Marks the end of a GET or LIST response                  |
| ERR_CHANMETABADTYPE    | The supplied type does not match the key's declared type |
| ERR_CHANMETABADVALUE   | The value does not conform to the declared type          |
| ERR_CHANMETAREADONLY   | The key is reserved and cannot be set or deleted         |
| ERR_CHANMETAUNKNOWN    | The requested key does not exist                         |
| ERR_CHANMETAFULL       | The channel has reached the CHANMETAKEYS limit           |
| ERR_CHANMETANOPERM     | The client lacks permission to perform this operation    |

RPL_CHANMETAENTRY has the following format:

```
:server RPL_CHANMETAENTRY <nick> <channel> <key> <type> :<value>
```

For long-form values, `RPL_CHANMETAENTRY` MUST be preceded by a `rsr.chat/chanmeta-batch`
batch opening (see Long-Form Values).

`RPL_CHANMETAEND` has the following format:

```
:server RPL_CHANMETAEND <nick> <channel> :End of channel metadata
```

## Long-Form Values

Long-form values use the `text` type and may contain embedded newlines or exceed
`CHANMETALEN` bytes. Servers that advertise `CHANMETALONGLEN` in ISUPPORT MUST support
this mechanism. Servers that do not advertise `CHANMETALONGLEN` MUST reject any attempt
to set a `text`-typed key with `ERR_CHANMETABADTYPE`.

### Setting a long-form value

Clients signal a long-form value by opening a `rsr.chat/chanmeta-batch` batch:

```
BATCH +<ref> rsr.chat/chanmeta-batch <channel> SET <key> text
@batch=<ref> CHANMETABODY :<line one>
@batch=<ref> CHANMETABODY :<line two>
BATCH -<ref>
```

Each CHANMETABODY line contributes one UTF-8 line to the value. Newlines are inserted
between consecutive CHANMETABODY payloads. The total assembled byte length MUST NOT
exceed `CHANMETALONGLEN`; the server MUST reject the batch with `ERR_CHANMETABADVALUE` if
this limit is exceeded.

### Receiving a long-form value

When a server sends a long-form value in response to `CHANMETA GET` or `CHANMETA LIST`, it
MUST wrap the `RPL_CHANMETAENTRY` in a `rsr.chat/chanmeta-batch` batch:

```
:server BATCH +<ref> rsr.chat/chanmeta-batch <channel> GET <key> text
:server @batch=<ref> RPL_CHANMETAENTRY <nick> <channel> <key> text :<line one>
:server @batch=<ref> RPL_CHANMETAENTRY <nick> <channel> <key> text :<line two>
:server BATCH -<ref>
```

Each `RPL_CHANMETAENTRY` in the batch carries one line of the value. Clients MUST
concatenate lines with a newline character to reconstruct the full value.

If a client has not negotiated IRCv3 batch support, servers MUST omit long-form values
from `CHANMETA LIST` responses entirely and MUST reply to `CHANMETA GET` for a `text` key
with `ERR_CHANMETABADTYPE`.

## Default Permissions

Unless overridden by `rsr.chat/rbac` or `rsr.chat/advanced-moderation`, the following
default permission rules apply:

- Any channel member MAY call `CHANMETA GET` and `CHANMETA LIST`.
- Channel operators (mode `+o`) MAY call `CHANMETA SET` and `CHANMETA DEL` for any
  non-reserved key.
- Setting the `topic` key follows the same rules as the TOPIC command under the channel's
  current mode.
- Server operators MAY call `CHANMETA SET` and `CHANMETA DEL` on any channel for any key.

## Guild Integration

When `rsr.chat/guild` is negotiated, a guild operator MAY set metadata defaults for all
channels within a guild using the guild-prefix scope syntax defined by the guild
specification. A guild-level default is overridden by any channel-specific value for the
same key. Guild-level defaults MUST appear in `CHANMETA LIST` responses with a `default`
flag appended to the `RPL_CHANMETAENTRY` line:

```
:server RPL_CHANMETAENTRY <nick> <channel> <key> <type> :<value> (default)
```

The parenthetical `(default)` is a server-supplied annotation token. Clients SHOULD
display or expose this distinction to the user.

## Category Integration

When `rsr.chat/channel-categories` is negotiated, metadata defaults MAY be set at the
category scope using the category scope identifier syntax defined by that specification
(`#category/`). Category-scoped defaults follow the same precedence rules as described
in that specification:

- Channel-specific values take precedence over category-scoped defaults.
- Category-scoped defaults take precedence over guild-level defaults.
- Guild-level defaults take precedence over server-wide defaults.

A category-scoped default is also surfaced in `CHANMETA LIST` responses with a `default`
annotation (see Guild Integration above).

## RBAC Integration

When `rsr.chat/rbac` is negotiated, per-key permissions MAY be defined using the RBAC
rule syntax defined by that specification. The following permission identifiers are
defined by this extension for use in RBAC rules:

| Permission identifier       | Description                               |
|-----------------------------|-------------------------------------------|
| `chanmeta.get`              | Read any metadata key                     |
| `chanmeta.set.<key>`        | Set the named key                         |
| `chanmeta.del.<key>`        | Delete the named key                      |
| `chanmeta.set.*`            | Set any non-reserved metadata key         |
| `chanmeta.del.*`            | Delete any non-reserved metadata key      |

RBAC rules for metadata permissions MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`. A scoped rule applies to
all matching channels as described in that specification.

Example (illustrative; normative RBAC syntax defined in rsr.chat/rbac):

```
RBACSET #engineering/ ROLE moderator PERMISSION chanmeta.set.topic
RBACSET #acmecorp/ ROLE admin PERMISSION chanmeta.set.*
```

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, metadata keys MAY be used as criteria
in moderation rules. The normative syntax for such rules is defined by
`rsr.chat/advanced-moderation`. This specification defines the following metadata
predicates for use in moderation rule conditions:

| Predicate                           | Matches when                                |
|-------------------------------------|---------------------------------------------|
| `chanmeta:<key>=<value>`            | The key exists and equals the given value   |
| `chanmeta:<key>~=<value>`           | The key exists and contains the given value |
| `chanmeta:<key> exists`             | The key exists regardless of value          |
| `chanmeta:<key> missing`            | The key does not exist                      |

These predicates MAY be used to restrict joins, posting, or other actions based on channel
metadata state.

## Interaction with Non-Negotiating Clients

Clients that have not negotiated `rsr.chat/channel-metadata` receive no CHANMETA
notifications. They interact with the `topic` reserved key solely through the standard
TOPIC command and RPL_TOPIC (332) numeric, which the server MUST continue to send
normally. All other metadata is invisible to such clients.

## Examples

### Setting and retrieving a short-form key

```
C: CHANMETA #engineering/general SET lang string :en-GB
S: :alice!alice@host CHANMETA #engineering/general SET lang string :en-GB
    (broadcast to all members that negotiated the capability)

C: CHANMETA #engineering/general GET lang
S: :server RPL_CHANMETAENTRY alice #engineering/general lang string :en-GB
S: :server RPL_CHANMETAEND alice #engineering/general :End of channel metadata
```

### Setting the topic via CHANMETA and observing TOPIC broadcast

```
C: CHANMETA #engineering/general SET topic text
    (long-form via batch, see Long-Form Values section)
S: :alice!alice@host TOPIC #engineering/general :This channel is for general engineering discussion.
    (broadcast to all members including non-negotiating clients)
S: :alice!alice@host CHANMETA #engineering/general SET topic text :(same value, to negotiating clients)
```

### Type mismatch rejection

```
C: CHANMETA #engineering/general SET lang int :42
S: :server ERR_CHANMETABADTYPE alice #engineering/general lang :Key type is string, not int
```

### Exceeding key limit

```
C: CHANMETA #engineering/general SET newkey string :value
S: :server ERR_CHANMETAFULL alice #engineering/general :Channel metadata key limit reached
```

### Listing all metadata on a channel

```
C: CHANMETA #engineering/general LIST
S: :server RPL_CHANMETAENTRY alice #engineering/general lang string :en-GB
S: :server RPL_CHANMETAENTRY alice #engineering/general color color :#3498db
S: :server RPL_CHANMETAENTRY alice #engineering/general topic text :(default)
S: :server RPL_CHANMETAEND alice #engineering/general :End of channel metadata
```

### Setting a long-form text value

```
C: BATCH +meta1 rsr.chat/chanmeta-batch #engineering/general SET description text
C: @batch=meta1 CHANMETABODY :This channel is for day-to-day engineering discussion.
C: @batch=meta1 CHANMETABODY :Please keep discussion on-topic.
C: BATCH -meta1
S: :alice!alice@host CHANMETA #engineering/general SET description text
    :This channel is for day-to-day engineering discussion.\nPlease keep discussion on-topic.
    (broadcast to negotiating members)
```

### Deleting a key

```
C: CHANMETA #engineering/general DEL lang
S: :alice!alice@host CHANMETA #engineering/general DEL lang
```

### Attempting to delete the reserved topic key

```
C: CHANMETA #engineering/general DEL topic
S: :server ERR_CHANMETAREADONLY alice #engineering/general topic :This key is read-only
```