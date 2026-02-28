---
title: rsr.chat/extended-line-length
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/extended-line-length

This extension allows servers to advertise support for IRC message lines longer than the
512-byte limit defined in RFC 1459, and for clients to opt in to sending and receiving them.

## Motivation

RFC 1459 limits IRC messages to 512 bytes including the trailing CRLF. This constrains
modern use cases such as long-form messaging, rich metadata in message tags, and encoded
content. This extension provides a negotiated, interoperable mechanism for raising that limit.

## Capability

This extension is advertised as the `rsr.chat/extended-line-length` capability during CAP
negotiation. The capability MUST carry a value indicating the maximum total line length in
bytes, including the leading colon (if present), all parameters, and the trailing CRLF:

```
CAP * LS :rsr.chat/extended-line-length=8192
```

The capability value is authoritative. Clients MUST use this value to determine the
maximum permitted line length and MUST NOT rely solely on the ISUPPORT token (see below).

## ISUPPORT Token

For compatibility with clients that do not perform CAP negotiation, servers MUST also
advertise the limit via the `LINELEN` token in `RPL_ISUPPORT` (005), sent after registration:

```
:server 005 nick LINELEN=8192 :are supported by this server
```

If the server's limit changes (e.g. after a configuration reload), a fresh `RPL_ISUPPORT`
MUST be sent to all connected clients reflecting the updated value.

## Client Behaviour

Clients that negotiate this capability MAY send messages up to the advertised byte length.

Clients that do not negotiate this capability MUST NOT send messages exceeding 512 bytes,
and servers MUST treat them as if this extension is not in effect.

## Server Behaviour

If a client that has negotiated this capability sends a message exceeding the advertised
limit, the server MUST reply with `ERR_INPUTTOOLONG` (417) and MUST NOT relay or process
the message:

```
:server 417 nick :Input line was too long
```

If a client that has *not* negotiated this capability sends a message exceeding 512 bytes,
the server MUST also reply with `ERR_INPUTTOOLONG` (417) and MUST NOT relay the message.

Servers MUST NOT silently truncate messages under any circumstance.

## Server-to-Server Considerations

This extension governs client-to-server line length only. Servers relaying messages to
other servers MUST NOT assume that the remote server supports extended line lengths unless
negotiated via a separate server-to-server protocol mechanism. Servers SHOULD truncate or
reject outbound messages that exceed the remote server's known or assumed limit and log a
warning when doing so.

## Examples

### Successful negotiation and use

```
C: CAP LS 302
S: CAP * LS :rsr.chat/extended-line-length=8192
C: CAP REQ :rsr.chat/extended-line-length
S: CAP * ACK :rsr.chat/extended-line-length
C: NICK alice
C: USER alice 0 * :Alice
S: 005 alice LINELEN=8192 :are supported by this server
S: 001 alice :Welcome to the network

C: PRIVMSG #channel :[4000-byte message body...]
S: (message relayed to channel members that have also negotiated the capability)
```

### Oversized message rejected

```
C: PRIVMSG #channel :[message exceeding 8192 bytes...]
S: :server 417 alice :Input line was too long
```

### Client without capability sends oversized message

```
C: PRIVMSG #channel :[600-byte message, capability not negotiated]
S: :server 417 alice :Input line was too long
```