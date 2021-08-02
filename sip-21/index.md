---
sip: 21
title: "[SIP21] Receipt Identifier"
author: "@lerencao"
sip_type: SDK
category: SDK
status: Alpha
created: 2021-05-06
weight: 21
---

## Receipt Identifier

For communicating account identity of payee, we propose using a compact, versioned and case-insensitive identifier. To meet this criteria, we selected the Bech32 encoding implementation used in Bitcoin Segwit (BIP 0173) excluding the Segwit byte known-length restrictions.

## Desired attributes
- Consistent - Users can build a muscle memory for identifying and using these account addresses
- Atomic - The string identifier feels like a single unit. Users shouldn’t try to separate or truncate the string
- Versioned - The format contains human readable information about how to interpret the payload, preventing subtle errors and reserving space for future identifier schemes
- Error detecting - Bech32 checksums help clients validate input and reduce risk of bad transactions due to mistypings and truncations

## Format

The Receipt Identifier consists of

1. A prefix (also known as hrp (human readable part) which identifies the network version this address is intended for: “stc” for Starcoin Network.
2. A Bech32 delimiter: The character “1” (one)
3. A Bech32 version identifier: The character “p” (version = 1).
4. A Bech32 encoded payload. For version 1, is Starcoin account address (16 bytes)
5. The last 6 characters correspond to the Bech32 checksum

The Receipt Identifier shall not be mixed-cases. It shall be all uppercases, or all lowercases. For example, stc1p8umxurxjd7kwgy058r3f7sgk8qgp509m or STC1P8UMXURXJD7KWGY058R3F7SGK8QGP509M are valid but stc1P8UMXURXJD7KWGY058R3F7SGK8QGP509M is not.

Overall address format: prefix | delimiter | version | encoded payload | checksum

Identifier information

Prefix (string)
Network: stc
Address type (version prefix): 01 (letter p in Bech32 alphabet)
Address payload (in hex)
Address: 0x3f366e0cd26face411f438e29f411638
Checksum: gp509m
Result: stc1p8umxurxjd7kwgy058r3f7sgk8qgp509m

## Looking ahead

In the future, we plan to define additional Receipt Identifier versions to support other forms of identity, such as more compact formats. These would leverage a similar overall structure but would have a different version identifier, preventing naming collisions.


### Basic implementation in Rust


``` rust
    impl AccountAddress {

        pub fn to_bech32(&self) -> String {
            let mut data = self.to_vec().to_base32();
            data.insert(
                0,
                bech32::u5::try_from_u8(1).expect("1 to u8 should success"),
            );
            bech32::encode("stc", data, bech32::Variant::Bech32).expect("bech32 encode should success")
        }

        fn parse_bench32(s: impl AsRef<str>) -> anyhow::Result<Vec<u8>> {
            let (hrp, data, variant) = bech32::decode(s.as_ref())?;

            anyhow::ensure!(variant == bech32::Variant::Bech32, "expect bech32 encoding");
            anyhow::ensure!(hrp.as_str() == "stc", "expect bech32 hrp to be stc");

            let version = data.first().map(|u| u.to_u8());
            anyhow::ensure!(version.filter(|v| *v == 1u8).is_some(), "expect version 1");

            let data: Vec<u8> = bech32::FromBase32::from_base32(&data[1..])?;

            if data.len() == AccountAddress::LENGTH {
                Ok(data)
            } else if data.len() == AccountAddress::LENGTH + 32 {
                // for address + auth key format, just ignore auth key
                Ok(data[0..AccountAddress::LENGTH].to_vec())
            } else {
                anyhow::bail!("Invalid address's length");
            }
        }

        //This method should not be part of the move core type, but can not implement from_str in starcoin project.
        //May be the AccountAddress should not be move core type, move only take care of AddressBytes.
        pub fn from_bech32(s: impl AsRef<str>) -> Result<Self, AccountAddressParseError> {
            Self::from_bytes(Self::parse_bench32(s).map_err(|_| AccountAddressParseError)?)
        }
    }

    #[test]
    fn test_bech32() {
        let hex = "0xca843279e3427144cead5e4d5999a3d0";
        let json_hex = "\"0xca843279e3427144cead5e4d5999a3d0\"";
        let bech32 = "stc1pe2zry70rgfc5fn4dtex4nxdr6qyyuevr";
        let json_bech32 = "\"stc1pe2zry70rgfc5fn4dtex4nxdr6qyyuevr\"";

        let address = AccountAddress::from_str(hex).unwrap();
        let bech32_address = AccountAddress::from_str(bech32).unwrap();
        let json_address: AccountAddress = serde_json::from_str(json_hex).unwrap();
        let json_bech32_address: AccountAddress = serde_json::from_str(json_bech32).unwrap();

        assert_eq!(address, bech32_address);
        assert_eq!(address, json_address);
        assert_eq!(address, json_bech32_address);
    }
  
```