# Overview

*Tanker* is a solution for implementing client-side encryption in *application*s. It supports end-to-end encryption, and provides an optional way for developers to ensure their *user*s can recover their cryptographic identities in case of *device* loss. This mechanism is referred to as *unlock* and depends on both the *application server* and a third-party verification server to protect and redistribute *user*s' cryptographic identities.

![Tanker big picture](./img/servers.png)

## Design considerations

Special care has been taken to ensure that, from an end-user standpoint, *Tanker* integrates seamlessly in *application*s, despite the challenges involved in doing end-to-end encryption. For developers, we made sure that the SDK is easy-to-use and intuitive, despite the complexity under the hood.

The following considerations were made when designing the solution:

- The encryption schemes must be end-to-end
- Cryptographic materials must be tamper-proof
- The integrity of cryptographic materials delivered by the *Trustchain* is checked client-side; it is not required to trust the Trustchain server
- All pieces of *data* given to *Tanker* must be encrypted with a different key
- The optional *unlock service* must not be able to access *user*s' *data*
- *Group* and *user* keys are rotated when appropriate to support revocation of *user*s and *device*s



## Tanker SDK
The *Tanker SDK* integrates in your *application* and runs where your *application* runs. It is available in Javascript, Java, Objective-C, C, and Python.

It has 3 main functionalities:

- It manages the client's cryptographic identity
- It fetches and verifies Trustchain *block*s to know the cryptographic identities of recipients
- It encrypts and decrypts *data* on-device

### Cryptographic Library Used

We use the Sodium cryptographic library for all our cryptographic needs. We have chosen this library for several reasons:

- It is open-source
- It is well documented
- Its code is well written
- It has a relatively small binary size
- Its algorithms are standard and cover all our needs
- It supports web browsers through [Emscripten](https://emscripten.org)

### Random generator

We use the random generator provided by the Sodium library. The library uses the best implementation available for the platform it runs on:

- Linux's `getrandom()`
- Windows' `RtlGenRandom()`
- WebCrypto's `getRandomValues()` for all web browsers
- Node.js' `crypto.randomBytes()`

### Symmetric Encryption

The *Tanker SDK* uses *libsodium*’s `crypto_aead_xchacha20poly1305_ietf()`.
As its name suggests, it uses the XChaCha20 algorithm with a 256-bits key size, a 192-bits nonce, and a 128-bits MAC size.

### Asymmetric Encryption

The *Tanker SDK* uses *libsodium*'s `crypto_box` functions.
These functions use *X25519*, which is an *ECDH* algorithm over *Curve25519*. It also uses *XSalsa20* and *Poly1305* for symmetric encryption.

### Data Signature

The *Tanker SDK* uses *libsodium*'s `crypto_sign` functions with *Ed25519* keys.

### Data and Password Hashing

For general-purpose hashing, the *Tanker SDK* uses *libsodium*'s `crypto_generichash` function, which uses the *BLAKE2b* hash algorithm.
When *Tanker* needs to hash passwords to store them, it first hashes them on *device* as described above, then hashes them again server-side using the *Argon2* function provided by Golang's `x/crypto`. Subsequently, they are salted with a 32-bits randomly generated salt. *Tanker*'s *Argon2id* parameters are 2 passes on a maximum of 1 thread with a minimum of 32MB memory usage.

### Secure communication layer

All communications between actors are done through HTTPS.
We use the [LibreSSL](http://www.libressl.org/) implementation on mobile and defer to the user-agent for web browsers and Node.js.
Server certificates are verified on all platforms even if, in some cases, we cannot use the ones provided by the platform.
In such occurrences, the *Tanker SDK* comes with the certificates embedded in its binary code.


## Trustchain

The *Trustchain* is an append-only cryptographic log of chained and signed *block*s similar to a Blockchain (because all *block*s are signed by the *Tanker SDK* and linked together).
It is operated by a Trustchain server and responsible for storing and distributing *block*s containing cryptographic materials required by the *Tanker SDK* to work. It is the source of truth for the public keys of all *device*s, *user*s, and *user group*s. The *Tanker SDK* pushes and pulls *block*s from the Trustchain.

There is usually one *Trustchain* per *application*, but it is possible to have more to improve the segmentation of *user*s. An example would be to isolate a big corporate client of an *application* in its own *Trustchain*. *Trustchain*s are completely isolated from each other.

## Unlock

Unlock is the mechanism *Tanker* uses to register new *device*s for a *user*.
It is based on an *unlock key* and has an associated *unlock device*. The *unlock device* is
a virtual *device* associated with a *user* like all other *device*s.

When a *user* signs up in an *application* using *Tanker*, an *unlock key* must be generated. It is possible to ask the *user* to store and remember it, or for ease of use, it can be stored on the *unlock service* with a set of credentials provided by the *user*.

When a *user* signs into an *application* using *Tanker*, the *unlock key* must be provided to validate the new device. It can either be provided directly by the *user* or recovered thought the *unlock service* once the user proves their identity.

### Unlock service

*Tanker* provides an *unlock service* that trades off the end-to-end property for user experience. In these schemes, the *unlock key*
ownership is split between the *application server* and the *Tanker* *unlock service*.

The *unlock service* stores the *unlock key* encrypted by the *user secret*. The latter is stored and distributed by the *application server*. In other words, elements from both the *application server* and the *unlock service* are required to get the *unlock key*.

To get the encrypted *unlock key* from the *unlock service*, the *user* must prove their identity. Currently we support authentication by password and through a verification email. Additional authentication factors are in the works.
