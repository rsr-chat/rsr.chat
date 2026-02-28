---
id: registration
sidebar_position: 400
---

# Accounts and Registration

The RSR protocol does not specify nor control user
or guild account registration. A user or guild
identity is represented by a DID document as specified 
by the [W3C Decentralized Identity Document](https://www.w3.org/TR/did-1.0/)
standard.

## What is an Identity?

An RSR-compatible identity URL is represented by
a string in the format `did:mmm:xxxxxxxxxx...`, where 
`example` is the [DID Method](https://www.w3.org/TR/did-1.0/#methods)
used to retrieve the corresponding document and `xxxx...`
is the method-specific unique identifier.

Clients and Servers MUST support at minimum the 
[`plc`](https://github.com/did-method-plc/did-method-plc), 
[`ref`](https://github.com/rsr-chat/did-method-ref), and 
[`web`](https://w3c-ccg.github.io/did-method-web/) (TLS only)
methods. They MAY support other methods and are encouraged
to do so.

DID URLs MUST resolve to a JSON document in the following format:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/multikey/v1",
    "https://w3id.org/security/suites/secp256k1-2019/v1"
  ],
  "alsoKnownAs": [
    "rsr://myname.example.com"
  ],
  "id": "did:mmm:xxxxxxxxxx...",
  "service": [
    {
      "id": "#rsr_pds",
      "serviceEndpoint": "https://plc-endpoint.example.com",
      "type": "AtprotoPersonalDataServer"
    }
  ],
  "verificationMethod": [
    {
        "controller": "did:plc:xxxxxxxxxxxxxx",
        "id": "did:plc:xxxxxxxxxxxxxx...",
        "publicKeyMultibase": "yyyyyyyyyyyyyyyyyyyy",
        "type": "Multikey"
    }
  ]
}
```

Other fields may be added as per the DID specification;
the above example represents the minimal requirements.

Identity-related data are stored in the linked `#rsr_pds`
ATProto Personal Data Server using the `chat.rsr` lexicons.

## Creating a new Identity

By design, creation of identities is not specified by the
RSR protocol. Frontend clients MAY elect to provide a user
interface for creating new identities, but MUST never send
information to any server which allows the server to control
that identity.

In general, clients seeking to create a new identity should
create a DID-complient identity via any compatible mechanism
and present the user with the keys to that identity record,
then register that newly created DID under an ATProto PDS.

Clients seeking to use a preexisting identity on behalf of
a user should retrieve an OAuth client grant from the user's
PDS - this grant must not be sent to any guild or remote
server.

Guild identities are created identically to user identities;
the difference is in what kind of data is available in that
identity's PDS.