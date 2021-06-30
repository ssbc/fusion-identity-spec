# Fusion Identity Spec v1 (DRAFT)

![dance][dancegif]

Purpose: primarily for multi-device

No removal of members, only tombstoning and potentially redirecting to a new

Private messages uses box2.

Format of a fusion id:
 - @sdfasldkrf;skjdf;laksjdf=.fusion-v1 (proposed)
 - @sdfasldkrf;skjdf;laksjdf=.ed25519 (current identities)
 - %sdfasldkrf;skjdf;laksjdf=.cloaked (private group)

Usages:
 - Can mention a fusion identity and all members can used that as
   notification
 - Can private message a fusion identity and all members will be able
   to decrypt the message and respond
 - Acts as a public record of what feeds are part of an identity
   (discoverability)
 - Following a fusion identity for replication

Out of scope for v1:
 - Private invitations to fusion identity
 - Inviting fusion identities to private groups
 - Use of fusion identity for authorisation logic
     - this is a priority, but will be addressed in subsequent versions
 - invite a fusion identity to an existing fusion identity

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

Later Alice is at a party and looses her rooted phone on the way
home. From this point on she `tombstones` the fusion identity to tell
other peers not to send private messages to the fusion identity any
longer as those messages might be read by a third party.

Later she luckily recovers her phone. She then goes through the same
mechanism as originally to create a new fusion identity.

```
   @laptop                      @phone
   -----------------            -----------------
   init ->
   invite: @phone ->
                                <- consent
   entrust ->
                                proof-of-key
```

This leaves her with 2 fusion identities. Lets say that Bob was
following her old fusion identity. Alice should create a `redirect`
and `attest` the redirect from both devices. This should allow Bobs
client to show what has happened. If Bob agrees that the redirect is
good, he then `attest` the redirect, unfollow the old fusion identity
and follows the new one.

```
   @laptop              @phone              @bob
   ------------         -------------       -------------
   redirect
   attest
                        attest
                                             attest
                                             unfollow
                                             follow
```


Parts:
  - Identity tangle
    - tombstone
      - justification
      - rules
  - Redirect tangle
    - provides more infomation about the evolution of identities
    - redirects are NOT a must have, but make automating some things easier (TODO)
  - Attestation tangle

---

## Fusion Identity Operations:

### init

Start the new identity

```js
{
  type: 'fusion/init',
  id: @fusionA,
  tangles: {
    fusion: { root: null, previous: null }
  }
}
```

To keep things simple the public encryption key for DMs is the same as
the identity.

In order to figure out the members of an identity one needs to look at
all the messages related to the tangle. These messages lives in
different feeds. One can use invites to trace the members or use
tangle sync to receive messages outside your follow scope.

### invite

Add a feed


```js
{
  type: 'fusion/invite',
  invited: [
    @feedDesktop=.ed25519, // FeedId (my desktop)
    @feedLaptop=.ed25519, // FeedId (my phone)
  ],
  tangles: {
    fusion: {
      root: %123asdasdsad=.sha256,  // Init MessageId
      previous: [%init]
    }
  }
}
```

Only a feed that was either the one that created the fusion identity
or has accepted an invite is allowed to create an invite.

NOTE:
- you can only invite other feedId
 

### consent

Accept the invite

```js
{
  type: 'fusion/consent',
  tangles: {
    fusion: {
      root: %init, // Init MessageId
      previous: [
        %12312lk3j12;lk3j123.=sh256,
        %1sadasd3j12;lk3j123.=sh256,
      ]
    }
  }
}
```

### entrust

Give secret key to someone that has consented an invite

Only the one that invited a feed should send the entrust message

```js
{
  type: 'fusion/entrust',
  secretKey: KEY, // make this consistent
  fusionRoot: %init,
  recps: [@fusionA, @feedDesktop]
}
```

NOTE:
- identity init author should entrust the key to themselves
  - otherwise if they have to rebuild their db's they won't have a copy!
- we could have included a private section of the `fusion/invite` which included the key, but decided against this because it makes the flow less clear
- adding an additional step to send the key after consent does not add significant latency because we are presumed to be in control of all the devices in this fusion identity

### proof of key

A way to publicly announce that you are in possession of private key 

```js
{
  type: fusion/proof-of-key,
  consentId: %consent,
  proof: sign(%consent + 'fusion/proof-of-key')
  tangles: {
    fusion: { root, previous }
  }
}
```

This step is optional in the protocol

NOTE:
- we use the MessageId of the `fusion/consent` message because this is a unique publicly auditable record that's part of the tangle (while the entrust is not)


### tombstone / revoke

Nullify the identity.

Given you have a shared private key, there is no easy way to "remove"
a device from a fusion identity.

Our solution is to "tombstone" identities and require you mint a new
fusion identity with the devices you trust. (read on for tools to help
with this transition - redirects + attestation).

```js
{
  type: 'fusion/tombstone',
  reason: 'Lost @feedPhone, state of key is unknown',
  tangles: {
    fusion: {
      root: %init,
      previous: [%consent]
    }
  }
}
```

RULES:
- you cannot undo a tombstone
- once a tombstone has been published, the only messages which are
  allowed to extend the tangle are other tombstone messages
- you MUST NOT DM a tombstoned identity

NOTE:
- tangles can have divergent state (many tips to the graph). We
  consider a tangle tombstoned if any of the tips are a tombstone
  message

## redirect operations

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
  old: @sdfasldkrf;skjdf;laksjdf=.fusion-v1,  // FusionId
  new: @ldkrf;skssjdfjdf;laksdfa=.fusion-v1,
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
 
```
// redirect tangle state changes
redirect -> attestation
redirect -> tombstone of redirect?
```

```
// attestation tangle state changs
attestation -> tombstone
```


## Private messages

Private messages uses box2.

FIXME: precise definition, like: what slot does it use of box2?

## Operations

Ideally we would like methods that:

 - Given a feed id lists all meta feeds this is a member of, and their
   state (including redirects?)
 - Given a fusion identity lists all active and pending members

## Scenarios

FIXME: describe some common scenarios and how they are handled

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

[dancegif]: assets/dance.gif
