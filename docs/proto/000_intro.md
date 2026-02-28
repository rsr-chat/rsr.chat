---
id: intro
sidebar_position: 000
---

# Introduction

## What is RSR?

RSR is a real-time communication and community management
system that builds upon numerous open-source and open-standard
components to create a decentralized, sovereign chat protocol.

## Design Philosophy

1. **Degrade Gracefully**       - No client should become unable to use the 
                                  core chat functionality as long as they can 
                                  act as a standards-compliant IRC client.
                                  
2. **First-Class Sovereignty**  - All users' data, except for sent messages,
                                  must be fully controlled by that user.
                                  No single community or hosting provider
                                  should be able to change, delete, or
                                  impersonate that user. Guild identities
                                  must similarly remain in full control of
                                  their owner(s), and those guild owner(s)
                                  must be able to revoke authority over
                                  their guild from a hosting server regardless
                                  of that host server's consent. When a guild
                                  moves hosts, user connections must follow
                                  automatically.

3. **Frictionless Flow**        - Usage of RSR must be effortless even for
                                  non-technical clients; it should not be
                                  difficult, frustrating, or overly technical
                                  to use RSR systems in the capacity of
                                  a user looking to communicate with other users.

## Vocabulary

The following terms are used throughout this document. Where a term has an 
established meaning in a referenced standard, that standard's definition 
applies unless otherwise noted.

- **Identity** — A [W3C Decentralized Identity Document (DID)](https://www.w3.org/TR/did-1.0/)
that authoritatively describes a principal in the RSR protocol. An Identity may represent a
User, a Guild, or both.

- **User** — A principal representing a human participant, identified by an
Identity. A User may belong to zero or more Guilds and interact with the
protocol through one or more Clients.

- **Guild** — A persistent community resource, identified by an Identity, of
which zero or more Users may be members. A Guild exists independently of any
particular Server and may be migrated or delegated across Servers.

- **Client** — A program acting on behalf of a User. A Client authenticates
using the User's Identity and manages one or more Sessions on the User's behalf.

- **Server** — A service that hosts one or more Guilds and accepts Client
connections. A Server facilitates access to Guild events and state but does
not own the Guilds it hosts; Guild ownership is determined by the Guild's Identity.

- **Session** — A single active connection between a Client and a Server over
the RSR protocol, through which the Client may send messages and subscribe to events.