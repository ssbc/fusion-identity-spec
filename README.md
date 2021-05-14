# Fusion Identity Spec

![dance][dancegif]

Purpose: primarily for multi-device

No removal of members, only tombstoning and potentially redirecting to a new

Private messages uses box2.

Format of a fusion id:
 - @sdfasldkrf;skjdf;laksjdf=.fusion1 (proposed)
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

Out of scope for first version:
 - Private invite flow (membership)
 - Can be used in authorisation logic
 - Can be part of private groups

## Operations:

### init

Start the new identity

```js
{
  type: 'fusion/init',
  id: @fusionA,
  tangle: {
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
    @feedDesktop,
  ],
  tangle: {
    fusion: { root: %init, previous: [%init] }
  }
}
```

Only a feed that was either the one that created the fusion identity
or has accepted an invite is allowed to create an invite.

Question:
 - Can we invite a fusion identity?

### consent

Accept the invite

```js
{
  type: 'fusion/consent',
  tangle: {
    fusion: { root: %init, previous: [%invite] }
  }
}
```

### entrust

Give secret key to invited

```js
{
  type: 'fusion/entrust',
  secretKey: KEY,
  tangle: {
    fusion: { root: %init, previous: [%consent] }
  },
  recps: [@fusionA, @feedDesktop]
}
```

The reason to do this as a separate message instead of including it in
the invite is that it makes it more clear if the device has had access
to the key and because this is primarily about multi-device we are in
control of the latency around each leg of this.

### proof of key

A way to publicly announce that you are in possession of private key 

```js
{
  type: fusion/proof-of-key,
  entrustId: %consent,
  proof: sign(%consent + 'fusion/proof-of-key')
  tangles: {
    fusion: { root, previous }
  }
}
```

This step is optional in the protocol

### tombstone / revoke

Nullify the identity

```js
{
  type: 'fusion/tombstone',
  reason: 'Lost @feedPhone, state of key is unknown',
  tangle: {
    fusion: { root: %init, previous: [%consent] }
  }
}
```

To keep things simple, we decided that you can't undo a tombstone
through attestation. 

Attestation in general makes it harder for an adversary to try and
hide the fact that a identity have been revoked.

Question:
 - should we include a list of current member of the identity to make
   reasoning about redirects easier?

### redirect

After a tombstone, what other identity can I use instead?

```js
{
  type: 'fusion/redirect',
  old: @sdfasldkrf;skjdf;laksjdf=.fusion1,  // FusionId
  new: @ldkrf;skssjdfjdf;laksdfa=.fusion1,
  tangle: {
    redirect: { root: null, previous: null }
  }
}
```

A redirect is an independant tangle. It is neither part of the old or
new fusion identity. This means there is no causality between the old,
the new and the redirect. Clearly the old and new needs to exist
before a redirect can be created but there is no need for a redirect
to be attested before the new identity can start inviting members.

For the case where you follow a fusion identity that is then
tombstoned, we need a clear rule to determine what redirect to follow
and at what point we start replicating the feeds of the new fusion
identity instead of the old. One way would be a majority of members in
the old fusion identity needs to attest a redirect. In the case of a
tie, yourself attesting a redirect breaks the tie. 

FIXME: consider a social tiebreaker where 3 of your friends also
breaks a tie?

FIXME: security consideration where a malicious member extends the
tangle branching of the last message before a tombstone and then
starts adding new members in order to gain the majority. Probably
include the members of the fusion identity in the tombstone.

### attestation

Is the redirect valid?

```js
{
  type: 'fusion/attestation',
  target: %redirectId,
  position: confirm|reject|null,
  reason: String, // optional
  tombstone: Tombstone // optional
  tangle: {
    attestation: { root: null, previous: null }
  }
}
```

Everyone who agrees with a redirect must attest it publicly

Question:
 - How do you use this to figure out if a thing (like redirect) is
   valid?

## Flow

A flow diagram of what action can follow another. Please note that
multiple invites can happen concurrently. The tangle structure keeps
track of the state of the identity. After a tombstone, the only valid
operation on the identity is attestation.

The `<state> -> tombstone` is left out in the list above because any
state can lead to the tombstone state. Similarly `<state> -> invite`
is also left out.

```
 invite -> consent
 invite -> attestation (deny)
 consent -> entrust
 entrust -> proof-of-key
 tombstone -> attestation
 tombstone -> redirect (not part of the identity)
 redirect -> attestation
 redirect -> tombstone of redirect?
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

## Related work

SoK: Multi-Device Secure Instant Messaging

https://eprint.iacr.org/2021/498.pdf

Ideas from that one:
 - Use 2 keys instead of one to allow differentiation between devices,
   meaning you could have some devices that you don't trust so much so
   they are only allowed to read messages to the identity, not to add
   members or revoke the identity.

[dancegif]: assets/dance.gif
