---
title: rsr.chat/rbac
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/rbac

This extension provides a general-purpose role-based access control system for IRC
channels, channel categories, and guilds. It defines a rule language for granting and
denying named permissions to roles and accounts, a command surface for managing rules,
and the evaluation model by which the server resolves permissions at runtime. All
permission identifiers referenced by other extensions in this suite are governed by
the evaluation model defined here.

## Motivation

Standard IRC access control is limited to a small set of hardcoded channel modes (`+o`,
`+v`, `+b`, and so on) with no extensibility, no inheritance, no scoping across channel
hierarchies, and no support for fine-grained per-feature permissions. The extensions in
this suite require a richer model: permissions scoped to categories, guilds, and
individual channels; custom roles beyond operator and voice; per-account overrides;
and integration with identity mechanisms such as `rsr.chat/did-sasl`. This extension
provides that model.

## Capability

This extension is advertised as the `rsr.chat/rbac` capability during CAP negotiation:

```
CAP * LS :rsr.chat/rbac
```

No capability value is used. Limits are communicated via ISUPPORT (see below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `RBACRULES=<n>` — the maximum number of RBAC rules permitted per scope target
  (channel, category, or guild). A value of `0` indicates no server-imposed limit.
  is negotiated this list MUST match the `MEMBERROLES` token defined by that extension.
- `RBACMAXDEPTH=<n>` — the maximum rule inheritance depth when resolving scoped rules
  through category and guild hierarchies. Servers MUST NOT enforce a value lower than 3.

Example:

```
:server 005 nick RBACRULES=256 RBACMAXDEPTH=5 :are supported by this server
```

If any of these values change after a configuration reload a fresh RPL_ISUPPORT MUST be
sent to all connected clients.

## Terminology

- **Permission**: a named capability string identifying a specific action or access right.
  Permission names are defined by individual extensions and listed in their specifications.
  Examples: `chanmeta.set.topic`, `reaction.add`, `emote.use.animated`.
- **Role**: a named rank. The built-in roles are those listed in `RBACROLES`. Custom roles
  MAY be defined within a scope (see Custom Roles).
- **Subject**: the entity to which a rule applies. A subject is one of:
  - A role name, e.g. `op` — applies to all users holding that role.
  - An account name, e.g. `account:alice` — applies to a specific authenticated account.
  - A DID, e.g. `did:did:web:alice.example.com` — applies to any account authenticated
    via that DID. Only valid when `rsr.chat/did-sasl` is negotiated.
  - The default target `*` which applies universally to all clients at the lowest
    priority (always overridden).
  - The wildcard `authenticated` — applies to all users identified to an account.
- **Effect**: `allow`, `deny`.
- **Rule**: a triple of (subject, permission, effect) associated with a scope target.
- **Scope target**: the entity to which a rule is attached — a channel, a category
  prefix, a guild, or the server globally.
- **Scope chain**: the ordered list of scope targets consulted during rule evaluation for
  a given channel, from most specific to most general (see Evaluation Model).
- **Inheritance**: the propagation of rules from a broader scope target to a narrower one
  unless overridden.

## Permission Identifier Syntax

Permission identifiers are dot-delimited ASCII strings matching the pattern
`[a-z0-9][a-z0-9_-]*(\.[a-z0-9*][a-z0-9_*-]*)*`. The wildcard component `*` may appear
as the final segment only and matches any single segment. For example `chanmeta.set.*`
matches `chanmeta.set.topic` and `chanmeta.set.lang` but not `chanmeta.get`.

Servers MUST expand wildcard permission rules at evaluation time and MUST NOT store them
as literal string matches.

## Built-in Roles and Default Privilege Mapping

The built-in roles map to IRC modes and carry a default set of permissions unless
overridden by explicit rules. The mapping is:

| Role     | IRC mode equivalent              | Default implied permissions                           |
|----------|----------------------------------|-------------------------------------------------------|
| `owner`  | `+q` or `+o` where +q absent     | All permissions in all scopes they own                |
| `admin`  | `+a` or `+o` where +a absent     | All permissions except those reserved to `owner`      |
| `op`     | `+o`                             | Moderation, metadata, membership management           |
| `voice`  | `+v`                             | Speak in moderated channels; limited metadata read    |
| `member` | (none)                           | Join, read, post, react, emote (subject to channel rules) |

The default implied permissions for each role are defined by the default permission tables
in each extension specification. Explicit RBAC rules ALWAYS take precedence over implied
defaults.

## Custom Roles

Roles beyond the built-in set MAY be defined within a scope via `RBACROLE CREATE`. Custom
roles have no IRC mode equivalent and confer no automatic mode application. They exist
solely for RBAC rule targeting.

Custom roles are ordered relative to the built-in roles at creation time. A custom role's
position in the precedence order determines which other roles may manage members holding
it: a user may only assign or remove a role that is lower in precedence than their own.

### RBACROLE CREATE

```
RBACROLE <scope> CREATE <rolename> AFTER <existing-role>
```

Creates a new role within the given scope, inserting it immediately after `<existing-role>`
in the precedence order. `<rolename>` MUST match `[A-Za-z0-9][A-Za-z0-9_-]*` and MUST
NOT conflict with any built-in role name.

### RBACROLE DELETE

```
RBACROLE <scope> DELETE <rolename>
```

Deletes a custom role. Any rules targeting this role are also deleted. Any members
assigned this role (when `rsr.chat/channel-membership` is negotiated) revert to `member`.

### RBACROLE LIST

```
RBACROLE <scope> LIST
```

Lists all roles (built-in and custom) within the scope, in precedence order. The server
responds with one or more `RPL_RBACROLEENTRY` numerics followed by `RPL_RBACEND`.

## Scope Targets

Every rule is attached to a scope target. Scope targets are expressed as follows:

| Scope target      | Syntax                  | Example                    |
|-------------------|-------------------------|----------------------------|
| Server-global     | `*`.                    | `*`                        |
| Guild             | `guild:<guildname>`     | `guild:acmecorp`           |
| Category          | `#<category>/`          | `#engineering/`            |
| Guild + Category  | `#<guild>/<category>/`  | `#acmecorp/engineering/`   |
| Channel           | `#<channel>`            | `#engineering/general`     |

The trailing slash on category targets is mandatory and distinguishes them from channel
targets, consistent with the scoping syntax defined in `rsr.chat/channel-categories`.

Server-global rules apply to all channels on the server. Guild rules apply to all
channels within the guild. Category rules apply to all channels whose name begins with
the category prefix. Channel rules apply to a single channel only.

## Scope Chain and Evaluation Model

When the server evaluates whether a subject holds a given permission in a specific channel,
it constructs the scope chain for that channel and evaluates rules from most specific to
most general, stopping at the first matching rule.

The scope chain for a channel `#guild/category/leaf` when both `rsr.chat/guild` and
`rsr.chat/channel-categories` are negotiated is:

1. Channel: `#guild/category/leaf`
2. Guild-scoped category: `#guild/category/`
3. Category (non-guild): `#category/` (if applicable)
4. Guild: `guild:<guildname>`
5. Server-global: `*`

For a channel `#category/leaf` without guild support:

1. Channel: `#category/leaf`
2. Category: `#category/`
3. Server-global: `*`

For a plain channel `#channel`:

1. Channel: `#channel`
2. Server-global: `*`

Within each scope target, rules are evaluated in subject specificity order:

1. Account-specific rule (`account:<name>` or `did:<did>`) — most specific.
2. Role-specific rule for the subject's exact role.
3. Role-specific rule for each role higher in the precedence order than the subject's
   role, from lowest to highest above them. This provides implicit inheritance: an `op`
   inherits permissions granted to `voice` and `member` unless explicitly denied.
4. Wildcard `authenticated` rule (if subject is identified to an account).
5. Wildcard `*` rule — least specific.

The first rule encountered that matches the subject and names the requested permission
determines the outcome. If the rule's effect is `allow`, the permission is granted. If
`deny`, the permission is denied. If no rule matches anywhere in the scope chain, the
server falls back to the extension-defined defaults for the subject's role.

### Deny Precedence

An explicit `deny` at a more specific scope ALWAYS overrides an `allow` at a less
specific scope, consistent with the most-specific-first evaluation order. An explicit
`allow` at a more specific scope ALWAYS overrides a `deny` at a less specific scope.
This allows guild-level `deny` rules to be lifted by channel-level `allow` rules and vice
versa.

### Short-circuit Evaluation

Once a matching rule is found at any step of the scope chain, evaluation stops. The
server MUST NOT continue evaluating less specific scopes after a match is found, even if
those scopes contain conflicting rules.

## Commands

### RBACSET

```
RBACSET <scope> <subject> <permission> <effect>
```

Creates or replaces an RBAC rule. If a rule already exists for the given (scope, subject,
permission) triple it is overwritten.

- `<scope>` — a scope target as defined in Scope Targets.
- `<subject>` — a role name, `account:<name>`, `did:<did>`, `authenticated`, or `*`.
- `<permission>` — a permission identifier, optionally with a wildcard final segment.
- `<effect>` — `allow` or `deny`.

On success the server MUST broadcast an `RBACSET` notification to all active members of
the affected scope that have negotiated this capability:

```
:setter!user@host RBACSET <scope> <subject> <permission> <effect>
```

### RBACDEL

```
RBACDEL <scope> <subject> <permission>
```

Removes an existing RBAC rule. If no matching rule exists the server MUST reply with
ERR_RBACUNKNOWNRULE.

On success the server MUST broadcast an `RBACDEL` notification to all active members of
the affected scope that have negotiated this capability:

```
:setter!user@host RBACDEL <scope> <subject> <permission>
```

### RBACLIST

```
RBACLIST <scope>
```

Requests all RBAC rules for the given scope target. The server responds with zero or more
RPL_RBACENTRY numerics followed by RPL_RBACEND. If IRCv3 `batch` is negotiated the
entire response MUST be wrapped in a `rsr.chat/rbaclist` batch.

### RBACCHECK

```
RBACCHECK <scope> <subject> <permission>
```

Performs a dry-run evaluation of whether the given subject holds the given permission in
the given scope, following the full scope chain evaluation defined in the Evaluation
Model. The server responds with RPL_RBACALLOW or RPL_RBACDENY, and MUST include the
scope target and rule that produced the outcome:

```
:server RPL_RBACALLOW <nick> <scope> <subject> <permission> :<matched-scope> <matched-subject> <matched-permission>
:server RPL_RBACDENY  <nick> <scope> <subject> <permission> :<matched-scope> <matched-subject> <matched-permission>
```

If the outcome is determined by the extension default rather than an explicit rule,
`matched-scope` MUST be `default` and `matched-subject` and `matched-permission` MUST
reflect the built-in defaults.

RBACCHECK is intended for operators and client developers. Servers MAY rate-limit it.

### RBACWHO

```
RBACWHO <scope> <permission>
```

Lists all subjects that have an explicit rule (of either effect) for the given permission
in the given scope. This does not include subjects that hold the permission by default.
The server responds with zero or more RPL_RBACWHOENTRY numerics followed by RPL_RBACEND.

```
:server RPL_RBACWHOENTRY <nick> <scope> <permission> <subject> <effect>
```

## Server Numerics

| Symbolic name            | Description                                                        |
|--------------------------|--------------------------------------------------------------------|
| `RPL_RBACENTRY`          | A single RBAC rule entry                                           |
| `RPL_RBACROLEENTRY`      | A single role entry in a RBACROLE LIST response                    |
| `RPL_RBACWHOENTRY`       | A single subject entry in a RBACWHO response                       |
| `RPL_RBACALLOW`          | RBACCHECK result: permission is granted                            |
| `RPL_RBACDENY`           | RBACCHECK result: permission is denied                             |
| `RPL_RBACEND`            | Marks the end of any multi-entry RBAC response                     |
| `ERR_RBACNOPERM`         | The client lacks permission to manage rules in this scope          |
| `ERR_RBACUNKNOWNRULE`    | The targeted rule does not exist                                   |
| `ERR_RBACUNKNOWNSCOPE`   | The specified scope target does not exist                          |
| `ERR_RBACUNKNOWNSUBJECT` | The specified subject (account or role) does not exist             |
| `ERR_RBACINVALIDPERM`    | The permission identifier is syntactically invalid                 |
| `ERR_RBACRULEFULL`       | The scope has reached the RBACRULES limit                          |
| `ERR_RBACROLEFULL`       | The scope has reached the maximum custom role limit                |
| `ERR_RBACROLEEXISTS`     | A role with this name already exists in the scope                  |
| `ERR_RBACROLEINVAL`      | The role name is syntactically invalid or conflicts with a built-in|

RPL_RBACENTRY has the following format:

```
:server RPL_RBACENTRY <nick> <scope> <subject> <permission> <effect> <set-by> <set-at>
```

Where `set-by` is the account name of the user who created or last modified the rule and
`set-at` is an ISO 8601 UTC timestamp.

`RPL_RBACROLEENTRY` has the following format:

```
:server RPL_RBACROLEENTRY <nick> <scope> <rolename> <precedence-index> <type> <created-by> <created-at>
```

Where `type` is `builtin` or `custom`.

## Who May Manage Rules

### Channel rules

A user may manage RBAC rules for a channel if:
- They hold role `op` or higher in that channel, OR
- They are a server operator, OR
- They hold a role for which `rbac.manage` has been explicitly granted via a higher-scope
  rule.

### Category rules

A user may manage RBAC rules for a category if:
- They hold role `admin` or higher in all channels within the category, OR
- They are a server operator, OR
- They hold `rbac.manage` for the category scope via a guild or server-global rule.

### Guild rules

A user may manage RBAC rules for a guild if they are a guild operator or a server
operator.

### Server-global rules

Only server operators may manage server-global RBAC rules.

### Rule management permission

A user MUST NOT be permitted to grant a permission they do not themselves hold, nor to
grant a role higher than their own. A rule that would elevate another user above the
rule-setter's own effective permissions MUST be rejected with ERR_RBACNOPERM.

## Consolidated Permission Registry

The following table lists all permission identifiers defined across the extension suite,
their defining extension, and a brief description. This table is informative; the
normative definition of each permission's semantics is in the originating extension.

### Core channel permissions

| Permission                  | Extension              | Description                                          |
|-----------------------------|------------------------|------------------------------------------------------|
| `rbac.manage`               | rsr.chat/rbac          | Manage RBAC rules in this scope                      |
| `rbac.check`                | rsr.chat/rbac          | Use RBACCHECK in this scope                          |
| `rbac.role.manage`          | rsr.chat/rbac          | Create and delete custom roles in this scope         |

### Channel metadata permissions

| Permission                  | Extension                    | Description                                     |
|-----------------------------|------------------------------|-------------------------------------------------|
| `chanmeta.get`              | rsr.chat/channel-metadata    | Read any metadata key                           |
| `chanmeta.set.<key>`        | rsr.chat/channel-metadata    | Set the named key                               |
| `chanmeta.del.<key>`        | rsr.chat/channel-metadata    | Delete the named key                            |
| `chanmeta.set.*`            | rsr.chat/channel-metadata    | Set any non-reserved metadata key               |
| `chanmeta.del.*`            | rsr.chat/channel-metadata    | Delete any non-reserved metadata key            |

### Channel membership permissions

| Permission                  | Extension                    | Description                                     |
|-----------------------------|------------------------------|-------------------------------------------------|
| `membership.list`           | rsr.chat/channel-membership  | View the persistent membership list             |
| `membership.add`            | rsr.chat/channel-membership  | Add a new persistent member                     |
| `membership.remove`         | rsr.chat/channel-membership  | Remove a persistent member                      |
| `membership.setrole`        | rsr.chat/channel-membership  | Change a member's role                          |
| `membership.invite`         | rsr.chat/channel-membership  | Issue a one-time session invite                 |
| `membership.self.add`       | rsr.chat/channel-membership  | Add oneself to a channel's membership list      |
| `membership.self.remove`    | rsr.chat/channel-membership  | Remove oneself from a channel's membership list |

### Reaction permissions

| Permission                  | Extension              | Description                                          |
|-----------------------------|------------------------|------------------------------------------------------|
| `reaction.add`              | rsr.chat/reactions     | Place a reaction on a message                        |
| `reaction.remove.own`       | rsr.chat/reactions     | Remove one's own reaction                            |
| `reaction.remove.any`       | rsr.chat/reactions     | Remove any user's reaction                           |
| `reaction.list`             | rsr.chat/reactions     | Call REACTLIST                                       |
| `reaction.configure`        | rsr.chat/reactions     | Set the reactions-enabled metadata key               |
| `reaction.custom`           | rsr.chat/reactions     | Place reactions using custom emotes                  |

### Emote permissions

| Permission                  | Extension              | Description                                          |
|-----------------------------|------------------------|------------------------------------------------------|
| `emote.add`                 | rsr.chat/emote         | Register a new emote in this scope                   |
| `emote.remove`              | rsr.chat/emote         | Remove an emote from this scope                      |
| `emote.rename`              | rsr.chat/emote         | Rename an emote in this scope                        |
| `emote.list`                | rsr.chat/emote         | List emotes in this scope                            |
| `emote.use`                 | rsr.chat/emote         | Use emotes from this scope in messages               |
| `emote.use.animated`        | rsr.chat/emote         | Use animated emotes from this scope                  |
| `emote.pack.manage`         | rsr.chat/emote         | Create, delete, and modify emote packs               |
| `emote.configure`           | rsr.chat/emote         | Set the emotes-enabled metadata key                  |

### Message link permissions

| Permission                  | Extension              | Description                                          |
|-----------------------------|------------------------|------------------------------------------------------|
| `msglink.resolve`           | rsr.chat/message-link  | Have inline references resolved by the server        |
| `msglink.crosschannel`      | rsr.chat/message-link  | Reference messages from other channels               |

### Typing notification permissions

| Permission                  | Extension              | Description                                          |
|-----------------------------|------------------------|------------------------------------------------------|
| `typing.send`               | rsr.chat/typing-notify | Send typing notifications to this target             |
| `typing.receive`            | rsr.chat/typing-notify | Receive typing notifications from this target        |

### DID authentication permissions

| Permission                  | Extension              | Description                                          |
|-----------------------------|------------------------|------------------------------------------------------|
| `did.auth.require`          | rsr.chat/sasl-did      | Only DID-authenticated clients may join or post      |
| `did.auth.method=did:web`   | rsr.chat/sasl-did      | Only did:web clients permitted                       |
| `did.auth.method=did:plc`   | rsr.chat/sasl-did      | Only did:plc clients permitted                       |
| `did.auth.flow=direct`      | rsr.chat/sasl-did      | Only direct-flow clients permitted                   |
| `did.auth.flow=delegate`    | rsr.chat/sasl-did      | Only delegate-flow clients permitted                 |

## Conflict Resolution Between Scope Levels

The scope chain evaluation defined in the Evaluation Model handles the general case.
Two specific scenarios warrant additional clarification.

### Allow at channel overriding deny at category

A guild administrator may set a `deny` rule at the category level while a channel
operator sets an `allow` rule at the channel level for the same permission. Because
channel scope is more specific than category scope and is evaluated first, the
channel-level `allow` takes effect for that channel only. This is the intended behaviour
and requires no special handling.

### Deny at guild overriding allow at channel

A guild operator may set a `deny` rule at the guild level to impose a floor restriction
across all guild channels. A channel operator may attempt to override this with an
`allow` at the channel level. Because channel scope is evaluated before guild scope the
channel-level `allow` takes effect. If the guild operator wishes their restriction to be
non-overridable, they MUST set it at the channel level as well, or they MUST restrict
the channel operator's `rbac.manage` permission at the guild or category level so that
the channel operator cannot set rules that contradict guild policy.

Servers do not enforce policy hierarchy automatically beyond the scope chain order.
Policy enforcement across scopes is the responsibility of administrators configuring
the rules.

## Interaction with rsr.chat/channel-membership Roles

When `rsr.chat/channel-membership` is negotiated, the roles in `RBACROLES` and
`MEMBERROLES` MUST be identical and in the same order. A member's RBAC role is
their channel membership role. RBAC rules targeting `op` apply to all persistent
members with role `op`. Mode-based roles for transient members (users joined but not
persistent members) are inferred from their current IRC channel mode in the absence of
a membership record:

| IRC mode | Inferred role |
|----------|---------------|
| `+q`     | `owner`       |
| `+a`     | `admin`       |
| `+o`     | `op`          |
| `+v`     | `voice`       |
| (none)   | `member`      |

When `rsr.chat/channel-membership` is not negotiated, role inference from IRC mode is
always used.

## Interaction with rsr.chat/advanced-moderation

When `rsr.chat/advanced-moderation` is negotiated, RBAC permission checks MAY be used
as conditions in moderation rules. The following predicates are defined by this extension
for use in moderation rule conditions:

| Predicate                            | Matches when                                          |
|--------------------------------------|-------------------------------------------------------|
| `rbac:has-perm=<permission>`         | The acting user holds the named permission            |
| `rbac:lacks-perm=<permission>`       | The acting user does not hold the named permission    |
| `rbac:role=<role>`                   | The acting user holds exactly the named role          |
| `rbac:role>=<role>`                  | The acting user's role is at least as high as named   |
| `rbac:role<=<role>`                  | The acting user's role is no higher than named        |
| `rbac:subject=account:<name>`        | The acting user is identified as the named account    |
| `rbac:subject=did:<did>`             | The acting user is authenticated as the named DID     |

## Interaction with Non-Negotiating Clients

Non-negotiating clients are subject to RBAC rules; the server enforces permission checks
regardless of whether the client has negotiated the capability. Non-negotiating clients
simply do not receive RBAC event notifications and cannot use RBACSET, RBACDEL,
RBACLIST, RBACCHECK, or RBACWHO. These commands MUST be rejected with
ERR_UNKNOWNCOMMAND for non-negotiating clients.

Permission denials experienced by non-negotiating clients are surfaced through standard
IRC error numerics (ERR_CHANOPRIVSNEEDED, ERR_CANNOTSENDTOCHAN, etc.) as they would be
in a server without this extension.

## Guild Integration

When `rsr.chat/guild` is negotiated, guild operators have implicit `allow` on all
permissions in all channels within their guild, equivalent to a server-global rule at
guild scope. Guild operators MAY set RBAC rules at the guild scope for any permission.

Guild-scope rules are inherited by all channels and categories within the guild via the
scope chain. A guild operator may create a rule that restricts a permission across the
entire guild and that cannot be overridden by channel operators unless the guild operator
explicitly grants channel operators `rbac.manage` for that permission.

## Category Integration

When `rsr.chat/channel-categories` is negotiated, RBACSET, RBACDEL, RBACLIST, and
RBACCHECK all accept category scope targets using the trailing-slash syntax. Category
rules are inherited by all channels within the category and are evaluated between the
channel scope and the guild or server-global scope in the scope chain.

RBACWHO at a category scope returns subjects with explicit rules in that category only
and does not aggregate channel-level rules within the category.

## Examples

### Setting a simple allow rule

    C: RBACSET #engineering/general voice chanmeta.get allow
    S: :alice!alice@host RBACSET #engineering/general voice chanmeta.get allow
       (broadcast to active members that negotiated the capability)

### Setting a category-scoped deny rule

    C: RBACSET #engineering/ member emote.use.animated deny
    S: :alice!alice@host RBACSET #engineering/ member emote.use.animated deny

### Channel-level allow overriding category-level deny

    (member emote.use.animated is denied at #engineering/ category scope)
    C: RBACSET #engineering/design member emote.use.animated allow
    S: :alice!alice@host RBACSET #engineering/design member emote.use.animated allow
    (members in #engineering/design may now use animated emotes despite the category rule)

### Per-account override

    C: RBACSET #engineering/general account:carol reaction.remove.any allow
    S: :alice!alice@host RBACSET #engineering/general account:carol reaction.remove.any allow
    (carol may remove any user's reaction in #engineering/general regardless of her role)

### Wildcard permission grant

    C: RBACSET #engineering/general op chanmeta.set.* allow
    S: :alice!alice@host RBACSET #engineering/general op chanmeta.set.* allow
    (operators may set any non-reserved metadata key)

### DID-scoped permission rule

    C: RBACSET guild:acmecorp did:did:web:alice.example.com rbac.manage allow
    S: :serverop!serverop@host RBACSET guild:acmecorp did:did:web:alice.example.com rbac.manage allow
    (any account authenticated as did:web:alice.example.com may manage RBAC in acmecorp)

### Listing rules for a scope

    C: RBACLIST #engineering/general
    S: :server BATCH +rl rsr.chat/rbaclist #engineering/general
    S: @batch=rl :server RPL_RBACENTRY alice #engineering/general voice chanmeta.get allow serverop 2024-01-10T09:00:00.000Z
    S: @batch=rl :server RPL_RBACENTRY alice #engineering/general account:carol reaction.remove.any allow alice_acct 2024-03-15T14:22:01.000Z
    S: @batch=rl :server RPL_RBACENTRY alice #engineering/general op chanmeta.set.* allow alice_acct 2024-03-15T14:25:00.000Z
    S: :server BATCH -rl
    S: :server RPL_RBACEND alice #engineering/general :End of RBAC rules

### Checking effective permission

    C: RBACCHECK #engineering/general account:bob reaction.add
    S: :server RPL_RBACALLOW alice #engineering/general account:bob reaction.add :#engineering/ member reaction.add
    (bob holds permission via the member role rule at the category scope)

    C: RBACCHECK #engineering/general account:dave emote.use.animated
    S: :server RPL_RBACDENY alice #engineering/general account:dave emote.use.animated :#engineering/ member emote.use.animated
    (dave is denied via the category-scope deny on member)

### Removing a rule

    C: RBACDEL #engineering/general account:carol reaction.remove.any
    S: :alice!alice@host RBACDEL #engineering/general account:carol reaction.remove.any

### Creating and using a custom role

    C: RBACROLE #engineering/ CREATE trusted AFTER voice
    (trusted is now between voice and member in the precedence order)
    C: RBACSET #engineering/ trusted msglink.crosschannel allow
    S: :alice!alice@host RBACSET #engineering/ trusted msglink.crosschannel allow
    (users assigned the trusted role in any #engineering/ channel may post cross-channel links)

### RBACWHO

    C: RBACWHO #engineering/general reaction.remove.any
    S: :server RPL_RBACWHOENTRY alice #engineering/general reaction.remove.any account:carol allow
    S: :server RPL_RBACEND alice #engineering/general :End of RBAC who

### Insufficient permission to set a rule

    (bob has role voice and attempts to grant op-level permissions)
    C: RBACSET #engineering/general member membership.add allow
    S: :server ERR_RBACNOPERM bob #engineering/general :Insufficient permission to manage rules in this scope

### Guild-wide DID requirement

    C: RBACSET guild:acmecorp * did.auth.require allow
    (all unauthenticated users are denied by default; only DID-authenticated users may proceed)
    C: RBACSET guild:acmecorp authenticated did.auth.require allow
    (authenticated but non-DID accounts are separately governed by the did.auth.require permission
     as evaluated by the sasl-did extension)