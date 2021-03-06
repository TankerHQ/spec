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
| Version number (2) | Varint          | 1 byte            |
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
| Version number (3) | Varint          | 1 byte            |
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

This is the default format used for simple resource encryption.

### Encryption format v5

#### Spec

| **Element**        | **Buffer type** | **Byte length**   |
| ------------------ | --------------- | ----------------- |
| Version number (5) | Varint          | 1 byte            |
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

Used when encrypting file metadata before upload. We reuse the resource ID generated for the file content so that a single resource ID can be reused for multiple encryptions.

## Chunk encryption

### Encryption format v4

#### Spec

This format aims at providing a stream encryption by splitting the data into chunks. The maximum encrypted chunk size is encoded in the header and is not supposed to always be the same (our current default: 1MB).

Here is the format of an encrypted chunk:

| **Element**                  | **Buffer type**      | **Byte length**    |
| ---------------------------- | -------------------- | ------------------ |
| Version number (4)           | Varint               | 1 byte             |
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

Number of chunks: NC = ceil[ (TCDS + 1) / MECS ]

Total variable overhead: O = CO * NC



When the total **clear** data size is a multiple of the maximum **clear** chunk size, an additional “empty” chunk is added at the end. This ensures that the encrypted data is not truncated by a malicious storage service. Corresponding formula:
TCDS % (MECS - CO) = 0  =>  +1 “empty” chunk of length CO = 61 bytes

The Resource ID is randomly generated once, and is the same in every chunk.

The IV seed is randomly generated for each chunk.

The IV used in encryption is derived from the IV seed and the index of the block with the following formula: IV = H(IV seed, index). The index is concatenated as an uint64 little endian and starts from 0.

This designs avoids using the same Key+IV combination multiple times if we need to support chunk rewriting, while also protecting against a malicious storage service that tries to reorder the chunks.

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

In example #2, we do have: TCDS % (MECS - CO) = 0, hence the third chunk of 61 bytes.

#### Issues

None.

#### Usage

Currently used to encrypt clear data with byte length >= 1 MB (e.g. big files).
