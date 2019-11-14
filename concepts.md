# Concepts

[Tanker App]: concepts.md#tanker-app "An application created in the Tanker dashboard"
[Trustchain]: concepts.md#trustchain "A Trustchain is a collection of signed blocks, attached to a given app"
[Device Encryption Key Pair]: concepts.md#device-keys "Used to encrypt the user keys"
[Device ID]: concepts.md#device-id "Unique identifier of a device belonging to a user"
[Device Signature Key Pair]: concepts.md#device-keys "Used when the user signs a block"
[Group Encryption Key Pair]: concepts.md#user-group-keys "Used when sharing data securely within a group"
[Group Signature Key Pair]: concepts.md#user-group-keys "Used when the user modifies a group"
[Local Encrypted Storage]: concepts.md#device-id "A place where key materials are stored, encrypted at rest while the Tanker session is closed"
[Resource Encryption Key]: concepts.md#resource-keys "A symmetric key that can be exchanged securely across users"
[Shared Encrypted Key]: concepts.md#resource-keys "The result of encrypting a Resource Encryption Key for a recipient"
[Trustchain Signature Key Pair]: concepts.md#trustchain-keys "Root of the Trustchain - used to sign user additions"
[User Encryption Key Pair]: concepts.md#user-keys "Used for sharing encrypted keys across users"
[User ID]: concepts.md#user-id "Unique identifier of a user"
[Verification Key]: concepts.md#verification-key "An opaque token that allows creating new devices"
[User Secret]: concepts.md#user-secret "A secret generated and stored on the application server that protects the local encrypted storage"
[Secret Permanent Identity]: concepts.md#secret-permanent-identity "An opaque string containing private data about user's identity"
[Public Permanent Identity]: concepts.md#public-permanent-identity "Generated from a Secret Permanent Identity - essentialy equivalent to a user ID"
[Secret Provisional Identity]: concepts.md#secret-provisional-identity "Same as Secret Permanent Identity, but for a user not registered on the Trustchain yet"
[Public Provisional Identity]: concepts.md#public-provisional-identity "Same as Public Permanent Identity, but for a user not registered on the Trustchain yet"

*Tanker Core*'s security is based on the separation of knowledge, between the *Tanker server*, the *user*, and the *application server*.
To establish trust between these actors and to enable sharing of encrypted *data* between *user*s, *Tanker Core* produces and uses cryptographic keys, IDs, and tokens.
The following section describes these elements, how they are generated, used, and, when applicable, stored.

It should be noted that they are only valid within a single *Trustchain*.

![IDs and keys ownership](./img/keys.png)

For a glossary of global terms used in this spec, please see [the relevant section](#glossary).

## Concepts index

Here's a list of concepts used in the rest of this document:

<dl>
 <dt><a href="#tanker-app">Tanker App</a></dt>
  <dd>An application created in the Tanker dashboard</dd>

 <dt><a href="#trustchain">Trustchain</a></dt>
  <dd>A Trustchain is a collection of signed blocks, attached to a given app<dd>

 <dt><a href="#user-id">User ID (UID)</a></dt>
  <dd>Unique identifier of a user</dd>

 <dt><a href="#device-id">Device ID (DID)</a></dt>
 <dd>Unique identifier of a device belonging to a user</dd>

 <dt><a href="#user-secret">User Secret (US)</a></dt>
 <dd>A secret generated and stored on the application server that protects the local encrypted storage</dd>

 <dt><a href="#device-keys">Device Encryption Key Pair (DEK)</a></dt>
 <dd>Used to encrypt the user keys</dd>

 <dt><a href="#device-keys">Device Signature Key Pair (DSK)</a></dt>
 <dd>Used when the user signs a block - like the addition of a new device</dd>

 <dt><a href="#user-group-keys">Group Encryption Key Pair (GEK)</a></dt>
 <dd>Used when sharing data securely within a group</dd>

 <dt><a href="#user-group-keys">Group Signature Key Pair (GSK)</a></dt>
 <dd>Used when the user modifies a group - like adding a new member to it</dd>

 <dt><a href="#device-keys">Local Encrypted Storage (LES)</a></dt>
 <dd>A place where key materials are stored, encrypted at rest while the Tanker session is closed</dd>

 <dt><a href="#resource-keys">Resource Encryption Key (REK)</a></dt>
 <dd>A symmetric key that can be exchanged securely across users</dd>

 <dt><a href="#resource-keys">Shared Encrypted Key (SEK)</a></dt>
 <dd>The result of encrypting a Resource Encryption Key for a recipient</dd>

 <dt><a href="#trustchain-keys">Trustchain Signature Key Pair (TSK)</a></dt>
 <dd>Root of the Trustchain - used to sign user additions</dd>

 <dt><a href="#user-keys">User Encryption Key Pair (UEK)</a></dt>
 <dd>Used for sharing encrypted keys across users. It is rotated when a device is revoked</dd>

 <dt><a href="#verification-key">Verification Key (VFK)</a></dt>
 <dd>An opaque token that allows creating new devices</dd>

 <dt><a href="#secret-provisional-identity">Secret Permanent Identity (SPerID)</a></dt>
 <dd>An opaque string containing private data about user's identity</dd>

 <dt><a href="#public-permanent-identity">Public Permanent Identity (PPerID)</a></dt>
 <dd>Generated from a Secret Permanent Identity - essentialy equivalent to a user ID</dd>

 <dt><a href="#secret-provisional-identity">Secret Provisional Identity (SProID)</a></dt>
 <dd>Same as Secret Permanent Identity, but for a user not registered on the Trustchain yet</dd>

 <dt><a href="#public-provisional-identity">Public Provisional Identity (PProID)</a></dt>
 <dd>Same as Public Permanent Identity, but for a user not registered on the Trustchain yet</dd>

 <dt><a href="#resource-id">Resource ID</a></dt>
 <dd>The unique ID of an encrypted data</dd>
</dl>

They are explained in more detail below.

### Tanker app

An app is created in the <a href="https://dashboard.tanker.io">Tanker dashboard</a> and managed by its creator.
Creating an app gives a *app secret* and an *app ID*.

### Trustchain

A *Trustchain* is an append-only cryptographic log of chained and signed *block*s similar to a Blockchain (because all *block*s are signed by *Tanker Core* and linked together).
It is operated by a Trustchain server and responsible for storing and distributing *block*s containing cryptographic materials required by *Tanker Core* to work. It is the source of truth for the public keys of all *device*s, *user*s, and *user group*s. *Tanker Core* pushes and pulls *block*s from the Trustchain.

### Trustchain keys

A *Trustchain* is identified by a unique ID, a name, and a [Trustchain Signature Key Pair].
The name is informative only.

The [Trustchain Signature Key Pair] is generated client-side by the *customer* during the *Trustchain* creation. The Private Signature Key is called *app secret* in the dashboard.

As such, it is only known by the *customer* and cannot be recovered by *Tanker* in any way - except if the "test mode" is enabled.

The public part of the [Trustchain Signature Key Pair] is included in the *Trustchain*'s root *block*.
The *Trustchain* ID is actually the hash of the *Trustchain* root *block* and is named *app ID* in the dashboard.

### User ID

The [User ID] identifies a *user* on the *Trustchain*.
It is provided by the *application* when the *user* is created, and is part of the user's [Secret Permanent Identity].
*Tanker Core* cryptographically hashes *User ID*s locally before sending them to the *Tanker server*.

### Tanker Session

The Tanker session must be opened for the [Local Encrypted Storage] to be accessible. Once it is closed, no other program running on the device
can access it.

### User secret

The [User Secret] is a symmetric key, randomly generated by the *application server* for each *user*.
It is part of the [Secret Permanent Identity], and is considered an implementation detail not exposed by the Tanker API.
A user secret cannot be changed once the *user*'s first *device* has been created.
It is used to encrypt and decrypt the [Local Encrypted Storage] and the [Verification Key] stored by the *identity verification service*.

### Delegation token

A delegation token is a proof of the *user*'s authentication with the *application server*. It is generated when the *user* opens their first session with *Tanker Core*.
The delegation token is composed of an ephemeral signature key pair, a delegation signature, and the [User ID].
The delegation signature is created by combining the ephemeral public key and the [User ID], then signing the result with the private [Trustchain Signature Key Pair].
The delegation token is only used when the *user* creates their first *device* on the *Trustchain*, to sign the first `device_creation` *block* of that *user*.

### Secret Permanent Identity

The [Secret Permanent Identity] is generated and stored by the *application server* and provided to a user only after successful authentication against the *application server*.
It should never be shared with other *user*s.
It contains some secret key material such as the [User Secret] and [delegation token](#delegation-token).
It represents the identity of the *user* for *Tanker Core* and is considered a proof of authentication against the *application server*.

### Public Permanent Identity

A [Public Permanent Identity] can be generated from a [Secret Permanent Identity], and is used to uniquely identify a *user*.
It contains a [User ID], but no secret key material and it is safe to share publicly.

### Device ID

Each *user* must have at least one *device*. *Device*s are identified in *Tanker Core* by a randomly attributed [Device ID]. Each *device* has a [Local Encrypted Storage].

### Device keys

Each *device* registered on the *Trustchain* has one [Device Encryption Key Pair] and one [Device Signature Key Pair].
Device keys are stored in the *device*'s [Local Encrypted Storage]. They are never replaced or modified after creation.
The public [Device Encryption Key Pair] and [Device Signature Key Pair] are pushed to the *Trustchain* in the `device_creation` *block*.
The private [Device Encryption Key Pair] and [Device Signature Key Pair] never leave the *device*.

### User keys

Every *user* registered on the *Trustchain* has one active [User Encryption Key Pair].
User keys are stored in each *device*'s [Local Encrypted Storage].
The Private [User Encryption Key Pair] is encrypted with each of the *user*'s *device*s' public [Device Encryption Key Pair] before being pushed to the *Trustchain*.
It is pushed to the *Trustchain* in the `device_creation` *block* and updated whenever a *device* is revoked.

### User group keys

A *user group* has one [Group Encryption Key Pair] and one [Group Signature Key Pair].
*User group* keys are stored in the *device*'s [Local Encrypted Storage].
The private [Group Signature Key Pair] is encrypted with the private [Group Encryption Key Pair], which is encrypted with each *group member*'s [User Encryption Key Pair].
They are pushed to the *Trustchain* in the `user_group_creation` *block* and updated whenever a *group member* is removed from a *user group*.

### Resource keys

A new [Resource Encryption Key] is randomly generated each time a *user* encrypts *data*.
The *data* is symmetrically encrypted with the [Resource Encryption Key].
The [Resource Encryption Key] can be encrypted for *user*s or *user group*s.
When sharing a resource key with a *user*, the [Resource Encryption Key] is encrypted using the [User Encryption Key Pair] of that *user* creating a [Shared Encrypted Key].
When sharing a resource key with a *user group*, the [Resource Encryption Key] is encrypted using the [Group Encryption Key Pair] of that *user group* creating a [Shared Encrypted Key].
[Shared Encrypted Key]s are pushed to the *Trustchain* in `key_publish` *block*s.
When received by a *device*, they are stored in the [Local Encrypted Storage].

### Verification key

The [Verification Key] is used to register new devices in a secure manner. See the [protocol documentation](protocol.md) for more details.

### Secret Provisional Identity

A [Secret Provisional Identity] represents the identity of a user that is not yet registered on Tanker. It is split into two halves that are stored on the *application server* and on *Tanker server*s.
A provisional identity is attached to some authentication methods. For the moment, email and OpenID connect `IDToken`s are supported.
Each half contains the name of the authentication method, a value (in case of email, the value is the email address), an encryption key pair and a signature key pair.

### Public Provisional Identity

A [Public Provisional Identity] can be generated from a [Secret Provisional Identity] and consists of the [Secret Provisional Identity] without its secrets parts.

### TLS

Transport Layer Security is always used between *Tanker Core* and *Tanker Server* to exchange information.
This guarantees the confidentiality and authenticity of the data exchanged.

### Resource ID

The unique ID part of an encrypted data.
This ID is used by *Tanker core* to associate encrypted data with their encryption key.

## Glossary

<dl>
  <dt>application</dt>
  <dd>any application, web, desktop, or mobile, that uses <em>Tanker Core</em> to protect <em>data</em> for its <em>user</em>s</dd>

  <dt>application server</dt>
  <dd>server supporting the <em>application</em>, responsible for the <em>application</em>'s business logic, <em>data</em> storage, and user authentication</dd>

  <dt>block</dt>
  <dd>an atomic element of a <em>Trustchain</em>, signed by its <em>device</em> author and linked to other <em>block</em>s</dd>

  <dt>customer</dt>
  <dd><em>Tanker</em>'s customer, the owner of the <em>application</em>, served to its <em>user</em>s</dd>

  <dt>data</dt>
  <dd>any data (including files), owned by the <em>user</em>, stored by the <em>customer</em> to be used in the <em>application</em></dd>

  <dt>device</dt>
  <dd>a web browser, mobile device, or computer on which the <em>application</em> runs</dd>

  <dt>group member</dt>
  <dd>a <em>user</em> being part of a <em>user group</em></dd>

  <dt>Tanker</dt>
  <dd>provider of <em>Tanker Core</em>, the <em>Trustchain</em>, and the <em>identity verification service</em></dd>

  <dt>Tanker Core</dt>
  <dd>a privacy SDK, integrated by the <em>customer</em> into the <em>application</em>, allowing to encrypt, decrypt, and share data between <em>user</em>s</dd>

  <dt>Tanker server</dt>
  <dd>server hosting the <em>Trustchain</em>, and the <em>identity verification service</em></dd>

  <dt>Trustchain</dt>
  <dd>a tamper-proof, append-only cryptographic log of chained and signed <em>block</em>s</dd>

  <dd>a 'virtual' device associated with a <em>user</em>, it is the public part of an <em>verification key</em> registered on the <em>Trustchain</em></dd>

  <dt>verification key</dt>
  <dd>a key stored either by the <em>user</em> or by the <em>identity verification service</em> to validate new <em>device</em> creations</dd>

  <dt>identity verification service</dt>
  <dd>an optional <em>Tanker</em> service involved in <em>device</em> management.</dd>

  <dt>user</dt>
  <dd>a user of the <em>application</em>, owning one or more <em>device</em>s</dd>

  <dt>user group</dt>
  <dd>a group of <em>user</em>s, created and managed by the <em>group members</em> themselves. <em>Data</em> shared with a <em>user group</em> is accessible to all its <em>group member</em>s</dd>

  <dt>resource</dt>
  <dd>same as <em>data</em></dd>
</dl>

