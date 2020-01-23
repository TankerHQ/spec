# Tanker: How it works

## Technical specification of the Tanker SDK and protocol for end-to-end encryption

This document is a deep dive into the design of *Tanker Core*, its infrastructure, the cryptographic underpinnings and algorithms, what transits over the protocol, and the security model. It aims to be open and to answer all of the possible questions one might ask when considering adopting this technology.

Note that since *Tanker FileKit* is implemented on top of *Tanker Core*, this specification works for both products.

If you have additional questions that you'd like to see answered, please [open an issue](../../issues/new).

We want to stress that you don't need to understand this spec in order to use the Tanker SDK. Its APIs are simple and require no cryptographic skills. To learn how to integrate the Tanker SDK in your apps, please refer to our [documentation](https://tanker.io/docs).

## Table of contents

* [Overview](overview.md)
* [Concepts](concepts.md)
* [Core features](features.md)
* [Trustchain Design](trustchain_design.md)
* [Protocol](protocol.md)
* [Blocks format](blocks_format.md)
* [Blocks verification rules](blocks_verification.md)
* [Encryption formats](encryption_formats.md)
* [Threat model](threat_model.md)
