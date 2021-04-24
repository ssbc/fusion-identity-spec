# Fusion Identity Spec

Purpose: primarily for multi-device

## Operations:

### init

Start the new identity

### invite

Add a feed

### consent

Accept the invite

### entrust

Give secret key to invited

### tombstone / revoke

Nullify the identity

### redirect

What other identity can I use instead?

### attestation

Is the redirect valid?

## Related work

SoK: Multi-Device Secure Instant Messaging

https://eprint.iacr.org/2021/498.pdf

Ideas from that one:
 - Use 2 keys instead of one to allow differentiation between devices,
   meaning you could have some devices that you don't trust so much so
   they are only allowed to read messages to the identity, not to add
   members or revoke the identity.
