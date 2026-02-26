# BRC-20 Single-Step Transfer and Related Changes Proposal v2

### Motivation

Due to historical reasons, BRC-20 does not yet support vindicated inscriptions, which has caused inconvenience for BRC-20-related tooling. Since most of the Ordinals ecosystem no longer distinguishes vindicated inscriptions, and the default inscription format created by common Bitcoin tools is vindicated, it is time to properly enable vindicated inscription support for BRC-20.

Among all inscriptions on the Bitcoin network, BRC-20 `transfer` and `mint` inscriptions constitute the overwhelming majority. These inscriptions are single-use at the BRC-20 protocol level. A user's address may accumulate a large number of such obsolete inscriptions, causing inconvenience during normal use. We propose supporting a "to" field in `mint` and `transfer` inscriptions to specify the recipient, and directly inscribing unbound inscriptions to eliminate the residual impact of these inscriptions as thoroughly and cost-effectively as possible. This also requires BRC-20 to support vindicated inscriptions.

In the BRC-20 protocol, the transfer process generates 3 on-chain transactions. To reduce transfer costs and improve user experience, we have designed a single-step transfer method.

We aim to improve the BRC-20 user experience in three areas:
1. Lower inscribing requirements by supporting vindicated inscriptions during inscribing
2. Reduce inscription-related interference for users by directly unbinding inscriptions
3. Lower transfer costs by reducing the number of transaction steps


### Vindicated Inscription Support

BRC-20 inscriptions currently do not accept Cursed inscriptions or Vindicated inscriptions. We propose largely relaxing this rule — deploy/mint/transfer inscriptions, including single-step transfer inscriptions, will all allow Vindicated inscriptions. However, inscriptions other than single-step transfer (i.e., mint/deploy/legacy-transfer) do not support multiple mints within a single transaction.


### Single-Step Transfer Mechanism

The transaction script for inscribing any inscription almost always includes a public key with a required signature, primarily to prevent others from replacing the inscribing transaction. For example:

```
OP_PUSHBYTES_32 9e2849b90a2353691fccedd467215c88eec89a5d0dcf468e6cf37abed344d746
OP_CHECKSIG
OP_FALSE
OP_IF
OP_PUSHBYTES_3 6f7264
...
OP_ENDIF
```

The public key signature verification within the Taproot script branch is a standard Schnorr signature. We can simply reuse this mechanism by treating the address corresponding to this public key as the source of the available balance locked during BRC-20 transfer inscribing, effectively preventing unauthorized transfer inscribing.

In legacy transfers, the locked transferable balance belongs to the inscription holder's address, which also serves as the source address of the available balance. However, since the public key can already designate the balance source address without relying on the inscription holder's address, the inscription holder's address can instead serve as the new recipient address. This creates a new method of transferring balance in a single step at the time of inscribing, as illustrated below:

![Single Step Transfer Function](./single-step.png)


### Support for the "to" Field in Mint and Single-Step Transfer

In earlier versions of BRC-20, the transfer inscription had a "to" field by design, but it was not utilized by the protocol because it was redundant with the actual inscription recipient address.

To reduce costs and eliminate interference from obsolete inscriptions, it is appropriate to use the "to" field to specify the recipient address in mint inscriptions and single-step transfers that have an explicit signer.

When a mint inscription includes a "to" field, the inscription recipient address is no longer used as the balance recipient. In this case, unbound inscriptions can be used to completely eliminate the impact of inscriptions on any address.

When a single-step transfer inscription includes a "to" field, the single-step transfer behaves similarly to a mint — it transfers the signer's available balance to the address specified in the "to" field as available balance in a single operation, after which the inscription becomes invalid regardless of which address the inscription is sent to. In this case, unbound inscriptions can be used to completely eliminate the impact of inscriptions on any address.

Note that deploy inscriptions do not appear to require the addition of a "to" field.


### Public Key Signature and Corresponding Address in Single-Step Transfer

The Ordinals protocol does not mandate whether an inscription includes a public key and signature. Since the common `OP_PUSHBYTES_32 + OP_CHECKSIG` pattern allows the attached public key signature to fail and can be bypassed to forge any public key, we need to use the stricter `OP_CHECKSIGVERIFY` for signature verification. Additionally, the address type corresponding to the public key must be explicitly specified. Referencing how Bitcoin's three mainstream address types use a single-byte prefix in ScriptPubKey to distinguish address types, we also use a single byte to describe the 8 address types that can be derived from a public key.

We define that the single-step transfer inscription script must begin with the format `OP_PUSHBYTES_32 + OP_CHECKSIGVERIFY + OP_N`, where N denotes the address type:

1. P2TR — empty script path
2. P2WPKH — compressed even public key
3. P2WPKH — compressed odd public key
4. P2PKH — compressed even public key
5. P2PKH — compressed odd public key
6. P2SH-P2WPKH: Nested SegWit — compressed even public key
7. P2SH-P2WPKH: Nested SegWit — compressed odd public key
8. P2TR — key path

For example, the simplest case — a Taproot address via the empty script path — has the following script format:

```
OP_PUSHBYTES_32 9e2849b90a2353691fccedd467215c88eec89a5d0dcf468e6cf37abed344d746
OP_CHECKSIGVERIFY
OP_1
OP_FALSE
OP_IF
...
OP_ENDIF
```

Inscriptions that do not belong to these 8 standard formats are not single-step transfer inscriptions.

Note that deploy/mint inscriptions do not need to support this signature mechanism.

### Activation Rollout

We can first enable single-step transfer for 5-character tickers, then enable it for 4-character BRC-20 tickers. This is much safer. And when we need to enable 4-character tickers, the workload is very small.


### Acknowledgments

Thanks to domo for providing numerous improvement suggestions during the finalization of the single-step transfer design, and thanks to seesharp for pointing out the need to address inscription UTXO bloat.

### Indexer Rules

The indexing rules are listed below:

- If a single-step transfer without a "to" field is minted directly to an `OP_RETURN` output, it is treated as a burn.
- If a single-step transfer without a "to" field is minted directly to a module's receiving address, it supports depositing into the module.
- Module withdrawals will ignore the single-step signature mechanism.
- Deploy/mint inscriptions will ignore the single-step signature mechanism.
- Single-step transfers support batch inscribing (direct vindicated support).
- Mint supports vindicated inscriptions but does not support multiple mints within a single transaction (the first of multiple mints is valid, filtered solely by inscription ID i0).
- Deploy also supports vindicated inscriptions normally, but does not support multiple deploys within a single transaction (the first of multiple deploys is valid, determined solely by inscription ID i0).
- For now, no distinction is made based on whether an address has used single-step transfer; legacy transfer continues to be supported as usual.
- Mint supports adding a "to" field in the inscription JSON to specify the target address.
- Single-step transfer inscriptions support adding a "to" field in the inscription JSON to specify the target address. In this case, the single-step inscription becomes invalid immediately after inscribing — like a mint — regardless of the recipient address. It is recommended to mint as an unbound inscription or mint to an `OP_RETURN` output for destruction.
- If the "to" field is not a valid address, it is treated as a burn. If the "to" field is not of string type or is empty, it is treated as if no "to" field is present.
- A single-step transfer with a "to" field minted directly to a module's receiving address will not deposit into the module; the "to" field takes priority.
- A single-step transfer without a "to" field that is minted to an address different from the signer's address will transfer the balance and then become invalid. (Already implemented in fb.)
- A single-step transfer without a "to" field that is minted to the same address as the signer's address behaves like a traditional transfer inscription — it supports transferring balance by sending the inscription. (This is the fundamental rule of single-step transfer, reiterated here for emphasis.)

regarding inscriptions within a single transaction — we need to emphasize the following:

- Multiple single-step transfer inscriptions inscribed within a single input can only have the same owner.
- Multiple single-step transfer inscriptions inscribed across different inputs can each have their own respective owner.
- Each transfer inscription is executed and validated independently and sequentially, as if they were in separate transactions. A single invalid transfer within a transaction will not invalidate the others. In other words, single-step transfers that have already been determined valid and executed will not be rolled back due to a subsequent transfer failure.


Regarding the ordering of inscription events within the same transaction, we need to define explicit rules rather than directly adopting the current ord output-based inscription ordering, since the output order of inscriptions in ord can be affected by multiple factors:

- Transfer events should always take priority over inscribe events. The current ord implementation is inconsistent in this regard — when multiple inscribes and transfers are mixed, and inscribes can specify a `point`, the two types of events may end up interleaved.
- Inscribe events should follow the construction order (i.e., sequence number) rather than the inscription number order. When a `point` is used to specify a reverse position, the number (and `i0`, `i1`, etc.) may be in reverse relative to the creation order.
- Transfer events should follow the input order of inscriptions (or equivalently, the output order — these two orderings are consistent).
