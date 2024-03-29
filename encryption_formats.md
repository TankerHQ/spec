# Encryption Formats

This document details the different encryption format used at Tanker:

Note that encrypted data emitted is always bigger than the clear data provided. The overhead is due to:

- the format metadata (e.g. version)
- the MAC (Message Authentication Code): adds bytes to guarantee the integrity of the encrypted data
- the IV (Initialisation Vector): a random vector that guarantees that no information can be deduced from the encrypted data about the clear data (semantic security).

## Simple encryption

### Encryption format v1

#### Spec

| **Element**        | **Buffer type** | **Byte length**   |
| ------------------ | --------------- | ----------------- |
| Version number (1) | Varint          | 1 byte            |
| Encrypted data     | Variable length | = clear data size |
| MAC                | Fixed length    | 16 bytes          |
| IV                 | Fixed length    | 24 bytes          |

#### Properties

Constant overhead: O = 41 bytes

#### Issues

Integrity issue: the v1 format was badly designed as we used to read the last 16 bytes as the “resource ID” although theses bytes were not the MAC but part of the IV. As a result the “resource ID” did not depend of the resource content itself.

#### Usage

Due to the issue described above, this format is not used to encrypt resources anymore (superseded by v3 format). It is still used in the JS SDK to encrypt data at rest: device keysafe, cache of resource keys.

### Encryption format v2

#### Spec

| **Element**        | **Buffer type** | **Byte length**   |
| ------------------ | --------------- | ----------------- |
| Version number (2) | Fixed length    | 1 byte            |
| IV                 | Fixed length    | 24 bytes          |
| Encrypted data     | Variable length | = clear data size |
| MAC                | Fixed length    | 16 bytes          |

#### Properties

Constant overhead: O = 41 bytes

#### Issues

Compactness issue: when used to encrypt a resource, where the key is “always” rotated, there would be no need to add a random IV to be secure. So we kind of waste 24 bytes…

#### Usage

Due to the issue described above, this format is not used to encrypt resources anymore (superseded by v3 format).

It is still used in the JS SDK when the same key is used to encrypt multiple times, i.e. :

- encrypt data at rest: group keys
- encrypt / decrypt the virtual devices used for account recovery
- encrypt the email address when using the email verification method

### Encryption format v3

#### Spec

| **Element**        | **Buffer type** | **Byte length**   |
| ------------------ | --------------- | ----------------- |
| Version number (3) | Fixed length    | 1 byte            |
| Encrypted data     | Variable length | = clear data size |
| MAC                | Fixed length    | 16 bytes          |

This format aims at providing the shortest encryption overhead possible.

It is safe for use in “one-time encryption” situation.

If you need to reuse the same encryption key, use v2 format instead.

#### Properties

Constant overhead: O = 17 bytes

IV = hash(encryption key)  *(i.e. the IV is not stored with the encrypted data but computed)*

**Issues**

None (but don’t use the same key more than once).

#### Usage

This format was used instead of v6 when the padding option was disabled.
It has been superseded by the v9 format.

### Encryption format v5

#### Spec

| **Element**        | **Buffer type** | **Byte length**   |
| ------------------ | --------------- | ----------------- |
| Version number (5) | Fixed length    | 1 byte            |
| Resource ID        | Fixed length    | 16 bytes          |
| IV                 | Fixed length    | 24 bytes          |
| Encrypted data     | Variable length | = clear data size |
| MAC                | Fixed length    | 16 bytes          |

This format has been designed for simple resource encryption requiring a “stable” resource ID.

#### Properties

Constant overhead: O = 57 bytes

#### Issues

None.

#### Usage

Used instead of format v7 when padding is disabled.

### Encryption format v6

#### Spec

| **Element**        | **Buffer type** | **Byte length**             |
|--------------------|-----------------|-----------------------------|
| Version number (6) | Fixed length    | 1 byte                      |
| Encrypted data     | Variable length | = clear data size + padding |
| MAC                | Fixed length    | 16 bytes                    |

This format is the same as *Encryption format v3* except some padding bytes are added to the clear data before encryption.
This addition is intended to prevent the leak of information about the data size once encrypted.

The number of padding bytes to add is computed using [the PADME algorithm](https://lbarman.ch/blog/padme/).

The implementation of padding follows the [ISO/IEC 7816-4](https://en.wikipedia.org/wiki/Padding_(cryptography)#ISO/IEC_7816-4) standard.
Let *N* be the size of the clear data and *P* the size of the clear data plus its padding. This standard consists of adding a `0x80` byte and the `0x00` bytes until *P* is reached. For example, to pad `true` to *P* = 8 bytes we do `7472 7565 8000 0000`. In addition, PADME results being unsatisfying for *N* < 10 bytes, this format has a minimal padding size of 10 (so `true` actually gets padded to `7472 7565 8000 0000 0000`).

#### Issues

None.

#### Properties

Overhead: O = *padme_overhead(N + 1)* + 17b with  
*padme_overhead(N)* = 10 bytes if *N* < 10b  
*padme_overhead(N)* < *N* + *N* * 12/100 if 10b < *N* < 1Mb  
*padme_overhead(N)* < *N* + *N* * 6/100 if 1Mb < *N* < 1Gb  
*padme_overhead(N)* < *N* + *N* * 3/100 if *N* > 1Gb

#### Usage

This used to be the simple resource encryption format, which was safe for use in “one-time encryption” situation.
It has been superseded by the v10 format.

### Encryption format v7

#### Spec

| **Element**        | **Buffer type** | **Byte length**             |
|--------------------|-----------------|-----------------------------|
| Version number (7) | Fixed length    | 1 byte                      |
| Resource ID        | Fixed length    | 16 bytes                    |
| IV                 | Fixed length    | 24 bytes                    |
| Encrypted data     | Variable length | = clear data size + padding |
| MAC                | Fixed length    | 16 bytes                    |

This format is the same as v5 but with the addition of padding. The padding algorithm and overhead is the same as v6.

#### Properties

Constant overhead: O = *padme_overhead(N + 1)* + 57 bytes

*padme_overhead(N)* is detailed in the v6 section.

#### Issues

None.

#### Usage

Default format for manually created encryption sessions. Used when encrypting file metadata before upload. We reuse the resource ID generated for the file content so that a single resource ID can be reused for multiple encryptions.

### Encryption format v9

#### Spec

| **Element**        | **Buffer type** | **Byte length**   |
|--------------------| --------------- | ----------------- |
| Version number (9) | Fixed length    | 1 byte            |
| Session ID         | Fixed length    | 16 bytes          |
| Resource ID / Seed | Fixed length    | 16 bytes          |
| Encrypted data     | Variable length | = clear data size |
| MAC                | Fixed length    | 16 bytes          |

This format has been designed for simple resource encryption within an encryption session.

#### Properties

Constant overhead: O = 49 bytes

#### Issues

None.

#### Usage

Used instead of format v10 when padding is disabled.

### Encryption format v10

#### Spec

| **Element**         | **Buffer type** | **Byte length**             |
|---------------------| --------------- |-----------------------------|
| Version number (10) | Fixed length    | 1 byte                      |
| Session ID          | Fixed length    | 16 bytes                    |
| Resource ID / Seed  | Fixed length    | 16 bytes                    |
| Encrypted data      | Variable length | = clear data size + padding |
| MAC                 | Fixed length    | 16 bytes                    |

This format has been designed for simple resource encryption within an encryption session.
It is the same as v9 but with the addition of padding. The padding algorithm and overhead is the same as v6.

#### Properties

Constant overhead: O = 49 bytes

Padding overhead: O = padme_overhead(N + 1) bytes
*padme_overhead(N)* is detailed in the v6 section.

#### Issues

None.

#### Usage

It is used by default for simple resource encryption, when not using a manually created session and when padding is not disabled.
The session used when encrypting with this format is created automatically (transparent session).

## Chunk encryption

### Encryption format v4

#### Spec

This format aims at providing a stream encryption by splitting the data into chunks. The maximum encrypted chunk size is encoded in the header and is not supposed to always be the same (our current default: 1MB).

Here is the format of an encrypted chunk:

| **Element**                  | **Buffer type**      | **Byte length**    |
| ---------------------------- | -------------------- | ------------------ |
| Version number (4)           | Fixed length         | 1 byte             |
| Maximum encrypted chunk size | Uint32 little endian | 4 bytes            |
| Resource ID                  | Fixed length         | 16 bytes           |
| IV seed                      | Fixed length         | 24 bytes           |
| Encrypted data               | Variable length      | = clear chunk size |
| MAC                          | Fixed length         | 16 bytes           |



All encrypted chunks have the same size (= maximum encrypted chunk size), except the last one which has always a shorter size.

#### Properties

Total clear data size: TCDS (variable)

Constant overhead per chunk: CO = 61 bytes

Maximum encrypted chunk size: MECS = variable (default value: 1 MB)

Maximum clear chunk size: MCCS = MECS - CO (default value: 1 MB - 61 bytes)

Number of chunks: NC = ceil[ (TCDS + 1) / MCCS ]

Total variable overhead: O = CO * NC



When the total **clear** data size is a multiple of the maximum **clear** chunk size, an additional “empty” chunk is added at the end. This ensures that the encrypted data is not truncated by a malicious storage service. Corresponding formula:
TCDS % (MECS - CO) = 0  =>  +1 “empty” chunk of length CO = 61 bytes

The Resource ID is randomly generated once, and is the same in every chunk.

The IV seed is randomly generated for each chunk.

The IV used in encryption is derived from the IV seed and the index of the block with the following formula: IV = H(IV seed, index). The index is concatenated as an uint64 little endian and starts from 0.

This design avoids using the same Key+IV combination multiple times if we need to support chunk rewriting, while also protecting against a malicious storage service that tries to reorder the chunks.

The Key is not derived and stays the same for all the chunks.

#### Examples

| **Properties**               | **Example #1**                      | **Example #2**                    |
| ---------------------------- | ----------------------------------- | --------------------------------- |
| Total clear data size        | 2.5 MB                              | 4 MB - 2 * 61 B                   |
| Maximum encrypted chunk size | 1 MB                                | 2 MB                              |
| Number of chunks             | 3                                   | 3                                 |
| Clear chunk sizes            | 1 MB - 61 B \| 1MB - 61 B \| 0.5 MB | 2 MB - 61 B \| 2 MB - 61 B \| 0 B |
| Encrypted chunk sizes        | 1 MB \| 1 MB \| 0.5 MB + 183 B      | 2 MB \| 2 MB \| 61 B              |
| Total overhead               | 3 * 61 = 183 B                      | 3 * 61 = 183 B                    |

In example #2, we do have: TCDS % (MCCS) = 0, hence the third chunk of 61 bytes.

#### Issues

None.

#### Usage

Used instead of format v8 when padding is disabled.

### Encryption format v8

#### Spec

This format aims at providing a stream encryption by splitting the data into chunks. The maximum encrypted chunk size is encoded in the header and is not supposed to always be the same (our current default: 1MB).

Here is the format of an encrypted chunk:

| **Element**                  | **Buffer type**      | **Byte length**              |
| ---------------------------- | -------------------- | ---------------------------- |
| Version number (8)           | Fixed length         | 1 byte                       |
| Maximum encrypted chunk size | Uint32 little endian | 4 bytes                      |
| Resource ID                  | Fixed length         | 16 bytes                     |
| IV seed                      | Fixed length         | 24 bytes                     |
| Encrypted data               | Variable length      | = clear chunk size + padding |
| MAC                          | Fixed length         | 16 bytes                     |



All encrypted chunks have the same size (= maximum encrypted chunk size), except the last one which has always a shorter size.

#### Properties

Total clear data size: TCDS (variable)

Constant overhead per chunk: CO = 62 bytes

Padding overhead: PO = *padme_overhead(TCDS + 1)*

*padme_overhead(N)* is detailed in the v6 section.

Maximum encrypted chunk size: MECS = variable (default value: 1 MB)

Maximum clear chunk size: MCCS = MECS - CO (default value: 1 MB - 62 bytes)

Number of chunks: NC = ceil[ (TCDS + PO + 1) / MCCS ]

Total variable overhead: O = PO + CO * NC



When the total **clear** data size + padding overhead is a multiple of the maximum **clear** chunk size, an additional “empty” chunk is added at the end. This ensures that the encrypted data is not truncated by a malicious storage service. Corresponding formula:
(TCDS + PO) % (MCCS) = 0  =>  +1 “empty” chunk of length CO = 62 bytes

The Resource ID is randomly generated once, and is the same in every chunk.

The IV seed is randomly generated for each chunk.

The IV used in encryption is derived from the IV seed and the index of the block with the following formula: IV = H(IV seed, index). The index is concatenated as an uint64 little endian and starts from 0.

This design avoids using the same Key+IV combination multiple times if we need to support chunk rewriting, while also protecting against a malicious storage service that tries to reorder the chunks.

The Key is not derived and stays the same for all the chunks.

#### Usage

Currently used to encrypt clear data with byte length >= 1 MB (e.g. big files) when using manual encryption sessions.

### Encryption format v11

#### Spec

This format aims at providing a stream encryption by splitting the data into chunks. The maximum encrypted chunk size is encoded in the header and is not supposed to always be the same (our current default: 1MB).

Here is the format of an encrypted chunk:


| **Element**          | **Buffer type**      | **Byte length**              |
|----------------------|----------------------|------------------------------|
| Version number (11)  | Fixed length         | 1 byte                       |
| Session ID           | Fixed length         | 16 bytes                     |
| Resource ID / Seed   | Fixed length         | 16 bytes                     |
| Encrypted chunk size | Uint32 little endian | 4 bytes                      |
| Chunk padding size   | Uint32 little endian | 4 bytes                      |
| Encrypted chunk data | Variable length      | = clear chunk size + padding |
| Encrypted chunk MAC  | Fixed length         | 16 bytes                     |

All encrypted chunks have the same size (= encrypted chunk size), except the last one which is always shorter.

#### Properties

Total clear data size: TCDS (variable)

Header overhead: HO = 37 bytes

Overhead per chunk: CO = 20 bytes

Padding overhead: PO = *padme_overhead(TCDS)*

*padme_overhead(N)* is detailed in the v6 section.

Encrypted chunk size: ECS = variable (default value: 1 MB)

Clear chunk size: CCS = ECS - CO (default value: 1 MB - 20 bytes)

Number of chunks: NC = ceil[ (TCDS + PO) / CCS ]

Total variable overhead: O = H0 + PO + CO * NC


When the total **clear** data size + padding overhead is a multiple of the maximum **clear** chunk size, an additional “empty” chunk is added at the end. This ensures that the encrypted data is not truncated by a malicious storage service. Corresponding formula:
(TCDS + PO) % (CCS) = 0  =>  +1 “empty” chunk of length CO = 20 bytes

The Resource ID is randomly generated once, and is also the seed used to derive the individual encryption key of this resource.

The IV used in encryption is derived from the session ID and the index of the chunk with the following formula: IV = H(session ID, index). The index is concatenated as an uint64 little endian and starts from 0.

This design avoids using the same Key+IV combination multiple times, while also protecting against a malicious storage service that tries to reorder the chunks.

The Key is derived from the Resource ID and the key of the encryption session, and is the same for all chunks.

#### Usage

Currently used to encrypt clear data with byte length >= 1 MB (e.g. big files) when *not* using manual encryption sessions.
This format is used even when padding is off, as the overhead of the format remains small either way.
