---
title: rsr.chat/did-sasl
---

:::warning

This specification is still a **draft** and is subject to change at any time.

:::

# rsr.chat/sasl-did

This extension defines two SASL authentication mechanisms, `RSR-DID-WEB` and `RSR-DID-PLC`, that
allow clients to authenticate to IRC servers using W3C Decentralised Identifiers. Both
mechanisms use a server-issued challenge and a client-produced JSON Web Signature over
that challenge. The signature need not be produced inline during the SASL
exchange: clients may pre-request a challenge, complete an OAuth 2.0 delegated signing
flow with a provider that holds their key, and present the resulting signature in a
subsequent SASL exchange. The server validates the signature against the DID document
regardless of how the client obtained it.

## Motivation

DID-based authentication allows users to authenticate using an identity they control
independently of the IRC network — rooted in a web domain they own (`did:web`) or in the
AT Protocol PLC directory (`did:plc`). However, not all clients hold private key material
directly. A user may delegate key custody to an OAuth-capable signing provider: an
identity wallet, an AT Protocol PDS, a hardware security device fronted by an API, or a
browser-based key store. Supporting these delegated signing arrangements requires
decoupling challenge issuance from the SASL exchange so that an interactive OAuth flow 
can complete before the client presents its response. This specification supports both
direct signing (client holds the key) and delegated signing (client obtains a signature
from an OAuth provider) under the same mechanism names, with the server indifferent to
which path the client took.

## Capability

This extension supplements but does not replace the IRCv3 `sasl` capability and MUST
only be offered when `sasl` is also available.

```
CAP * LS :sasl=PLAIN,RSR-DID-WEB,RSR_DID-PLC
```

Clients that wish to authenticate via a DID method MUST negotiate `sasl` and use the
`AUTHENTICATE` command with the mechanism name `RSR-DID-WEB` or `RSR-DID-PLC` as appropriate.

## ISUPPORT Tokens

Servers MUST advertise the following tokens in `RPL_ISUPPORT` (005) when this capability
is active:

- `DIDDNSTIMEOUT=<ms>` — the maximum time in milliseconds the server will wait for DID
  document resolution. Recommended default: `5000`.
- `DIDNONCELEN=<n>` — the byte length of server-generated nonces. Servers MUST NOT
  enforce a value lower than 32.
- `DIDCACHETTL=<s>` — seconds for which the server MAY cache a resolved DID document.
  `0` means no caching.
- `DIDINLINETTL=<s>` — seconds before an inline SASL challenge expires. Recommended
  default: `60`. This is the window available to direct-signing clients.
- `DIDPREISSUETTL=<s>` — seconds before a pre-issued challenge expires. Recommended
  default: `600` (10 minutes). This is the window available to clients completing an
  OAuth flow.
- `DIDALGS=<alg1>,<alg2>,...` — supported JWS algorithm identifiers, in server
  preference order.

Example:

```
:server 005 nick DIDDNSTIMEOUT=5000 DIDNONCELEN=32 DIDCACHETTL=300 DIDINLINETTL=60 DIDPREISSUETTL=600 DIDALGS=EdDSA,ES256K,ES256 :are supported by this server
```

## Two Signing Flows

Both `RSR-DID-WEB` and `RSR-DID-PLC` support two signing flows, declared by the client in its
initial SASL message. The server's behaviour during verification is identical in both
cases; the flow distinction affects only challenge issuance timing and TTL.

### Direct Flow

The client holds the private key locally. The challenge is issued inline during the SASL
exchange and expires after `DIDINLINETTL` seconds. This is appropriate for clients with
direct access to key material.

### Delegate Flow

The client does not hold the private key directly and will obtain a signature from an
external OAuth-capable signing provider. To accommodate the interactive OAuth exchange,
the client pre-requests a challenge before starting SASL using the `DIDCHALLENGE` command
(see Pre-Issued Challenges). The pre-issued challenge expires after `DIDPREISSUETTL`
seconds. During the SASL exchange the client references the pre-issued challenge by its
ID rather than waiting for an inline challenge.

The server does not participate in the OAuth exchange and does not communicate with the
signing provider. From the server's perspective, a delegate flow is simply a SASL
exchange in which the client presents a pre-computed JWS referencing a previously issued
challenge.

## Pre-Issued Challenges

### DIDCHALLENGE REQUEST

```
DIDCHALLENGE REQUEST <did>
```

Requests a challenge for the given DID without beginning SASL. The server resolves the
DID document (see Resolution and Validation) and, if resolution succeeds, issues a
challenge with a TTL of `DIDPREISSUETTL` seconds.

The server replies with `RPL_DIDCHALLENGE`:

```
:server RPL_DIDCHALLENGE <nick> <challenge-id> <did> :<challenge-json-base64>
```

Where:

- `challenge-id` is a server-generated opaque identifier for this pre-issued challenge,
  distinct from the nonce. Clients include this ID in their SASL initial message to
  indicate they are responding to a pre-issued challenge.
- `challenge-json-base64` is the base64-encoded challenge JSON, identical in structure
  to the inline challenge (see Server Challenge Message).

Pre-issued challenges are bound to the connection that requested them. A challenge issued
on one connection MUST NOT be accepted on a different connection.

### DIDCHALLENGE CANCEL

```
DIDCHALLENGE CANCEL <challenge-id>
```

Cancels a pre-issued challenge. The server MUST discard the challenge immediately. This
SHOULD be called by the client if the OAuth flow fails or is abandoned, so that the
server can free the associated state.

### Numerics

| Symbolic name           | Description                                             |
|-------------------------|---------------------------------------------------------|
| RPL_DIDCHALLENGE        | A pre-issued challenge                                  |
| ERR_DIDCHALLENGEFAIL    | Challenge could not be issued (resolution failure, etc.)|
| ERR_DIDCHALLENGEUNKNOWN | The referenced challenge-id is not known or has expired |

## SASL Mechanism: RSR-DID-WEB and RSR-DID-PLC

The two mechanisms are identical in message structure. The mechanism name determines
which DID resolution method the server uses.

### Step 1: Client Initial Message

After sending `AUTHENTICATE RSR-DID-WEB` or `RSR-AUTHENTICATE DID-PLC` the server sends
`AUTHENTICATE +`. The client then sends its initial message as a base64-encoded JSON
object:

For a **direct flow**:

```json
{
  "flow": "direct",
  "did": "<the client's full DID string>",
  "vmId": "<verification method ID>"
}
```

For a **delegate flow**:

```json
{
  "flow": "delegate",
  "did": "<the client's full DID string>",
  "vmId": "<verification method ID>",
  "challengeId": "<the challenge-id from a prior DIDCHALLENGE REQUEST>"
}
```

Fields:

- `flow` — MUST be `direct` or `delegate`. Absent field MUST be treated as `direct`.
- `did` — MUST be a syntactically valid DID matching the mechanism's method. A
  `DID-WEB` exchange MUST present a `did:web:...` DID; `DID-PLC` MUST present a
  `did:plc:...` DID. Method mismatch MUST cause the server to abort with ERR_SASLFAIL.
- `vmId` — the `id` of the verification method in the DID document the client will use
  to sign. The server confirms this method exists and appears under `authentication`.
- `challengeId` — delegate flow only. MUST reference a pre-issued challenge obtained
  via `DIDCHALLENGE REQUEST` on this connection. If absent in a delegate flow, or if
  the referenced challenge has expired or is unknown, the server MUST abort with
  ERR_SASLFAIL.

### Step 2: Server Challenge Message

**Direct flow**: The server issues a fresh challenge with TTL `DIDINLINETTL` and sends
it as a base64-encoded JSON object via `AUTHENTICATE`:

```json
{
  "challengeId": null,
  "nonce": "<DIDNONCELEN random bytes, base64url-encoded>",
  "did": "<the DID the server resolved, echoed>",
  "vmId": "<the vmId echoed>",
  "ts": "<ISO 8601 UTC timestamp of challenge issuance>",
  "ttl": <DIDINLINETTL value in seconds>
}
```

**Delegate flow**: The server has already sent the challenge via RPL_DIDCHALLENGE. It
echoes the challenge ID to confirm it is accepted and proceeds immediately to await the
client response:

```json
{
  "challengeId": "<the challenge-id>",
  "nonce": "<nonce from the pre-issued challenge, verbatim>",
  "did": "<did echoed>",
  "vmId": "<vmId echoed>",
  "ts": "<ts from the pre-issued challenge, verbatim>",
  "ttl": <remaining TTL in seconds at time of SASL initiation>
}
```

In both cases the client MUST verify that the echoed `did` and `vmId` match its request
before signing.

### Step 3: Client Response

The client produces a JSON Web Signature (JWS, RFC 7515) in Compact Serialisation form
over the following payload and sends the result base64-encoded:

```json
{
  "did": "<the client's DID>",
  "vmId": "<the verification method ID>",
  "nonce": "<the nonce from the challenge, verbatim>",
  "ts": "<the ts from the challenge, verbatim>",
  "aud": "<the server's hostname as presented in the 001 welcome message>",
  "flow": "<direct or delegate>"
}
```

In a direct flow the client signs this payload with its locally held private key.

In a delegate flow the client passes this payload to its OAuth signing provider and
receives a signed JWS in return (see OAuth Delegation). The JWS presented to the server
is the one returned by the provider, not one produced by the client itself.

In either case:

- The JWS `alg` header MUST be one of the algorithms listed in `DIDALGS`.
- The JWS `kid` header MUST be set to `vmId`.
- The server MUST verify the `aud` claim matches its own hostname.
- The server MUST verify the `nonce` and `ts` match the issued challenge.
- The server MUST verify the challenge has not expired.

### Server Verification

On receipt of the client response the server MUST:

1. Decode and parse the JWS.
2. Verify `aud` matches the server's own hostname.
3. Locate the challenge (inline or pre-issued) and confirm it has not expired.
4. Verify the `nonce` and `ts` in the payload match the challenge verbatim.
5. Locate the verification method identified by `vmId` in the resolved DID document.
6. Verify the located method appears in the `authentication` verification relationship.
7. Extract the public key material from the verification method.
8. Validate the JWS signature against the payload using the extracted key.
9. Consume the nonce; it MUST NOT be accepted again.

If all checks pass the server completes authentication with RPL_SASLSUCCESS (903). If any
check fails the server MUST reply with ERR_SASLFAIL (904) and MUST NOT disclose which
specific check failed.

## OAuth Delegation

This section describes the client-side OAuth delegation pattern. The server does not
participate in this flow and requires no OAuth-specific configuration.

### Overview

In the delegate flow, the user's private key is held by an OAuth-capable signing
provider rather than the client. The DID document's verification method references the
provider's public key (or a key the provider holds on the user's behalf). The client:

1. Pre-requests a challenge from the IRC server via `DIDCHALLENGE REQUEST`.
2. Authenticates to the signing provider via an OAuth 2.0 flow.
3. Requests the provider to sign the JWS payload containing the IRC nonce.
4. Receives the signed JWS from the provider.
5. Presents the JWS to the IRC server via SASL.

The signing provider sees and signs the IRC nonce but does not communicate with the IRC
server directly. The IRC server sees and verifies a JWS but does not communicate with
the provider. The client mediates between the two.

### DID Document Convention for Delegated Signing

Verification methods that represent keys held by a signing provider SHOULD include a
`serviceEndpoint` property pointing to the provider's signing API. This is a convention
for client discovery; the server does not read or validate it:

```json
{
  "id": "did:web:alice.example.com#key-oauth-1",
  "type": "JsonWebKey2020",
  "controller": "did:web:alice.example.com",
  "publicKeyJwk": { ... },
  "serviceEndpoint": "https://wallet.example.com/sign"
}
```

Clients that discover a `serviceEndpoint` on the target verification method SHOULD use
it as the signing API endpoint for the delegate flow. Clients MUST NOT transmit the
`serviceEndpoint` to the IRC server.

### OAuth 2.0 Signing Flow

The specific OAuth grant type and API shape are defined by the signing provider. The
following is a RECOMMENDED approach that client and provider authors SHOULD follow for
interoperability.

#### 1. Obtain a Signing Token via OAuth 2.0 with PKCE

The client initiates a standard OAuth 2.0 authorisation code flow (RFC 6749) with PKCE
(RFC 7636) against the provider's authorisation server. The requested scope SHOULD
include a provider-defined scope that grants signing permission, e.g. `irc:sign` or
`did:sign`. For non-interactive clients with pre-established credentials a client
credentials grant MAY be used instead.

Clients and providers that support OAuth 2.0 Pushed Authorisation Requests (PAR,
RFC 9126) SHOULD use PAR to bind the IRC challenge nonce to the authorisation request,
providing an additional binding between the OAuth flow and the IRC challenge:

```json
POST /oauth/par
{
  "client_id": "...",
  "response_type": "code",
  "scope": "irc:sign",
  "code_challenge": "...",
  "code_challenge_method": "S256",
  "irc_challenge_nonce": "<the nonce from the IRC server>"
}
```

The authorisation server MAY display the `irc_challenge_nonce` and the IRC server
hostname to the user during the consent screen so they can verify what they are
authorising.

#### 2. Exchange the Token for a Signature

After obtaining an access token the client calls the provider's signing endpoint with
the JWS payload to be signed:

```http
POST /sign
Authorization: DPoP <access-token>
Content-Type: application/json

{
  "vmId": "<the verification method ID>",
  "payload": {
    "did": "...",
    "vmId": "...",
    "nonce": "...",
    "ts": "...",
    "aud": "...",
    "flow": "delegate"
  }
}
```

DPoP (RFC 9449) is RECOMMENDED for binding the access token to the signing request and
preventing token exfiltration. Providers that support DPoP MUST require it for signing
requests.

The provider signs the payload with the private key corresponding to `vmId`, constructs
the JWS in Compact Serialisation, and returns it:

```json
{
  "jws": "<header>.<payload>.<signature>"
}
```

#### 3. Present the JWS to the IRC Server

The client uses the returned JWS as its SASL response in Step 3 of the SASL exchange.
The IRC server has no knowledge that the JWS was produced by a provider rather than the
client and validates it identically.

### Non-Interactive Delegate Flow

For automated clients that have a pre-established API relationship with a signing
provider (e.g. a bot whose key is managed by a secrets manager), the PKCE flow is
unnecessary. Such clients MAY use a client credentials grant or API key to obtain a
signing token and call the signing endpoint programmatically. Provided the total time
stays within `DIDPREISSUETTL` seconds this is operationally transparent.

### did:plc and AT Protocol PDS Signing

Users with a `did:plc` identity whose key is managed by an AT Protocol PDS SHOULD use
the PDS's OAuth 2.0 interface as the signing provider. AT Protocol PDSes that implement
the AT Protocol OAuth specification expose a signing endpoint compatible with the
delegate flow described here. The PDS verification method in the DID document references
the PDS-held key, and the PDS signs the IRC challenge after the user authorises the
request via the standard AT Protocol OAuth consent flow.

Clients that target AT Protocol PDSes SHOULD use PAR with the `irc_challenge_nonce`
extension to bind the authorisation request to the IRC challenge. The PDS's OAuth server
SHOULD surface the IRC server hostname and challenge timestamp on its consent screen.

## DID Method Support

### did:web

A `did:web` DID document is retrieved from an HTTPS endpoint derived from the DID:

| DID                               | Document URL                                   |
|-----------------------------------|------------------------------------------------|
| `did:web:example.com`             | `https://example.com/.well-known/did.json`     |
| `did:web:example.com:users:alice` | `https://example.com/users/alice/did.json`     |

Servers MUST use HTTPS. Servers MUST NOT follow redirects to non-HTTPS endpoints. TLS
certificates MUST be validated. Non-200 responses and malformed JSON MUST fail with
ERR_SASLFAIL.

### did:plc

A `did:plc` DID document is retrieved from:

```
https://plc.directory/<did>
```

Servers MUST use the canonical PLC directory. Servers MUST NOT permit clients to specify
an alternative directory. TLS validation applies as for `did:web`.

## Supported Algorithms

Servers MUST support:

| Algorithm | Description                     | Key type in DID document                              |
|-----------|---------------------------------|-------------------------------------------------------|
| `EdDSA`   | Ed25519 (RFC 8037)              | `JsonWebKey2020` or `Ed25519VerificationKey2020`      |
| `ES256K`  | ECDSA with secp256k1 (RFC 8812) | `JsonWebKey2020` or `EcdsaSecp256k1VerificationKey2019` |
| `ES256`   | ECDSA with P-256 (RFC 7518)     | `JsonWebKey2020`                                      |

Servers MAY support additional algorithms. `EdDSA` (Ed25519) is RECOMMENDED. Clients
SHOULD prefer it where their key material supports it.

## Resolution and Validation

### DID Document Requirements

Before issuing any challenge (inline or pre-issued) the server MUST resolve and validate
the DID document. The server MUST confirm:

- The `id` field matches the client-presented DID by exact string equality.
- A `verificationMethod` array is present with at least one entry.
- The `vmId` specified by the client exists within `verificationMethod`.
- The matching method contains `publicKeyJwk` or `publicKeyMultibase`.
- The matching method is referenced from the `authentication` array.

Failure on any condition MUST produce ERR_SASLFAIL for inline flows or
ERR_DIDCHALLENGEFAIL for pre-issued challenge requests.

### Resolution Security

Servers MUST:

- Reject plain HTTP resolution.
- Not follow redirects that cross scheme or target private or loopback address ranges.
- Enforce `DIDDNSTIMEOUT` as a hard wall-clock deadline on the entire operation.
- Reject response bodies larger than 64 KiB.
- Not execute active content in the response.
- Prefer `Content-Type: application/did+ld+json`; reject other types unless explicitly
  configured to accept them.

### Caching

If `DIDCACHETTL` is non-zero the server MAY cache resolved documents by DID string for
the advertised TTL. Cached documents MUST be stored in memory only. Servers MUST NOT
cache resolution failures. Users who have recently rotated keys SHOULD wait for the
cache to expire before re-authenticating.

## Account Association

On successful authentication the server associates the client's DID with their IRC
account. The following invariants MUST hold:

- A DID MAY be associated with at most one IRC account. Authentication from a DID
  already associated with a different account MUST fail with ERR_SASLFAIL.
- An IRC account MAY have multiple DIDs associated, subject to a server-configured limit.
- Association is persistent across sessions.
- The authenticated DID is exposed in WHOIS (see WHOIS Integration).

Servers MAY support automatic account creation on first successful DID authentication
or MAY require pre-registration. If a client authenticates with a DID that has no
associated account and the server does not support auto-registration, the server MUST
return ERR_SASLFAIL and log the DID for operator review.

## WHOIS Integration

When a client is authenticated via a DID mechanism the server MUST include the
authenticated DID in WHOIS via RPL_WHOISDID:

```
:server RPL_WHOISDID <querier> <nick> :<did>
```

Servers MUST include this in self-WHOIS. Servers MAY suppress it in WHOIS by other
users depending on server policy.

## Key Rotation

Rotating keys in the DID document does not invalidate existing sessions. Servers MAY
implement a `DIDSYNC` command or operator procedure to force re-authentication of active
sessions, outside the scope of this specification. Clients and signing providers that
rotate keys SHOULD re-authenticate at the next session.

For the delegate flow, key rotation at the signing provider requires updating the
verification method's public key in the DID document and, if applicable, its
`serviceEndpoint`. Clients SHOULD request a fresh pre-issued challenge after a provider
key rotation rather than presenting a challenge resolved against a stale document.

## Guild Integration

When `rsr.chat/guild` is negotiated, a user's DID serves as a portable identity anchor
for guild membership. Servers that support both this extension and `rsr.chat/guild` MAY
allow guild administrators to require DID-based authentication for guild join, rejecting
join attempts from accounts not authenticated via a DID mechanism.

## Channel Membership Integration

When `rsr.chat/channel-membership` is negotiated, persistent membership records MAY
store the authenticated DID alongside the account name as an informational field. This
field is not used for access control decisions unless `rsr.chat/rbac` defines rules that
reference it.

## RBAC Integration

When `rsr.chat/rbac` is negotiated, the following permission identifiers are defined:

| Permission identifier        | Description                                             |
|------------------------------|---------------------------------------------------------|
| `did.auth.require`           | Only DID-authenticated clients may join or post         |
| `did.auth.method=did:web`    | Only did:web authenticated clients are permitted        |
| `did.auth.method=did:plc`    | Only did:plc authenticated clients are permitted        |
| `did.auth.flow=direct`       | Only direct-flow authenticated clients are permitted    |
| `did.auth.flow=delegate`     | Only delegate-flow authenticated clients are permitted  |

RBAC rules using these identifiers MAY be scoped to a channel category or guild prefix
using the scope syntax defined by `rsr.chat/channel-categories`.

## Advanced Moderation Integration

When `rsr.chat/advanced-moderation` is negotiated, the following predicates are defined:

| Predicate                  | Matches when                                                   |
|----------------------------|----------------------------------------------------------------|
| `did:authenticated`        | The user is authenticated via any DID mechanism                |
| `did:method=did:web`       | The user is authenticated via did:web                         |
| `did:method=did:plc`       | The user is authenticated via did:plc                         |
| `did:flow=direct`          | The user authenticated via the direct signing flow             |
| `did:flow=delegate`        | The user authenticated via the OAuth delegate flow             |
| `did:domain=<domain>`      | The user's did:web DID is rooted at the specified domain       |
| `did:value=<did>`          | The user's authenticated DID matches exactly                   |

## Error Handling Summary

| Condition                                     | Response                          |
|-----------------------------------------------|-----------------------------------|
| Method mismatch                               | ERR_SASLFAIL (904)                |
| DID resolution timeout                        | ERR_SASLFAIL or ERR_DIDCHALLENGEFAIL |
| DID document fetch non-200 or malformed       | ERR_SASLFAIL or ERR_DIDCHALLENGEFAIL |
| vmId not found or not in authentication       | ERR_SASLFAIL or ERR_DIDCHALLENGEFAIL |
| challengeId not found or expired              | ERR_SASLFAIL (904)                |
| challengeId belongs to a different connection | ERR_SASLFAIL (904)                |
| Unsupported JWS algorithm                     | ERR_SASLFAIL (904)                |
| Signature verification failure                | ERR_SASLFAIL (904)                |
| Nonce or timestamp mismatch                   | ERR_SASLFAIL (904)                |
| aud mismatch                                  | ERR_SASLFAIL (904)                |
| Inline challenge expired                      | ERR_SASLFAIL (904)                |
| Pre-issued challenge expired                  | ERR_SASLFAIL (904)                |
| DID associated with a different account       | ERR_SASLFAIL (904)                |
| No account and no auto-registration           | ERR_SASLFAIL (904)                |

Servers MUST NOT distinguish between these conditions in the ERR_SASLFAIL message text
in order to avoid leaking information about account existence or document state.

## Security Considerations

### Nonce Freshness

Nonces MUST be generated using a cryptographically secure random source. Nonces MUST be
single-use and MUST be consumed on first verification attempt regardless of success or
failure. A consumed nonce MUST NOT be reissued.

### Challenge Binding

Pre-issued challenges are bound to the connection that requested them. Servers MUST
reject a pre-issued challenge presented on a different connection than the one that
called `DIDCHALLENGE REQUEST`. This prevents challenge theft by a third party who
intercepts a challenge-id.

### aud Binding

The `aud` claim in the JWS payload binds the signature to a specific IRC server hostname,
preventing cross-server replay. Servers MUST validate this claim. Clients in the delegate
flow MUST include the IRC server's hostname in the payload they send to the signing
provider; providers SHOULD surface this value to the user during consent.

### Delegate Flow Considerations

In the delegate flow the signing provider sees the IRC nonce and could, in principle,
replay the signed JWS against the IRC server within the `DIDPREISSUETTL` window. Clients
SHOULD use providers they trust. Clients SHOULD use PAR with the `irc_challenge_nonce`
parameter and DPoP token binding when supported by the provider to reduce the window
for provider-side misuse.

The extended `DIDPREISSUETTL` window compared to `DIDINLINETTL` is a deliberate
trade-off: it accommodates interactive OAuth flows at the cost of a larger replay window.
Servers that are highly sensitive to replay risk MAY configure `DIDPREISSUETTL` lower
than the default, accepting that some OAuth flows may time out.

### DID Document Injection

Servers MUST compare the `id` field of a resolved document to the requested DID using
exact string equality. Servers MUST NOT accept a document whose `id` differs from the
requested DID even by case or Unicode normalisation.

### Key Compromise

If a user's private key (or the signing provider's key) is compromised, an attacker can
authenticate until the key is rotated and the server's cache expires. Users SHOULD rotate
keys promptly on suspected compromise and notify server operators so that cached documents
may be invalidated out-of-band.

## Examples

### Direct flow: DID-WEB

```
C: CAP LS 302
S: CAP * LS :sasl=PLAIN,DID-WEB,DID-PLC rsr.chat/sasl-did
C: CAP REQ :sasl rsr.chat/sasl-did
S: CAP * ACK :sasl rsr.chat/sasl-did
C: AUTHENTICATE DID-WEB
S: AUTHENTICATE +
C: AUTHENTICATE eyJmbG93IjoiZGlyZWN0IiwiZGlkIjoiZGlkOndlYjphbGljZS5leGFtcGxlLmNvbSIsInZtSWQiOiJkaWQ6d2ViOmFsaWNlLmV4YW1wbGUuY29tI2tleS0xIn0=
(server resolves https://alice.example.com/.well-known/did.json, issues inline challenge)
S: AUTHENTICATE eyJjaGFsbGVuZ2VJZCI6bnVsbCwibm9uY2UiOiJyYW5kb21ub25jZWJhc2U2NHVybCIsImRpZCI6ImRpZDp3ZWI6YWxpY2UuZXhhbXBsZS5jb20iLCJ2bUlkIjoiZGlkOndlYjphbGljZS5leGFtcGxlLmNvbSNrZXktMSIsInRzIjoiMjAyNC0wMy0xNVQxNDoyMjowMS4wMDBaIiwidHRsIjo2MH0=
C: AUTHENTICATE <base64-encoded JWS signed by alice's local key>
S: :server 903 alice :Authentication successful
S: 001 alice :Welcome to the network
S: 005 alice DIDDNSTIMEOUT=5000 DIDNONCELEN=32 DIDCACHETTL=300 DIDINLINETTL=60 DIDPREISSUETTL=600 DIDALGS=EdDSA,ES256K,ES256 :are supported by this server
```

### Delegate flow: DID-PLC via AT Protocol PDS

```
(before starting SASL, the client pre-requests a challenge)
C: DIDCHALLENGE REQUEST did:plc:ewvi7nxzyoun6zhhandbv25b
(server resolves https://plc.directory/did:plc:ewvi7nxzyoun6zhhandbv25b)
S: :server RPL_DIDCHALLENGE alice ch_01HQ7K2N4R8VXJ did:plc:ewvi7nxzyoun6zhhandbv25b :eyJjaGFsbGVuZ2VJZCI6ImNoXzAxSFE3SzJONFI4VlhKIiwibm9uY2UiOiJyYW5kb21ub25jZSIsInRzIjoiMjAyNC0wMy0xNVQxNDoyMjowMS4wMDBaIiwidHRsIjo2MDB9

(client now initiates OAuth 2.0 + PKCE flow against the AT Protocol PDS)
(PDS consent screen shows: "IRC server irc.example.net is requesting a signature
    for challenge nonce aBcDeFgH... issued at 2024-03-15T14:22:01Z")
(user approves; client receives access token and calls PDS signing API)
(PDS returns signed JWS; entire OAuth + signing round-trip takes ~25 seconds)

(client now begins SASL)
C: AUTHENTICATE DID-PLC
S: AUTHENTICATE +
C: AUTHENTICATE eyJmbG93IjoiZGVsZWdhdGUiLCJkaWQiOiJkaWQ6cGxjOmV3dmk3bnh6eW91bjZ6aGhhbmRidjI1YiIsInZtSWQiOiJkaWQ6cGxjOmV3dmk3bnh6eW91bjZ6aGhhbmRidjI1YiNhdGpwcm90b0Jsa0UiLCJjaGFsbGVuZ2VJZCI6ImNoXzAxSFE3SzJONFI4VlhKIn0=
(server locates pre-issued challenge ch_01HQ7K2N4R8VXJ, confirms not expired,
    echoes it back)
S: AUTHENTICATE eyJjaGFsbGVuZ2VJZCI6ImNoXzAxSFE3SzJONFI4VlhKIiwibm9uY2UiOiJyYW5kb21ub25jZSIsImRpZCI6ImRpZDpwbGM6ZXd2aTdueHp5b3VuNnpoaGFuZGJ2MjViIiwidm1JZCI6Ii4uLiIsInRzIjoiMjAyNC0wMy0xNVQxNDoyMjowMS4wMDBaIiwidHRsIjo1NzV9
C: AUTHENTICATE <base64-encoded JWS returned by the AT Protocol PDS signing API>
S: :server 903 alice :Authentication successful
```

### Pre-issued challenge cancelled after failed OAuth flow

```
C: DIDCHALLENGE REQUEST did:web:alice.example.com
S: :server RPL_DIDCHALLENGE alice ch_01HQAB3T2F cancelled
(OAuth flow fails at provider; client cleans up)
C: DIDCHALLENGE CANCEL ch_01HQAB3T2F
(server discards the challenge)
```

### WHOIS showing authenticated DID and flow

```
C: WHOIS alice
S: :server 311 bob alice ~alice alice.example.com * :Alice
S: :server RPL_WHOISDID bob alice :did:plc:ewvi7nxzyoun6zhhandbv25b
S: :server 318 bob alice :End of /WHOIS list
```
### Resolution failure during DIDCHALLENGE REQUEST

```
C: DIDCHALLENGE REQUEST did:web:unreachable.example.com
S: :server ERR_DIDCHALLENGEFAIL alice :DID document could not be resolved
```

### Expired pre-issued challenge presented in SASL

```
(challenge ch_01HQ7K issued 620 seconds ago; DIDPREISSUETTL=600)
C: AUTHENTICATE DID-WEB
S: AUTHENTICATE +
C: AUTHENTICATE <initial message referencing ch_01HQ7K>
S: :server 904 alice :SASL authentication failed
```

### Challenge presented on wrong connection

```
(ch_01HQCD was issued on a different connection)
C: AUTHENTICATE DID-WEB
S: AUTHENTICATE +
C: AUTHENTICATE <initial message referencing ch_01HQCD>
S: :server 904 alice :SASL authentication failed
```