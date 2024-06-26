<!-- note: duplicated from concepts.md for now -->
[Tanker App]: concepts.md#tanker-app "An application created using Tanker's App management API"
[Trustchain]: concepts.md#trustchain "A Trustchain is a collection of signed blocks, attached to a given app"
[Device Encryption Key Pair]: concepts.md#device-keys "Used to encrypt the user keys"
[Device ID]: concepts.md#device-id "Unique identifier of a device belonging to a user"
[Device Signature Key Pair]: concepts.md#device-keys "Used when the user signs a block"
[Delegation Token]: concepts.md#delegation-token "Part of a Secret Permanent Identity. Combination of an ephemeral private key and the user ID and the signature of them both"
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
[TLS]: concepts.md#transport-layer-security "Tanker Core and server uses the TLS protocol to communicate across the Internet, preventing eavesdropping and tampering"
[Resource ID]: concepts.md#resource-id "The unique ID part of an encrypted data"
[Verification Method]: concepts.md#Verification-method "A verification method allows a user to retrieve their encrypted verification key"
[Verification Code]: concepts.md#verification-code "A 8 digits random code used to confirm a user's identity"
[OIDC Challenge]: concepts.md#oidc-challenge "A 24 bytes random code used to perform an additional challenge during OIDC verification"
[Salt]: concepts.md#salt "A salt is random used as an additional input to a hash function"

# Protocol

The following chapters describe how the previously defined [concepts](#concepts) are used in Tanker.

The protocols have been split in 4 sections for ease of reading:

- The [session management](#session-management) section describes how *Tanker Core* ensures that a *user*'s cryptographic identity is available on every *device*, and how the optional *identity verification service* prevents a *user* from losing their cryptographic identity
- The [verification methods](#verification-methods) section describes how different methods are available to the user to retrieve it's cryptographic identity using the *Tanker Identity* service.
- The [encryption and decryption](#encryption-and-decryption) section describes how *resource*s are encrypted, decrypted, and shared with *user*s
- The [preregistration](#preregistration) section describes how a registered user can shared a resource to a non yet registered user.

## Communications

Every communication and information exchanges between *Tanker Core* and the *Tanker Server* are done through [TLS] connection. In the same way, exchanges between *Tanker Core* and the *application server* also use [TLS], in particular for Tanker Identity retrievals.

## Session management

### Cryptographic identity management

The following diagram describes how the different protocols interact together when opening a *Tanker* session.

![Protocol parts involved in opening a Tanker session](./img/open_chart.png "Protocol parts involved in opening a Tanker session")

Please note that when the user handles the [Verification Key] themselves (without using the *identity verification service*), the system is fully end-to-end.

Before being able to encrypt/decrypt and exchange keys, a *user* must exist on the *Trustchain*. This means they have at least one valid *device* (not revoked) registered on *Trustchain*. To register on the *Trustchain*, the *Tanker server* asks for the *user*'s proof of authentication against the *application server*. The *application server* must be able to generate a [Secret Permanent Identity] and deliver it to the *application* securely.

One of the first action taken by *Tanker Core* is to ask the *Tanker server* for the user's existence on the *Trustchain* and if so, if the current *device* exists. The result of this request will either initiate a [user registration](#user-registration), a [device registration](#device-registration), or the required follow-up to a [device authentication](#device-authentication).

If no [Local Encrypted Storage] is found, *Tanker Core* will create one automatically. *Tanker Core* extracts the [User Secret] and the [User ID] from the [Secret Permanent Identity], and use them to create the *device*'s [Local Encrypted Storage].

### User Registration

Prerequisite: The *user* is registered on the *application*, but not on the *Trustchain*.
Once the *user* is authenticated, the *application* can fetch the *user*'s [Secret Permanent Identity] from the *application server*.

*Tanker Core* creates a [User Encryption Key Pair].

*Tanker Core* creates a [Device Encryption Key Pair] and a [Device Signature Key Pair] for the *virtual device*. It serializes them as an opaque token, the [Verification Key].

A first `device_creation` block is constructed with the *virtual device*'s public [Device Encryption Key Pair] and public [Device Signature Key Pair] and the encrypted [User Encryption Key Pair]. This block is signed with the ephemeral private key of the [Delegation Token] found in the [Secret Permanent Identity].

*Tanker Core* creates a [Device Encryption Key Pair] and a [Device Signature Key Pair] for the *physical device*.

A second `device_creation` block is created with the *physical device*'s [Device Encryption Key Pair] and [Device Signature Key Pair]. *Tanker Core* creates a new ephemeral key pair, signs this block with the ephemeral private key and uses the *virtual device*'s [Device Signature Key Pair] to sign the *delegation* and fill the block's *delegation_signature* field.

Finally, both `device_creation` *block*s are pushed to the *Trustchain*, the *virtual device* before the *physical one*, and the [Verification Key] is encrypted and uploaded to the *Tanker server* according to the [Verification Method] used by the *user*.
If the [Verification Method] is an end to end method (e.g. the end to end passphrase method), the [Verification Key] to be uploaded is encrypted once with the [User Encryption Key Pair] and once with the end to end method's secret value (e.g. the end to end passphrase). Otherwise, for non-end to end methods, it is encrypted only with the [User Secret].

Assuming the pushed *block*s are correct, the *Tanker server* stores them, and initiates an authenticated session for the *physical device*. The just created *physical device* permanently stores its [Device Encryption Key Pair] and [Device Signature Key Pair] in the [Local Encrypted Storage].

### User Enrollment

Prerequisite: The *user* is registered on the *application*, but not on the *Trustchain*.
The *application* trusts or verified one of the *user*'s [Verification Method](#verification-methods).

[User enrollment](#user-enrollment) is done from the *application server*.
It must be used when converting an existing *user* base to migrate an *application* to Tanker.
The feature must be disabled in production mode.

Using the *user*'s [Secret Permanent Identity], *Tanker Core* creates a [User Encryption Key Pair].

*Tanker Core* creates a [Device Encryption Key Pair] and a [Device Signature Key Pair] for the *virtual device*. It serializes them as an opaque token, the [Verification Key].
This *device* is not a "physical" one, it has no [Local Encrypted Storage].

A `device_creation` block is constructed with the *virtual device*'s public [Device Encryption Key Pair] and public [Device Signature Key Pair] and the encrypted [User Encryption Key Pair]. This block is signed with the ephemeral private key of the [Delegation Token] found in the [Secret Permanent Identity].

Finally, the `device_creation` *block* is pushed to the *Trustchain*, the [Verification Key] is encrypted with the [User Secret] and uploaded to the *Tanker server* alongside the *user*'s [Verification Method]s trusted or verified by the *application server*.

As opposed to a [user registration](#user-registration), only a single `device_creation` block is pushed, for the *virtual device*.

Assuming the pushed *block* is correct, the *Tanker server* stores it.

### Device Registration

Prerequisite: the *user* registered with a [Verification Method](#verification-methods) on a previous device.

The first time the *user* starts a Tanker session on a new device, their identity must be *verified* using one of their [Verification Method]s.

Steps to register a new *device*:

1. The *user* uses one the [Verification Method]s available to them to obtain their encrypted [Verification Key]
1. *Tanker Core* decrypts the encrypted [Verification Key] using either the [User Secret] for a non-end to end method, or the end to end method's secret.
1. *Tanker Core* extracts the *virtual device*'s [Device Encryption Key Pair] and [Device Signature Key Pair] from the [Verification Key]
1. *Tanker Core* requests the last encrypted [User Encryption Key Pair] and the *virtual device*'s *device id* from the *Tanker server*
1. *Tanker Core* decrypts the encrypted [User Encryption Key Pair] using the *virtual device*'s [Device Encryption Key Pair]
1. *Tanker Core* generates the new *physical device*'s [Device Encryption Key Pair] and [Device Signature Key Pair]
1. *Tanker Core* constructs the new *physical device*'s `device_creation` *block* with the *virtual device*'s *device id*, retrieved from the *Tanker server*, as the block author
1. *Tanker Core* uses the *virtual device*'s private [Device Signature Key Pair] to sign the delegation and fill the block's *delegation signature* field
1. *Tanker Core* creates a new ephemeral signature key pair
1. *Tanker Core* signs the `device_creation` *block* with the private ephemeral key pair
1. *Tanker Core* pushes the `device_creation` *block* to the *Trustchain*
1. The *Tanker server* validates the *block*, stores it, and initiates an authenticated session for the newly created *physical device*

Note that this procedure can only create *physical devices*, not *virtual ones*.

### Session Tokens

Prerequisite: The user is registered on the application.

A *session token* can be obtained when a user creates their first device, or at any time later when proving ownership of a  [Verification Method](#verification-methods).

Steps to obtain and check a *session token*:
1. The *user* uses one the [Verification Method]s available to them, and requests a *session token* through the API
1. *Tanker Core* generates a nonce and sends it to the *Tanker server* along with the verification
1. *Tanker Core* generates a *session certificate* containing the time and [Verification Method] used, and signs it with the *physical device*'s [Device Signature Key Pair]
1. *Tanker Core* sends the *session certificate* to the *Tanker server* along with the previously generated random nonce
1. The *Tanker server* checks that the received *session certificate* matches the [Verification Method] used with this random nonce
1. The *Tanker server* stores the *session certificate* and sends a *session token* to the *user* in exchange
1. The *user* sends the received *session token* to the *application* servers
1. The *application* makes a request to the *Tanker server* with the provided *session token* and the expected [Verification Method]
1. The *Tanker server* checks that the expected [Verification Method] matches with the *session certificate*, and returns the status to the *application*

### Device authentication

Prerequisite: the *device* is already registered on the *Trustchain*, the [Secret Permanent Identity] has been retrieved from the *application server* after the *user* has been authenticated against the *application server*.

*Tanker Core* uses the [User Secret] retrieved from the [Secret Permanent Identity] to access the [Device Encryption Key Pair] and [Device Signature Key Pair] stored in the [Local Encrypted Storage].

The *device* initiates the authentication by requesting an *authentication challenge* from *Tanker server*. This challenge made of:

- A fixed prefix shared between the *Tanker server* and all of the *Tanker Core* implementations
- A random part

The *device* then sends the *authentication message* containing:

- The *Tanker App* ID
- The [Device ID]
- The challenge
- The signature of the challenge with the private [Device Signature Key Pair]

This allows the *Tanker server* to check that:

- The *device* is registered on the *Trustchain*
- The *device* is correctly associated with the provided [User ID]
- The signature matches the challenge

If any of these checks fail, the authenticated session is not created.

### Device Revocation

Prerequisite: The *user* has a session opened.

A *user* may want to dispose of a device, for various reasons (eg: not used anymore, being compromised, etc). A revoked device cannot authenticate itself to the *Tanker server* anymore, it won't new received blocks for newly shared resources, but it could still decrypts the one received before its revocation. Some *Tanker Core* implementation may attempt to destroy the [Local Encrypted Storage] when receiving the `device_revocation` block. The *user* uses one of their authenticated *device* to revoke another. The targeted *device* may or may not be connected at the time of the revocation.

The *user* provides the device ID they want to revoke. *Tanker Core* generates a new [User Encryption Key] and encrypts the private key for every *user*'s devices with their public [Device Encryption Key Pair]. *Tanker Core* also encrypts the previous private [User Encryption Key Pair] with the new public [User Encryption Key Pair].*Tanker Core* constructs a `device_revocation` block with all of the above and push it to the *Trustchain*.

A *virtual device* can not be revoked.

## Verification Methods

*Tanker Core* requires at least one verification method to be registered at all time for the *user*. These methods allow a *user* to register a new *device* on the Trustchain by providing a shared proof between him and the *Tanker server*.

The only way to add more *device*s, after the first one created during [user registration](#user-registration), is to sign a new `device_creation` block containing the new device's keys with a *user*'s existing device.

For this purpose, during [user registration](#user_registration) *Tanker Core* creates a *virtual device* and pushes its `device_creation` block. This *device* is not a "physical" one, it has no [Local Encrypted Storage]. Instead, the [Device Signature Key Pair] and the [Device Encryption Key Pair] are serialized as an opaque token, we call it the [Verification Key].

There are two types of verification methods: end-to-end (E2E) methods, and non-E2E methods.
The difference is that for non-E2E methods, the [Verification Key] is encrypted with the [User Secret] and stored on the *Tanker server*, while for E2E methods it is encrypted with the [User Encryption Key Pair] and the E2E method's secret value (e.g. an E2E passphrase) before being stored on the *Tanker server*.
The reason for this difference is that the [User Secret] can be accessed by parties others than the end-users themselves, namely the *application*, so it is not strictly end-to-end. End-to-end verification methods offer stricter guarantees but do not allow for any possibility of recovery if the verification method's secret value (e.g. passphrase) is lost.

*User*s can use their [Verification Key] 'as is' when registering a new device. But *Tanker Core* proposes several [Verification Method]s as an easier way to handle the [Verification Key].

[Verification Method] registration happens at least once during [User Registration] and can be done anytime by the *user* through an authenticated device.

### Verification Key

*Tanker Core* accepts a [Verification Key]. It will be used 'as is'. See [device registration](#device-registration) for how the [Verification Key] is used to register a device.

### Passphrase

The *user* provides a passphrase when registering a new [Verification Method]. The passphrase will be hashed by *Tanker Core* before sending it to the *Tanker server*. The *Tanker server* will rehash with a salt the received hashed passphrase before storing it alongside the user's encrypted [Verification Key].

When the *user* needs to [register a new device](#device-registration), they will provide their passphrase. *Tanker Core* will hash the passphrase before sending it to the *Tanker server*. Then, the *Tanker server* will rehash with the seed the received hashed passphrase and match this result with the stored passphrase. If they match, the *Tanker server* will return the encrypted [Verification Key].

### E2E Passphrase

This method is an end-to-end (E2E) verification method.

The *user* provides a passphrase when registering a new [Verification Method]. The passphrase will be hashed by *Tanker Core*, with a unique constant value appended, before sending it to the *Tanker server*.
The passphrase will be hashed a second time by *Tanker Core*, with a different unique constant, and the resulting hash will be used to encrypt the [Verification Key]. The [verification Key] will also be encrypted with the [User Encryption Key Pair]. Then both ciphertexts are sent to the *Tanker server*.
The *Tanker server* will store the hashed passphrase alongside the user's encrypted [Verification Key].

When the *user* needs to [register a new device](#device-registration), they will provide their passphrase. *Tanker Core* will hash the passphrase, again with the first unique constant value appended, before sending it to the *Tanker server*. Then, the *Tanker server* will match this result with the stored passphrase. If they match, the *Tanker server* will return the encrypted [Verification Key].

### Email

For the *application* to offer Email as a verification method, they need to send `HTTP` requests to the *Tanker server*. To allow that, the *application* administrator needs to register their domain origin of their requests to the *Tanker App*.

Before registering an email address as a [Verification Method], the *Tanker server* must make sure the *user* has access to this email address.

The steps to verify the *user*'s email address and register their email [Verification Method] are as follow:

1. The *user* provides an email address
1. The *application* makes a request to the *Tanker server* with the provided email address and some payload for the email message
1. The *Tanker server* generates a [Verification Code]
1. The *Tanker server* records the hashed email address and the [Verification Code]
1. The *Tanker server* sends an email to the provided email address with the [Verification Code]
1. The *user* provides the [Verification Code] to the *application*
1. The *application* forwards the email address of the *user* and the [Verification Code] to *Tanker Core*
1. *Tanker Core* hashes the email address and, with the [Verification Code], sends a request to the *Tanker server*
1. The *Tanker server* will match the provided hashed email address and the [Verification Code]

If the *Tanker server* does not returns an error, it means the process has ended successfully and the *user* has now registered their provided email address as a [Verification Method].

The process to [register a new device](#device-registration) with an email [Verification Method] is the same as described above. The only difference is that at the end of the process the *Tanker server* returns the *user*'s [Verification Key].

### Phone number

For the *application* to offer Phone number as a verification method, they need to send `HTTP` requests to the *Tanker server*. To allow that, the *application* administrator needs to register their domain origin of their requests to the *Tanker App*.

Before registering a phone number as a [Verification Method], the *Tanker server* must make sure the *user* has access to this phone number.

The steps to verify the *user*'s phone number and register their phone number [Verification Method] are as follow:

1. The *user* provides a phone number
1. The *application* makes a request to the *Tanker server* with the provided phone number and some payload for the SMS message
1. The *Tanker server* generates a [Verification Code]
1. The *Tanker server* records the hashed phone number and the [Verification Code]
1. The *Tanker server* sends an SMS to the provided phone number with the [Verification Code]
1. The *user* receives the SMS with the [Verification Code] inside
1. The *user* provides the [Verification Code] to the *application*
1. The *application* forwards the phone number of the *user* and the [Verification Code] to *Tanker Core*
1. *Tanker Core* generates a private [Salt] from the [user secret] and sends a request to the *Tanker server*. The request contains the [Verification Code], phone number in clear text and the [Salt].
1. The *Tanker server* will hash the provided phone number and match this hash and the [Verification Code]. It will record a second hash generated from the phone number and [Salt]

If the *Tanker server* does not return an error, it means the process has ended successfully and the *user* has now registered their provided phone number as a [Verification Method].

The process to [register a new device](#device-registration) with a phone number [Verification Method] is the same as described above. The only difference is that at the end of the process the *Tanker server* returns the *user*'s [Verification Key].

### OpenID Connect (deprecated)

For the *application* to offer OpenID Connect as a verification method, *application* owners need to register their `OIDC`'s `Client ID` to the *Tanker App*.

The steps to verify the *user* using OpenID Connect are as follows:
1. The *user* requests a random `nonce` from *Tanker Core*
1. *Tanker Core* generates an asymetric key pair and records the key pair on the *user*'s device
1. *Tanker Core* encodes the public key and returns the encoded result to use as `nonce`
1. The *user* authenticates against the OpenID Connect provider using the generated `nonce`
1. The *user* receives an `authorization code` from the OpenID Connect provider
1. The *application* exchanges the `authorization code` to receive the *user* `ID Token` from the OpenID Connect provider
1. The *application* provides this `ID Token` to *Tanker Core* during [user registration](#user-registration) or [new device registration](#device-registration)
1. *Tanker Core* extracts the `nonce` from the `ID Token` and requests an [OIDC Challenge] from the *Tanker server*
1. The *Tanker server* generates an [OIDC Challenge]
1. The *Tanker server* records the `nonce` and the [OIDC Challenge]
1. The *Tanker server* sends the [OIDC Challenge]
1. *Tanker Core* signs the [OIDC Challenge] with the secret key matching the `nonce`
1. *Tanker Core* sends a request to the *Tanker server*. The request contains the `ID Token`, the [OIDC Challenge] and the [OIDC Challenge]'s signature
1. The *Tanker server* verifies the provided `ID Token` according to [the OpenID recommendation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)
1. The *Tanker server* extracts the `nonce` from the `ID Token`, matches the `nonce`, the request's [OIDC Challenge] and the recorded [OIDC Challenge]
1. The *Tanker server* decodes the `nonce` into a public key and verifies the [OIDC Challenge]'s signature with the key
1. The *Tanker server* extracts the *user*'s `subject` from the `ID Token` and hashes the `subject` before storing it.

The process to [register a new device](#device-registration) with an `OIDC` [Verification Method] is the same as described above. The only difference is that at the end of the process the *Tanker server* returns the *user*'s [Verification Key].

### OpenID Connect with authorization code flow

The *Tanker Server* is integrated with a list of trusted OpenID connect providers. The *Tanker Server* uses its own `client ID` and `client secret` to act as a `service provider`. It follows the [OpenID protocol](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth).

For the *application* to offer OpenID Connect as a [Verification Method], *application* owners need to register the trusted OpenID Connect provider with the *Tanker App*.

The steps to verify the *user* using OpenID Connect are as follows:
1. The *user* queries the `OIDC` sign-in endpoint of the *Tanker server*
1. The *Tanker server* prepares an authentication request and redirects the user to the OpenID Connect provider
1. The OpenID Connect provider authenticates the *user* and obtains their consent/authorization
1. The OpenID Connect provider responds with an `authorization code`
1. *Tanker Core* sends a request to the *Tanker server*. The request contains the `authorization code`
1. The *Tanker server* requests, from the OpenID Connect provider, an `ID Token` in exchange for the `authorization code` and the *Tanker server*'s `client ID` and `client secret`
1. The *Tanker server* verifies the provided `ID Token` according to [the OpenID recommendation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)
1. The *Tanker server* extracts the *user*'s `subject` from the `ID Token` and hashes the `subject` before storing it

The process to [register a new device](#device-registration) with an `OIDC` [Verification Method] is the same as described above. The only difference is that at the end of the process the *Tanker server* returns the *user*'s [Verification Key].

## Encryption and Decryption

### Data encryption

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*.

Encrypting *data* implies automatically sharing the [Resource Encryption Key] with the user themselves so that their other devices can decrypt the data.
Recipients, users and/or groups, can be provided so that the [Resource Encryption Key] be shared with them.
The steps to encrypt a resource are as follows:

1. *Tanker Core* generates the [Resource Encryption Key]
2. *Tanker Core* symmetrically encrypts the given *data* with the [Resource Encryption Key]
3. *Tanker Core* shares the [Resource Encryption Key] with the recipients as described in [Sharing with users](#sharing-with-users) and [Sharing with user groups](#sharing-with-user-groups)

### Sharing with users

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*, and some *data* has been encrypted.

Given the [Resource Encryption Key], sharing encrypted *data* with another *user* is done as follows:

1. The *application* fetches the recipients' [Public Permanent Identity] from the *application server*
1. *Tanker Core* fetches the recipients' first `device_creation` *block* (or all the `device_creation` and `device_revocation` *block*s if the user has at least one revocation) from the *Trustchain*
1. *Tanker Core* verifies the received *block*s
1. *Tanker Core* extracts the recipient's public [User Encryption Key Pair]
1. For each recipient, *Tanker Core* asymmetrically encrypts the [Resource Encryption Key] with the public recipient's [User Encryption Key Pair] creating a [Shared Encrypted Key] for each one of them
1. For each recipient, *Tanker Core* creates a `key_publish` *block* containing the [Shared Encrypted Key] and the recipient's public [User Encryption Key Pair], and pushes it to the *Trustchain*
1. The *Tanker server* validates the *block*

### Data decryption

Prerequisite: the user's has access to the encrypted data

1. *Tanker Core* extracts the [Resource ID] from the encrypted data
1. *Tanker Core* retrieves for the recipients' *device* the `key_publish` *block*,
1. *Tanker Core* decrypts the [Shared Encrypted Key] of the `key_publish` block using the private [User Encryption Key Pair], obtaining the [Resource Encryption Key]
1. *Tanker Core* decrypts the data using the [Resource Encryption Key]

### Group encryption

#### User group creation

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*.

The steps to create a new *user group* are as follows:

1. The *application* fetches the [Public Permanent Identity] for each future *group member*
1. *Tanker Core* fetches the future *group member*s' first `device_creation` *block* (or all the `device_creation` and `device_revocation` *block*s if the user has at least one revocation) from the *Trustchain*
1. *Tanker Core* verifies them and extracts their public [User Encryption Key Pair]
1. *Tanker Core* generates the [Group Encryption Key Pair] and the [Group Signature Key Pair]
1. *Tanker Core* encrypts the private [Group Signature Key Pair] with the public [Group Encryption Key Pair]
1. *Tanker Core* encrypts the private [Group Encryption Key Pair] with each future *group member*'s public [User Encryption Key Pair]
1. Using the private [Group Signature Key Pair], *Tanker Core* signs the public and the encrypted private [Group Encryption Key Pair] and [Group Signature Key Pair]
1. *Tanker Core* creates a `user_group_creation` *block* with all of the above and pushes it to the *Trustchain*
1. The *Tanker server* validates the *block*

#### Add users to a user group

This operation adds users to an existing user group. The group ID, [Group Encryption Key Pair] and the [Group Signature Key Pair] remains unchanged at the end of the operation.

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*. The *user* is a member of the group they want to add users to.

1. The *application* fetches the [Public Permanent Identity] for each new *group member* to add
1. *Tanker Core* fetches the future *group member*s' first `device_creation` *block* (or all the `device_creation` and `device_revocation` *block*s if the user has at least one revocation) from the *Trustchain*
1. *Tanker Core* verifies them and extracts their public [User Encryption Key Pair]
1. *Tanker Core* encrypts the private [Group Encryption Key Pair] with each added *group member*'s public [User Encryption Key Pair]
1. Using the private [Group Signature Key Pair], *Tanker Core* signs all the non-signature fields, in the order defined [here](blocks_format.md#usergroupaddition)
1. *Tanker Core* creates a `user_group_addition` *block* with all of the above and pushes it to the *Trustchain*
1. The *Tanker server* checks that the author is a member of the group
1. The *Tanker server* validates the *block*

#### Remove users from a user group

This operation allows members to remove other members from a group.
This is not done by changing the group keys but only by access control from the *Tanker server*.

1. *Tanker Core* prepares the list of all (permanent and provisional) members to remove from the group
1. Using the private [Group Signature Key Pair], *Tanker Core* signs all the non-signature fields with the prefix string `UserGroupRemoval Signature`, in the order defined [here](blocks_format.md#usergroupremoval)
1. *Tanker Core* creates a `user_group_removal` *block* with all of the above and pushes it to the *Trustchain*
1. The *Tanker server* checks that the author is a member of the group
1. The *Tanker server* validates the *block* but does not store it. Instead, it records the removed members from the group and stops distributing resource keys to these users

#### Sharing with user groups

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*, and some *data* has been encrypted.

Sharing with a *user group* is pretty much the same as sharing with a *user* but requires using a [GID](#group-id).
The steps are as follows:

1. The *application* fetches the [GID](#group-id) to share with
1. *Tanker Core* looks for the [Group Encryption Key Pair] associated with the [GID](#group-id) in the [Local Encrypted Storage]
1. If the [Group Encryption Key Pair] is not already present, *Tanker Core* fetches the corresponding `user_group_creation` from the Trustchain
1. *Tanker Core* verifies the received `user_group_creation`
1. *Tanker Core* encrypts the [Resource Encryption Key] with the public [Group Encryption Key Pair]. The result is the [Shared Encrypted Key] for this particular *user group* and *resource*
1. *Tanker Core* creates a *block* containing the [Shared Encrypted Key] and the recipient public [Group Encryption Key Pair]
1. *Tanker Core* pushes the *block* to the *Trustchain*
1. The *Tanker server* validates the *block*

#### Decrypting with user groups

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*, they have access to the encrypted data and they are member of the current Tanker group.

1. *Tanker Core* extracts the [Resource ID] from the encrypted data
1. *Tanker Core* retrieves for the recipients' *device* the `key_publish` *block*,
1. *Tanker Core* decrypts the [Shared Encrypted Key] using the private [Group Encryption Key Pair], obtaining the [Resource Encryption Key]
1. *Tanker Core* decrypts the data using the [Resource Encryption Key]

## Preregistration

![Claiming a provisional identity](./img/preshare.png "Claiming a provisional identity")

### Provisional identity creation

*Tanker* supports sharing with users that are not yet registered.
The currently supported verification methods for these identities are emails and phone numbers.

The ownership of a provisional identity is split between *Tanker* and the *application server*.
When the *user* wants to claim the provisional identity, they will authenticate, by email or SMS, against both the *application server* and against *Tanker* so that they get both halves of the [Secret Provisional Identity].

### Sharing with a provisional identity

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*, some *data* has been encrypted

1. The *application* requests a public identity for a user's email or phone number which is not registered yet
1. The *application server* generates a [Public Provisional Identity] and sends it back to the *application*
1. *Tanker Core* requests a [Public Provisional Identity] for the user from the *Tanker server*
1. The *Tanker server* generates a [Public Provisional Identity]
1. *Tanker Core* asymmetrically encrypts the [Resource Encryption Key] with the *application* [Public Provisional Identity]'s encryption key
1. *Tanker Core* asymmetrically encrypts the previous result with the *Tanker* [Public Provisional Identity]'s encryption key to get the [Shared Encrypted Key]
1. *Tanker Core* creates a `key_publish` *block* containing the [Shared Encrypted Key] and both of the recipient's [Public Provisional Identity]s' public signature keys, and pushes it to the *Trustchain*
1. The *Tanker server* validates and holds the block until it is claimed

### Creating a group with a provisional user

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*.

1. The *application* requests a public identity for a user's email or phone number which is not registered yet
2. The *application server* generates the [Public Provisional Identity] for the future *group member*
3. The *application* calls `tanker.createGroup` with the obtained [Public Provisional Identity]
4. *Tanker Core* requests a [Public Provisional Identity] for the user from the *Tanker server*
5. The *Tanker server* generates a [Public Provisional Identity]
6. *Tanker Core* generates the [Group Encryption Key Pair] and the [Group Signature Key Pair]
7. *Tanker Core* encrypts the private [Group Signature Key Pair] with the public [Group Encryption Key Pair]
8. *Tanker Core* encrypts the private [Group Encryption Key Pair] with the *application* [Public Provisional Identity]'s encryption key
9. *Tanker Core* encrypts the result of the previous step with the *Tanker* [Public Provisional Identity]'s encryption key
10. Using the private [Group Signature Key Pair], *Tanker Core* signs the public and encrypted private [Group Encryption Key Pair] and [Group Signature Key Pair]
11. *Tanker Core* creates a `user_group_creation` *block* with all the above and pushes it to the *Trustchain*
12. The *Tanker server* validates and holds the block until it is claimed

### Claiming a provisional identity

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*.

1. The *application server* sends a verification code to the *user* (via email or SMS, depending on the verification method associated with the provisional identity)
2. The *device* gets the *application* [Secret Provisional Identity] using the verification code
3. The *Tanker server* sends a verification code to the *user*, via email or SMS
4. *Tanker Core* gets the *Tanker* [Secret Provisional Identity] using the verification code
5. *Tanker Core* encrypts both [Secret Provisional Identity]s' private encryption keys with the [User Encryption Key Pair]
6. *Tanker Core* signs its [Device ID] with the *application* [Secret Provisional Identity]'s private signature key
7. *Tanker Core* signs its [Device ID] with the *Tanker* [Secret Provisional Identity]'s private signature key
8. *Tanker Core* creates a block with both [Secret Provisional Identity]s' public signature keys, the aforementioned signatures, and the encrypted [Secret Provisional Identity] keys and pushes it to the *Trustchain*
9. The *Tanker server* validates the block and stores it
10. The *Tanker server* publishes all the `key_publish` and *group* blocks related to the claimed provisional identity to the *device*
11. *Tanker Core* decrypts the [Shared Encrypted Key]s with both [Secret Provisional Identity]s' private encryption keys when needed
12. *Tanker Core* decrypts the [Group Encryption Key Pair]s with both [Secret Provisional Identity]s' private encryption keys when needed
