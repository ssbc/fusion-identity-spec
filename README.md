<!--
SPDX-FileCopyrightText: 2021 Anders Rune Jensen

SPDX-License-Identifier: CC-BY-4.0
-->

# Fusion Identity Spec v1 (DRAFT)

The fusion identity spec is a specification for how to relate multiple
devices in such a way that they represent a new combined (fusion)
identity. The primary purpose is an individual with multiple devices
and thus multiple [SSB] identities. Only new members can be added to a
fusion, there is no removal of members only tombstoning.

In the case where a fusion identity is tombstoned because a member is
removed and another fusion must take its place, the concept of
redirecting and attestations of these was considered. But instead of
baking these concepts into the protocol in an initial version, we have
decided to include them in an future version once we get more real
world experience.

The string representation of a fusion id is:
`ssb:identity/fusion/<PUBLIC_KEY>`

Usages:
 - Can mention a fusion identity and all members can use that as
   notification
 - Can private message a fusion identity and all members will be able
   to decrypt the message and respond
 - Acts as a public record of what feeds are part of an identity
   (discoverability)
 - Following a fusion identity for replication
 - Messages can be shown as coming from the same identity across
   multiple devices

Out of scope for v1:
 - Redirection
 - Attestation of redirects
 - Private invitations to fusion identity
 - Inviting fusion identities to private groups
 - Use of fusion identity for authorisation logic
     - this is a priority, but will be addressed in subsequent versions
 - Invite a fusion identity to an existing fusion identity

## Overview

Alice has SSB installed on per laptop and installs a client on her
phone. She wants one identity that links her two devices. Either
device can do an `init` to create a new fusion identity. Lets assume
laptop does. Then laptop `invite` phone to the identity. The phone
`consent` the invite, after which laptop will `entrust` phone with the
private key. Lastly phone posts a `proof-of-key` message announcing
the possession of the key. At this point phone is now a member of the
fusion identity and can invite other devices.

```
   @laptop                      @phone
   -----------------            -----------------
   init ->
   invite: @phone ->
                                <- consent
   entrust ->
                                proof-of-key
```

If Alice is at a party and looses her rooted phone on the way
home. From this point on she `tombstones` the fusion identity to tell
other peers not to send private messages to the fusion identity any
longer as those messages might be read by a third party.

## Fusion Identity Operations:

### init

Start the new identity. This results in two messages: an init message
and a key to self.

```js
{
  type: 'fusion',
  subtype: 'fusion/init',
  id: 'ssb:identity/fusion/3h89vDLAcoi68wXZRZ2kVrXs0WT0vthGo9uW1XBPLhU=',
  members: {
    '@mmEKdNyVBtxQM3bwAVS8ujJvi/C1PR07tEQZVtyCp1c=.ed25519': 1
  },
  tangles: {
    fusion: { root: null, previous: null }
  }
}
```

DM key to self:

```js
{
  type: 'fusion/entrust',
  secretKey: 'k4eUDLq4rzUvWRxYjVM8PE64DiU6tidLEn+5ERax3OnbO0IXQhe4yQAHom/lnGjrV+hIi3fvSHoG/UuIuuecvA==',
  rootId: '%ZxAJbfRTwhkhpD9viErN9zBzIEzrm6FTndaH/bEnbfI=.sha256',
  recps: [
    'ssb:identity/fusion/3h89vDLAcoi68wXZRZ2kVrXs0WT0vthGo9uW1XBPLhU=',
    '@mmEKdNyVBtxQM3bwAVS8ujJvi/C1PR07tEQZVtyCp1c=.ed25519'
  ]
}
```

`secretKey` is in base64.

To keep things simple the public encryption key for DMs is the same as
the identity.

The author of the message is now the first member of the fusion
identity.

Preconditions:
 - The `id` must not be used in another fusion identity (different
   root), in that case all fusions with the same id is considered
   tombstoned.

### invite

Invite another feed to join a fusion.

```js
{
  type: 'fusion',
  subtype: 'fusion/invite',
  invited: {
    '@2Cu6gvifd39hHvE/HkT4M7dP5KY5CZ+AsYzM1w2mtT8=.ed25519': 1
  },
  tangles: {
    fusion: {
      root: '%ZxAJbfRTwhkhpD9viErN9zBzIEzrm6FTndaH/bEnbfI=.sha256', // init
      previous: ['%ZxAJbfRTwhkhpD9viErN9zBzIEzrm6FTndaH/bEnbfI=.sha256'] // init
    }
  }
}
```

Preconditions:
 - The feed writing this message message must be a member
 - You can invite multiple feeds
 - Inviting yourself is considered an error

### consent

Accept an invitation message. One is not a member of the fusion yet.

```js
{
  type: 'fusion',
  subtype: 'fusion/consent',
  consented: { 
    '@2Cu6gvifd39hHvE/HkT4M7dP5KY5CZ+AsYzM1w2mtT8=.ed25519': 1
  },
  tangles: {
    fusion: {
      root: '%ZxAJbfRTwhkhpD9viErN9zBzIEzrm6FTndaH/bEnbfI=.sha256', // init
      previous: ['%1UFhwpETE+dR/3IiqXpMIxsh5ga6DsCj6if8Q0JMYAE=.sha256'] // invite
    }
  }
}
```

Preconditions:
 - The author must be invited to the fusion
 - The author must not have consented the fusion before
 - The author must not be a member of the fusion

### entrust

Give secret key to someone that has consented an invite. This is
similar to the message to sent self on `fusion/init`.

```js
{
  type: 'fusion/entrust',
  secretKey: 'k4eUDLq4rzUvWRxYjVM8PE64DiU6tidLEn+5ERax3OnbO0IXQhe4yQAHom/lnGjrV+hIi3fvSHoG/UuIuuecvA==',
  rootId: '%ZxAJbfRTwhkhpD9viErN9zBzIEzrm6FTndaH/bEnbfI=.sha256',
  consentId: '%J0IY7Xxx6OqUr9HXlhIQGJoad+saCEOxkYlkkViY8Oc=.sha256',
  recps: [
    'ssb:identity/fusion/3h89vDLAcoi68wXZRZ2kVrXs0WT0vthGo9uW1XBPLhU=',
    '@2Cu6gvifd39hHvE/HkT4M7dP5KY5CZ+AsYzM1w2mtT8=.ed25519'
  ]
}
```

`secretKey` is in base64.

Preconditions:
 - The recipient must have consented the fusion invite

NOTE:
 - we could have included a private section of the `fusion/invite`
   which included the key, but decided against this because it makes
   the flow less clear
 - adding an additional step to send the key after consent does not
   add significant latency because we are presumed to be in control of
   all the devices in this fusion identity

### proof-of-key / member

A way to publicly announce that you are in possession of private key
and thus announce that you are now a member.

```js
{
  type: 'fusion',
  subtype: fusion/proof-of-key,
  members: { 
    '@2Cu6gvifd39hHvE/HkT4M7dP5KY5CZ+AsYzM1w2mtT8=.ed25519': 1
  },
  proofOfKey: 'fqhDYLkijSEKhYD3nQziMcszCFVxBaIAEZuue+1RA/dhm14OgryVHOXK6fhwsdlrFzj58HWBPZUAjVz4zafYCQ==.sig.ed25519',
  tangles: {
    fusion: { 
      root: '%ZxAJbfRTwhkhpD9viErN9zBzIEzrm6FTndaH/bEnbfI=.sha256', // init
      previous: ['%71lcX2CY306hlZQW1UtoZ0uODm/RurnGlc8mCCjwn7w=.sha256'] // consent
  }
}
```

`proofOfKey` must be signed by the entrusted fusion key as
sign(%consentId + 'fusion/proof-of-key').

Preconditions:
 - The author must be have consented an invite in %consentId
 - The author writes this proof-of-key message when it sees the
   entrust, this ensures that they are in posession of the key

### tombstone

Nullify or revoke a fusion identity.

Given you have a shared private key, there is no easy way to "remove"
a device from a fusion identity. Our solution is to "tombstone"
identities and require that you mint a new fusion identity with the
devices you trust.

```js
{
  type: 'fusion',
  tombstone: { 
    set: { 
      date: 1641983852373, 
      reason: 'bye' 
    }
  },
  tangles: {
    fusion: {
      root: '%ZxAJbfRTwhkhpD9viErN9zBzIEzrm6FTndaH/bEnbfI=.sha256', // init
      previous: ['%Wo2vAn/LuCCDauDfbZEncCu/5eV/uuk2M1tIbT0DjC8=.sha256'] // proof-of-key
    }
  }
}
```

Preconditions:
 - Only a member of a fusion identity can tombstone it
 - There is no undo for a tombstone
 - After a tombstone, the only messages which are allowed to extend
   the tangle are other tombstone messages

NOTE:
 - you MUST NOT DM a tombstoned identity
 - tangles can have divergent state (many tips to the graph). We
   consider a tangle tombstoned if any of the tips are a tombstone
   message

## Flow

A flow diagram of what action can follow another. Please note that
multiple invites can happen concurrently. The tangle structure keeps
track of the state of the identity. After a tombstone, the only valid
operation on the identity is attestation.

The `<state> -> tombstone` is left out in the list above because any
state can lead to the tombstone state. Similarly `<state> -> invite`
is also left out.

```
// identity tangle state changes
invite -> consent
invite -> attestation (deny)
consent -> entrust
entrust -> proof-of-key
tombstone -> attestation
```

## Private messages

Private messages uses box2.

FIXME: precise definition. See https://github.com/ssbc/fusion-identity-spec/issues/6

## Operations

Besides the operations operating directly on a fusion identity
described above: init, invite, consent, entrust, proof-of-key,
tombstone, the following operations can be useful:

### read

Gets the current state of a fusion including the status of feeds such
as invited, members, consented.

### invitations

Which fusion identities have I been invited to and not yet responded to.

### all

The current active fusion identies

### tombstoned

All tombstoned fusion identies

## Out of scope for v1

### redirect operations

The purpose of redirects is to make it easy to point from a tombstoned
record to it's replacement.

A redirect is an independant tangle. It is neither part of the old or
new fusion identity. This means there is no causality between the old,
the new and the redirect. Clearly the old and new needs to exist
before a redirect can be created but there is no need for a redirect
to be attested before the new identity can start inviting members.

```js
{
  type: 'fusion/redirect',
  old: ssb:identity/fusion/id1,
  new: ssb:identity/fusion/id2,
  tangles: {
    redirect: {
      root: null,
      previous: null
    }
  }
}
```

```js
{
  type: 'fusion/redirect',
  tombstone: {
    date,
    reason
  },
  tangles: {
    redirect: {
      root,      // the root message of a fusion/redirect tangle
      previous
    }
  }
}
```

It might also be useful to have [authenticated redirects].

NOTE:
- there can be many redirects, which ones you choose to trust are up
  to you (you might like to consider who authored it, and who's
  atteseted it - see below below)
- the only person allowed to tombstone a redirect is the person who
  published it

### attestation

Is the redirect valid?

```js
{
  type: 'fusion/attestation',
  target: %redirectId,                    // static
  position: confirm|reject|null,
  reason: String,                         // optional
  tombstone: { reason }                   // optional
  tangles: {
    attestation: { root: null, previous: null }
  }
}
```

Everyone who agrees with a redirect must attest it publicly because it
makes it harder for an adversary to try and hide the fact that a
identity have been revoked by withholding messages from a single feed.

### Flow


```
// redirect tangle state changes
redirect -> attestation
redirect -> tombstone of redirect?
```

```
// attestation tangle state changs
attestation -> tombstone
```


## Attacks

Someone trying to add fake identities to a fusion just before posting
a tombstone in order for them to appear to have the majority when it
comes to redirecting.

## Related work

SoK: Multi-Device Secure Instant Messaging

https://eprint.iacr.org/2021/498.pdf

Ideas from that one:
 - Use 2 keys instead of one to allow differentiation between devices,
   meaning you could have some devices that you don't trust so much so
   they are only allowed to read messages to the identity, not to add
   members or revoke the identity.

Backchannel

https://www.inkandswitch.com/backchannel/


[SSB]: https://github.com/ssbc/
[box2]: https://github.com/ssbc/private-group-spec
[authenticated redirects]: https://ssb-ngi-pointer.github.io/Audit%20Report_%20Secure%20Scuttlebutt%20Partial%20Replication%20and%20Fusion%20Identity.html#Suggestion-A-Explore-Protocol-Extension-for-Authentication-for-Redirects
