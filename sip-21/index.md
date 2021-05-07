---
sip: 21
title: "[SIP21] Account Identifier"
author: "@lerencao"
type: SDK
category: SDK
status: Alpha
created: 2021-05-06
weight: 21
---

## Account identifiers

For communicating account identity, we propose using a compact, versioned and case-insensitive identifier. To meet this criteria, we selected the Bech32 encoding implementation used in Bitcoin Segwit (BIP 0173) excluding the Segwit byte known-length restrictions.

## Desired attributes
- Consistent - Users can build a muscle memory for identifying and using these account addresses
- Atomic - The string identifier feels like a single unit. Users shouldn’t try to separate or truncate the string
- Versioned - The format contains human readable information about how to interpret the payload, preventing subtle errors and reserving space for future identifier schemes
- Error detecting - Bech32 checksums help clients validate input and reduce risk of bad transactions due to mistypings and truncations

## Format

The Starcoin Account Identifier consists of

1. A prefix (also known as hrp (human readable part) which identifies the network version this address is intended for:

- “st” for Mainnet addresses
- "pst" for Pre-Mainnet addresses
- “tst” for Testnet addresses

2. A Bech32 delimiter: The character “1” (one)
3. A Bech32 version identifier: The character “p” (version = 1) for on-chain with subaddress
4. A Bech32 encoded payload. For version 1, is Starcoin account address + auth key (16 + 32 bytes)
5. The last 6 characters correspond to the Bech32 checksum

The Starcoin Account Identifier shall not be mixed-cases. It shall be all uppercases, or all lowercases. For example, st1pu9w0v6vny0hnv2kvhzkh6fwvq5xut42wh8tukg3ra3vy7m6g2al5y4253sm4svf3npwqjevdcssyyse3v94v or ST1PU9W0V6VNY0HNV2KVHZKH6FWVQ5XUT42WH8TUKG3RA3VY7M6G2AL5Y4253SM4SVF3NPWQJEVDCSSYYSE3V94V are valid but st1PU9w0V6vny0HNV2KVHZKH6FWVQ5XUT42WH8TUKG3RA3VY7M6G2AL5Y4253SM4SVF3NPWQJEVDCSSYYSE3V94V is not.

Overall address format: prefix | delimiter | version | encoded payload | checksum

Identifier information

Prefix (string)
Network: st
Address type (version prefix): 01 (letter p in Bech32 alphabet)
Address payload (in hex)
Address: 0x531b12d0eac8845301ece01fcaa2ad18
AuthKey: 0xf7093af50bc7ff2a4b7534a2eeb97c86767bc373e5431fe6e2b411039e3d5e7f
Checksum: 60qyfw
Result: st1p2vd39582ezz9xq0vuq0u4g4drrmsjwh4p0rl72jtw5629m4e0jr8v77rw0j5x8lxu26pzqu78408760qyfw


## Looking ahead

In the future, we plan to define additional Account Identifier versions to support other forms of identity, such as more compact formats. These would leverage a similar overall structure but would have a different version identifier, preventing naming collisions.


### Basic implementation in Rust

``` rust
let mut data = vec![];
data.append(address.to_vec().as_mut());
data.append(auth_key.to_vec().as_mut());

let mut data = data.to_base32();
data.insert(0, bech32::u5::try_from_u8(1).unwrap());
let encoded = bech32::encode("st", data, bech32::Variant::Bech32).unwrap();
println!(
    "address: {}, auth_key: {}, id: {}",
    address, auth_key, encoded
);
```