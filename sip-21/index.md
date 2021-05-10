---
sip: 21
title: "[SIP21] Receipt Identifier"
author: "@lerencao"
type: SDK
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
4. A Bech32 encoded payload. For version 1, is Starcoin account address + optional auth key (16 + 32 bytes)
5. The last 6 characters correspond to the Bech32 checksum

The Receipt Identifier shall not be mixed-cases. It shall be all uppercases, or all lowercases. For example, st1pu9w0v6vny0hnv2kvhzkh6fwvq5xut42wh8tukg3ra3vy7m6g2al5y4253sm4svf3npwqjevdcssyyse3v94v or ST1PU9W0V6VNY0HNV2KVHZKH6FWVQ5XUT42WH8TUKG3RA3VY7M6G2AL5Y4253SM4SVF3NPWQJEVDCSSYYSE3V94V are valid but st1PU9w0V6vny0HNV2KVHZKH6FWVQ5XUT42WH8TUKG3RA3VY7M6G2AL5Y4253SM4SVF3NPWQJEVDCSSYYSE3V94V is not.

Overall address format: prefix | delimiter | version | encoded payload | checksum

Identifier information

Prefix (string)
Network: stc
Address type (version prefix): 01 (letter p in Bech32 alphabet)
Address payload (in hex)
Address: 0x1603d10ce8649663e4e5a757a8681833
AuthKey: 0x93dcc435cfca2dcf3bf44e9948f1f6a98e66a1f1b114a4b8a37ea16e12beeb6d
Checksum: 3mmwta
Result: stc1pzcpazr8gvjtx8e895at6s6qcxwfae3p4el9zmnem738fjj83765cue4p7xc3ff9c5dl2zmsjhm4k63mmwta

## Looking ahead

In the future, we plan to define additional Receipt Identifier versions to support other forms of identity, such as more compact formats. These would leverage a similar overall structure but would have a different version identifier, preventing naming collisions.


### Basic implementation in Rust


``` rust
    #[derive(Copy, Clone, Debug)]
    pub enum ReceiptIdentifier {
        V1(AccountAddress, Option<AuthenticationKey>),
    }

    impl FromStr for ReceiptIdentifier {
        type Err = anyhow::Error;

        fn from_str(s: &str) -> Result<Self, Self::Err> {
            Self::decode(s)
        }
    }
    impl std::fmt::Display for ReceiptIdentifier {
        fn fmt(&self, f: &mut Formatter<'_>) -> std::fmt::Result {
            write!(f, "{}", self.encode())
        }
    }

    impl ReceiptIdentifier {
        pub fn encode(&self) -> String {
            match self {
                ReceiptIdentifier::V1(address, auth_key) => {
                    let mut data = vec![];
                    data.append(address.to_vec().as_mut());
                    if let Some(auth_key) = auth_key {
                        data.append(auth_key.to_vec().as_mut());
                    }

                    let mut data = data.to_base32();
                    data.insert(0, bech32::u5::try_from_u8(1).unwrap());
                    bech32::encode("stc", data, bech32::Variant::Bech32).unwrap()
                }
            }
        }
        pub fn decode(s: impl AsRef<str>) -> Result<ReceiptIdentifier> {
            let (hrp, data, variant) = bech32::decode(s.as_ref()).unwrap();

            anyhow::ensure!(variant == bech32::Variant::Bech32, "expect bech32 encoding");
            anyhow::ensure!(hrp.as_str() == "stc", "expect bech32 hrp to be stc");

            let version = data.first().map(|u| u.to_u8());
            anyhow::ensure!(version.filter(|v| *v == 1u8).is_some(), "expect version 1");

            let data: Vec<u8> = bech32::FromBase32::from_base32(&data[1..])?;

            let (address, auth_key) = if data.len() == AccountAddress::LENGTH {
                (AccountAddress::from_bytes(data.as_slice())?, None)
            } else if data.len() == AccountAddress::LENGTH + AuthenticationKey::LENGTH {
                let address = AccountAddress::from_bytes(&data[0..AccountAddress::LENGTH])?;
                let auth_key = AuthenticationKey::try_from(&data[AccountAddress::LENGTH..])?;
                (address, Some(auth_key))
            } else {
                anyhow::bail!("invalid data");
            };
            Ok(ReceiptIdentifier::V1(address, auth_key))
        }
    }
    #[test]
    pub fn test_rust_bench32() {
        let address = AccountAddress::random();
        let auth_key = AuthenticationKey::random();

        let encoded = ReceiptIdentifier::V1(address, Some(auth_key)).to_string();
        println!(
            "address: {}, auth_key: {}, id: {}",
            address, auth_key, &encoded
        );

        let id = ReceiptIdentifier::from_str(encoded.as_str()).unwrap();
        match id {
            ReceiptIdentifier::V1(decoded_address, decoded_auth_key) => {
                assert_eq!(decoded_address, address);
                assert_eq!(decoded_auth_key, Some(auth_key));
            }
        }
    }    
```