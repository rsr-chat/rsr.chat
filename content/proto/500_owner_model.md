---
id: data-ownership
sidebar_position: 500
---

# Content Ownership Model

The RSR Protocol breaks down all content 
transmitted and received into discrete categories
based on the *burden of hosting*. All content
transmitted and received must be hosted by
either the Identity that created the content,
or the server relaying the content.

As a general rule, Servers act as commodity
infrastructure that exist to relay signals in
real-time, and do not authoritatively control
User or Guild configuration. 

## Identity-Hosted Content

Uploaded media such as Images, Videos, custom
emoji, stickers, et cetera are all hosted on
ther relevant Identity's PDS. Users upload their
media to their own PDS, and then can send their
media as a hyperlink to the uploaded media blob
in a textual message they send to a server.

User configuration that affects how clients behave
are stored in client-specific lexicons on the user 
identity's PDS. User social graph information is
also stored in their PDS.

Guild configuration is entirely stored in that
guild's PDS, which includes membership information 
(incl. bans and invites), role-based access control 
configuration, et cetera are all stored in the Guild
Identity's PDS.

This means that if a User leaves a community, or
if a User joins multiple communities across different
servers, that User's media, client configuration, etc.
all remain in their control, and no third party may
interfere with that data. Similarly, it means Guilds
may not dictate what data a User creates, only what
data it accepts from the User; moderation remains
within the boundaries of the Guild only.

## Server-Hosted Content

Servers are responsible for hosting textual message
history. They MAY cache identity-hosted content,
but MUST NOT ever change the information they cache.

This means that when a Guild changes the servers its
hosted on, all its configuration, member lists, uploaded
media, et cetera remains in full control of the Guild
owner(s), and the host server has no say in whether
or not this information is allowed to leave its
jurisdiction.

Message history is too heavy to be stored in an ATProto
PDS, but the RSR Protocol's external attestation and
replication capabilities mitigate the risk of losing
guild messaging history to a server that misbehaves or
acts in bad faith.