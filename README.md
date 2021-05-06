# Fusion Identity Spec

![dance][dancegif]

Purpose: primarily for multi-device

## Operations:

### init

Start the new identity

```js
{
  type: 'fusion/init',
  id: ????,
  fusionId: @fusionA, // PublicKey
  tangle: {
    fusion: { root: null, previous: null }
  }
}
```

Question:
 - Should the `id` and `fusionId` be different to enable cycling keys?

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

## Related work

SoK: Multi-Device Secure Instant Messaging

https://eprint.iacr.org/2021/498.pdf

Ideas from that one:
 - Use 2 keys instead of one to allow differentiation between devices,
   meaning you could have some devices that you don't trust so much so
   they are only allowed to read messages to the identity, not to add
   members or revoke the identity.

[dancegif]: assets/dance.gif
