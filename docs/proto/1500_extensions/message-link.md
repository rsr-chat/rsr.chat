---
title: rsr.chat/message-link
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/message-link

This extension assigns a unique server-generated ID tag to every message and provides an
inline syntax by which clients may reference a prior message within a later one. Servers
resolve inline link references and attach structured tags so that receiving clients can
display contextual previews or threading UI without querying history separately. The
mechanism integrates with message history replay, channel metadata, and the access control
extensions.

## Motivation

Threaded or quoted replies are a fundamental feature of modern messaging platforms. IRC
has historically had no standard mechanism for this: clients either paste truncated text
quotes manually or rely on proprietary conventions. This extension provides a
server-authoritative, ID-based reference system that is compact on the wire, survives
history replay faithfully, and degrades gracefully for clients that have not negotiated
the capability.

## Capability

This extension is advertised as the `rsr.chat/message-link` capability during CAP
negotiation:

```
CAP * LS :rsr.chat/message-link
```
No capability value is used. Supplementary limits are communicated via ISUPPORT (see
below).

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `MSGLINKMAX=<n>` — the maximum number of inline message references permitted in a
  single outbound message. Servers MUST NOT enforce a value lower than 1.
- `MSGLINKLEN=<n>` — the byte length of server-generated message IDs. Clients MUST treat
  IDs as opaque byte strings of exactly this length and MUST NOT assume any internal
  structure.

Example:

```
:server 005 nick MSGLINKMAX=5 MSGLINKLEN=26 :are supported by this server
```

If these values change after a configuration reload, a fresh `RPL_ISUPPORT` MUST be sent
to all connected clients.

## Message ID Tags

When this capability is active, the server MUST attach a `MSGLINK` message tag to
every `PRIVMSG` and `NOTICE` it delivers to any client that has negotiated this capability.
The tag value is a server-generated, globally unique, URL-safe base64 string of exactly
`MSGLINKLEN` bytes. Message IDs MUST be unique within the lifetime of the server instance
and SHOULD be unique across restarts.

The tag is attached by the server and MUST NOT be accepted from clients. If a client
sends a message that includes a `MSGLINK` tag, the server MUST strip it before
relay.

Example message as received by a negotiating client:

```
@MSGLINK=01HQ7K2N4R8VXJD3MCWBPZT56Y :alice!alice@host PRIVMSG #engineering/general :hello
```

Clients that have not negotiated this capability receive messages without the tag. The
tag MUST be preserved in message history storage and replayed verbatim during history
retrieval (see Message History).

## Inline Link Syntax

A message may reference a prior message by embedding a link token anywhere in its
plaintext body. The link token syntax is:

```
>>MSGLINK
```

Where `MSGLINK` is a message ID of exactly `MSGLINKLEN` characters drawn from the URL-safe
base64 alphabet (`A-Z`, `a-z`, `0-9`, `-`, `_`). The `>>` sigil introduces the reference.
The token ends at the first character that is not a valid message ID character or at end
of string.

Multiple references MAY appear in a single message body up to the `MSGLINKMAX` limit.
References MAY appear at any position in the body; they are not required to be at the
start of the line.

A reference in a `NOTICE` MUST NOT be resolved or annotated by the server.

### Example body with a reference

```
>>01HQ7K2N4R8VXJD3MCWBPZT56Y I agree with this point entirely.
```

## Client Rendering

Clients that have negotiated this capability SHOULD NOT render any contextual quote or 
preview beside or above the referencing message. Clients SHOULD render the link in the
style of a hyperlink `[>#channel]`, but the specific rendering is implementation-defined.

Clients that have not negotiated this capability receive messages with `>>MSGLINK` tokens
as plain text and receive no preview tags. This is the correct degraded behaviour and
requires no special handling by the server.

## Unresolvable and Expired References

Servers are not required to retain messages indefinitely. A reference to a message whose
ID is no longer in the server's history store is silently unresolvable. Clients SHOULD NOT
assume that an unresolved reference implies the referenced message never existed.

## Message History

Servers that support message history replay (via `CHATHISTORY` or an equivalent mechanism)
MUST include the `MSGLINK` tag on replayed messages exactly as it was assigned at
original delivery time. Tag values MUST NOT be regenerated on replay.

If a client requests history for a channel or query target, the server MUST include message
link tags in the replayed batch in the same form they were delivered originally. Clients
that have not negotiated this capability receive history without these tags.

## Cross-Channel References

A client MAY reference a message from a different channel.

## Examples

### Capability negotiation

```
C: CAP LS 302
S: CAP * LS :rsr.chat/message-link
C: CAP REQ :rsr.chat/message-link
S: CAP * ACK :rsr.chat/message-link
C: NICK alice
C: USER alice 0 * :Alice
S: 005 alice MSGLINKMAX=5 MSGLINKLEN=26 :are supported by this server
S: 001 alice :Welcome to the network
```

### Messages delivered with ID tags

```
:bob!bob@host PRIVMSG #engineering/general :The deploy pipeline is broken again.
(alice receives this as:)
@rsr.chat/MSGLINK=01HQ7K2N4R8VXJD3MCWBPZT56Y :bob!bob@host PRIVMSG #engineering/general :The deploy pipeline is broken again.
```

### Sending a message that references a prior one

```
C: PRIVMSG #engineering/general :>>01HQ7K2N4R8VXJD3MCWBPZT56Y I've filed a ticket for this.

(Other channel members that negotiated the capability receive:)
@rsr.chat/MSGLINK=01HQAB3T2F7PPXC4NK8RWLM09S :alice!alice@host PRIVMSG #engineering/general :>>01HQ7K2N4R8VXJD3MCWBPZT56Y I've filed a ticket for this.

(Channel members that did not negotiate the capability receive:)
:alice!alice@host PRIVMSG #engineering/general :>>01HQ7K2N4R8VXJD3MCWBPZT56Y I've filed a ticket for this.
```

### Exceeding MSGLINKMAX

(Server has MSGLINKMAX=2)

```
C: PRIVMSG #engineering/general :>>ID1 >>ID2 >>ID3 too many refs.
S: :server ERR_CANNOTSENDTOCHAN alice #engineering/general :Too many message link references
```