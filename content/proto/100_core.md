---
id: core-proto
sidebar_position: 100
---

# Core IRCv3 Protocol

The protocol underlying the RSR chat system is the 
[Internet Relay Chat v3](https://ircv3.net/irc/) protocol 
(herenafter referred to as "IRCv3").

The original specification for the IRC protocol aimed
for real-time messaging in an era where client connectivity
was limited to times when users were actively chatting;
a user would connect to the IRC network, join some channels,
chat, and then leave the network when they were done chatting.
Modern communications software, however, often presents an
interface that provides the appearance of constant connectivity
which involve communities with persistent *membership* lists
as opposed to IRC's transient "people entering and exiting a
room" model.

In order to support future changes to the protocol spec as
well as vendor-specific features, the IRCv3 working group
has added a new feature called *Capability Negotiation*
which allows a server to advertise extensions that clients
may request and enable, but which are otherwise disabled
by default. This allows older clients to connect and use
the IRC network as usual, while newer clients can provide
enhanced functionality to users as they implement the 
respective features.

RSR is built upon this foundation of capability negotiation.
An RSR-compliant community can be as simple as a single IRC
server that implements `rsr.chat/did-signing` ... or it can 
be as complex as a custom distributed backend that implements
every single extension under the sun. Even further to that 
point, RSR communities may optionally accept unsigned,
"legacy" IRC connections as well if their owners so choose, 
meaning that no community needs to be permanently tied to 
RSR paradigms in the first place. 