# Protocol

The following chapter describes how the previously defined [concepts](#concepts) are used in Tanker.

The protocols have been split in 3 sections for ease of reading:

- The [cryptographic identity management](#cryptographic-identity-management) section describes how the *Tanker SDK* ensures that a *user*'s cryptographic identity is available on every *device*, and how the optional *unlock service* prevents a *user* from losing their cryptographic identity
- The [encryption](#encryption) section describes how *resource*s are encrypted and shared with *user*s
- The [group encryption](#group-encryption) section describes how *user group*s are managed and how to share encrypted *resource*s with them


## Cryptographic identity management

The following diagram describes how the different protocols interact together when opening a *Tanker* session.

![Protocol parts involved in opening a Tanker session](./img/open_chart.png "Protocol parts involved in opening a Tanker session")

Please note that when the user handles the [ULK](#unlock-key) themselves (without using the *unlock service*), the system is fully end-to-end.

### User registration

Prerequisite: the *user* is registered on the *application server*, but not on the *Trustchain*.
Once the *user* is authenticated, the *application* can fetch the *user*'s [SPerID](#secret-permanent-identity) from the *application server*.

The *Tanker SDK* extracts the [US](#user-secret) and the [UID](#user-id) from the [SPerID](#secret-permanent-identity), and use them to create the *device*'s [LES](#device-id). Next, the *Tanker SDK* creates the [DEK](#device-keys), [DSK](#device-keys), and [UEK](#user-keys), and stores them to the [LES](#device-id).

A `device_creation` *block* is then constructed with these keys, and signed with the ephemeral private key of the [delegation token](#delegation-token) found in the [SPerID](#secret-permanent-identity) retrieved earlier. The *block* is pushed to the *Trustchain* and verified by the *Tanker server*.

Assuming the pushed *block* is correct, the *Tanker server* acknowledges it, allowing the *device* to [authenticate itself](#device-authentication).

### Device authentication

Prerequisite: the *device* is already registered on the *Trustchain*, the [SPerID](#secret-permanent-identity) has been retrieved from the *application server* after the *user* has been authenticated against the *application server*.

The *Tanker SDK* uses the [US](#user-secret) retrieved from the [SPerID](#secret-permanent-identity) to access the [DEK](#device-keys) and [DSK](#device-keys) stored in the [LES](#device-id).

When the *user* opens their *Tanker* session, the *device* opens an HTTPS WebSocket connection with the *Tanker server*.
Once the connection is established, the *device* asks for an authentication challenge.

This challenge is an array of bytes made of:

- a fixed prefix shared between the *Tanker server* and all *Tanker SDK* implementations
- a random part

The *device* then sends the authentication message containing:

- the signature of the challenge with the private [DSK](#device-keys)
- the [UID](#user-id)
- the `Trustchain ID`
- the public [DSK](#device-keys), acting as a unique identifier for the *device*

This allows the *Tanker server* to check that:

- the *device* is registered on the *Trustchain*
- the *device* is correctly associated with the provided [UID](#user-id)
- the signature matches the challenge

If that is the case, the authentication is successful, and the session is open, otherwise the connection is closed.

### Unlock key registration

Prerequisite: the *user* has already registered their first *device*.

To add an additional *device*, the *Tanker SDK* must first create an *unlock device* and push it to the *Trustchain*.
The created [DEK](#device-keys) and [DSK](#device-keys) pairs are not saved in the [LES](#device-id) but serialized in an opaque token; the [ULK](#unlock-key).

This [ULK](#unlock-key) can either be given to the *user* to be stored in a safe place or stored on the *unlock service*. It can then be used to add any other *device*.

### Unlock with the unlock service

#### Unlock setup

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server* and they have created a [ULK](#unlock-key).

Tanker offers an *unlock service* to store the [ULK](#unlock-key) safely for the *user*.

The [ULK](#unlock-key) is first encrypted on the *device* with the [US](#user-secret) and sent to the *Tanker server*.
Its access is protected by one of the following methods:

- Password protection: the *user* sets a password that is used later for [unlocking by password](#unlock-by-password)
- Email protection: the *user* sets up their email address that is used later for [unlocking by email](#unlock-by-email)

#### Unlock by password

Prerequisite: the *user* has set up the *Tanker* *unlock service* with password protection beforehand and is now trying to use a new *device* that is not registered on the *Trustchain* yet.

The *user* must provide their password to the *Tanker SDK*. It is hashed client-side, then sent to the *Tanker server* to fetch the encrypted [ULK](#unlock-key).

The *user* still needs to authenticate against the *application server* to obtain their [US](#user-secret) to decrypt the [ULK](#unlock-key).

The *user* can then use the [ULK](#unlock-key) to [register a new
device](#device-registration).

#### Unlock by email

Prerequisite: the *user* has set up the *Tanker* *unlock service* with email protection beforehand and is now trying to use a new *device* that is not registered on the *Trustchain* yet.

The unlocking by email process is used to authenticate a *user* against the *Tanker server* when they don't have a password or have forgotten it.

The *user* triggers an unlock through a request to the *application server* which in turn calls the *Tanker server*. The *Tanker server* then sends an email containing the *Tanker verification code*.

The *user* clicks on the link contained in the email. It allows them to be authenticated against the *Tanker server* to fetch their encrypted [ULK](#unlock-key).

The *user* still needs to authenticate against the *application server* to obtain their [US](#user-secret) to decrypt the [ULK](#unlock-key). The *application* usually includes an additional token in the aforementioned email for that.

The *user* can then use the [ULK](#unlock-key) to [register a new
device](#device-registration).

### Device registration

Prerequisite: the *user* has created a [ULK](#unlock-key).

Except for the first *device*, which is validated in a specific way previously described, additional *device*s must be validated by an already registered *device*. In practice, these *device*s are validated by the *unlock device*.

Given the *user*'s [ULK](#unlock-key), the steps to register a new *device* are as follows:

1. Extract the *unlock device*'s [DEK](#device-keys) and [DSK](#device-keys) from the [ULK](#unlock-key)
2. Pull the *Trustchain* up to the *unlock device*'s `device_creation` *block*, verify it and extract the [UEK](#user-keys) from it
3. Decrypt the [UEK](#user-keys) using the  *unlock device*'s private [DEK](#device-keys)
4. Generate the new *device*'s [DEK](#device-keys) and [DSK](#device-keys)
5. Construct the new *device*'s `device_creation` *block* and sign it with the *unlock device*'s private [DSK](#device-keys)
6. Push the *block* to the *Trustchain*
7. The *Tanker server* validates the *block*
8. The new *device* authenticates against the *Tanker server*

## Encryption

### Data encryption and decryption

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*.

Encrypting *data* implies automatically sharing the [REK](#resource-keys) with the user themselves so that their other devices can decrypt the data. The steps to encrypt a resource are as follows:

1. The *application* calls `tanker.encrypt`
2. The *Tanker SDK* generates the [REK](#resource-keys)
3. The *Tanker SDK* symmetrically encrypts the given *data* with the [REK](#resource-keys)
4. The *Tanker SDK* shares the [REK](#resource-keys) with the recipients as described in [Sharing with users](#sharing-with-users) and [Sharing with user groups](#sharing-with-user-groups)

### Sharing with users

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*, and some *data* has been encrypted.

Given the [REK](#resource-keys), sharing encrypted *data* with another *user* is done as follows:

1. The *application* fetches the [PPerID](#public-permanent-identity) for the recipient from the *application server*
2. The *application* calls `tanker.share` with the [PPerID](#public-permanent-identity)
3. The *Tanker SDK* fetches the recipient's `device_creation` *block*s from the *Trustchain*
4. The *Tanker SDK* verifies them and extract the recipient's public [UEK](#user-keys)
5. The *Tanker SDK* asymmetrically encrypts the [REK](#resource-keys) with the public [UEK](#user-keys), creating the [SEK](#resource-keys)
6. The *Tanker SDK* creates a `key_publish` *block* containing the [SEK](#resource-keys) and the recipient's public [UEK](#user-keys), and pushes it to the *Trustchain*
7. The *Tanker server* validates the *block* and notifies all the recipients' *device*s
8. The recipient's *device* retrieves the *block*, verifies it and decrypts the [SEK](#resource-keys) using the private [UEK](#user-keys), obtaining the [REK](#resource-keys)
9. The *application* calls `tanker.decrypt` on the recipient's *device*
10. The *Tanker SDK* decrypts the data using the [REK](#resource-keys)

## Group encryption
### User group creation

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*.

The steps to create a new *user group* are as follows:

1. The *application* fetches the [PPerID](#public-permanent-identity) for each future *group member*
2. The *application* calls `tanker.createGroup` with the obtained [PPerID](#public-permanent-identity)s
3. The *Tanker SDK* fetches all future *group member*s' `device_creation` *block*s from the *Trustchain*
4. The *Tanker SDK* verifies them and extracts their public [UEK](#user-keys)
5. The *Tanker SDK* generates the [GEK](#user-group-keys) and the [GSK](#user-group-keys)
6. The *Tanker SDK* encrypts the private [GSK](#user-group-keys) with the public [GEK](#user-group-keys)
7. The *Tanker SDK* encrypts the private [GEK](#user-group-keys) with each future *group member*'s public [UEK](#user-keys)
8. Using the private [GSK](#user-group-keys), the *Tanker SDK* signs the public and encrypted private [GEK](#user-group-keys) and [GSK](#user-group-keys)
9. The *Tanker SDK* creates a `user_group_creation` *block* with all of the above and pushes it to the *Trustchain*
10. The *Tanker server* validates the *block* and sends the update to all *group member*s' *device*s
11. Each *group member*s' *device* retrieves the *block*, verifies it and decrypts the [GEK](#user-group-keys) and [GSK](#user-group-keys) using their private [UEK](#user-keys)

### Sharing with user groups

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*, and some *data* has been encrypted.

Sharing with a *user group* is pretty much the same as sharing with a *user* but requires using a [GID](#group-id) returned by `tanker.createGroup`.
The steps are as follows:

1. The *application* calls `tanker.share` with the [GID](#group-id) to share with
2. The *Tanker SDK* fetches the recipient *group*'s public [GEK](#user-group-keys) from the *Trustchain*, if not already present in the [LES](#device-id)
3. The *Tanker SDK* encrypts the [REK](#resource-keys) with the public [GEK](#user-group-keys). The result is the [SEK](#resource-keys) for this particular *user group* and *resource*
4. The *Tanker SDK* creates a *block* containing the [SEK](#resource-keys) and the recipient public [GEK](#user-group-keys)
5. The *Tanker SDK* pushes the *block* to the *Trustchain*
6. The *Tanker server* validates the *block* and notifies all the recipients' *device*s
8. The recipients' *device*s retrieve the *block*, verify it and decrypt the [SEK](#resource-keys) using the private [GEK](#user-group-keys), obtaining the [REK](#resource-keys)
7. The *application* calls `tanker.decrypt` on one of the recipient group's *device*s
10. The *Tanker SDK* decrypts the data using the [REK](#resource-keys)

## Preregistration
### Provisional identity creation

*Tanker* supports sharing with users that are not yet registered.
The only currently supported authentication method for these identities is email.

The ownership of a provisional identity is split between *Tanker* and the *application server*.
When the *user* wants to claim the provisional identity, they will authenticate with an email against both the *application server* and against *Tanker* so that they get both halves of the [SProID](#secret-provisional-identity).

### Sharing with a provisional identity

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*, some *data* has been encrypted

1. The *application* requests a public identity for a user's email which is not registered yet
2. The *application server* generates a [PProID](#public-provisional-identity) and sends it back to the *application*
3. The *application* calls `tanker.share` with the *application* [PProID](#public-provisional-identity)
4. The *Tanker SDK* requests a [PProID](#public-provisional-identity) for the user from the *Tanker server*
5. The *Tanker server* generates a [PProID](#public-provisional-identity)
6. The *Tanker SDK* asymmetrically encrypts the [REK](#resource-keys) with the *application* [PProID](#public-provisional-identity)'s encryption key
7. The *Tanker SDK* asymmetrically encrypts the previous result with the *Tanker* [PProID](#public-provisional-identity)'s encryption key to get the [SEK](#resource-keys)
8. The *Tanker SDK* creates a `key_publish` *block* containing the [SEK](#resource-keys) and both of the recipient's [PProID](#public-provisional-identity)s' public signature keys, and pushes it to the *Trustchain*
9. The *Tanker server* validates and holds the block until it is claimed

### Creating a group with a provisional user

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server*.

1. The *application* requests a public identity for a user's email which is not registered yet
2. The *application server* generates the [PProID](#public-provisional-identity) for the future *group member*
3. The *application* calls `tanker.createGroup` with the obtained [PProID](#public-provisional-identity)
4. The *Tanker SDK* requests a [PProID](#public-provisional-identity) for the user from the *Tanker server*
5. The *Tanker server* generates a [PProID](#public-provisional-identity)
6. The *Tanker SDK* generates the [GEK](#user-group-keys) and the [GSK](#user-group-keys)
7. The *Tanker SDK* encrypts the private [GSK](#user-group-keys) with the public [GEK](#user-group-keys)
8. The *Tanker SDK* encrypts the private [GEK](#user-group-keys) with the *application* [PProID](#public-provisional-identity)'s encryption key
9. The *Tanker SDK* encrypts the result of the previous step with the *Tanker* [PProID](#public-provisional-identity)'s encryption key
10. Using the private [GSK](#user-group-keys), the *Tanker SDK* signs the public and encrypted private [GEK](#user-group-keys) and [GSK](#user-group-keys)
11. The *Tanker SDK* creates a `user_group_creation` *block* with all of the above and pushes it to the *Trustchain*
12. The *Tanker server* validates and holds the block until it is claimed

### Claiming a provisional identity

Prerequisite: the *user*'s *device* is authenticated against the *Tanker server* and some *users* have shared *data* with a provisional identity owned by them or added it to a group.

1. The *application server* sends an email to the *user* with a verification code
2. The *device* gets the *application* [SProID](#secret-provisional-identity) using the verification code
3. The *Tanker server* sends an email to the *user* with a verification code
4. The *Tanker SDK* gets the *Tanker* [SProID](#secret-provisional-identity) using the verification code
5. The *Tanker SDK* encrypts both [SProID](#secret-provisional-identity)s' private encryption keys with the [UEK](#user-keys)
6. The *Tanker SDK* signs its [DID](#device-id) with the *application* [SProID](#secret-provisional-identity)'s private signature key
7. The *Tanker SDK* signs its [DID](#device-id) with the *Tanker* [SProID](#secret-provisional-identity)'s private signature key
8. The *Tanker SDK* creates a block with both [SProID](#secret-provisional-identity)s' public signature keys, the aforementioned signatures, and the encrypted [SProID](#secret-provisional-identity) keys and pushes it to the *Trustchain*
9. The *Tanker server* validates the block and stores it
10. The *Tanker server* publishes all the `key_publish` and *group* blocks related to the claimed provisional identity to the *device*
11. The *Tanker SDK* decrypts the [SEK](#resource-keys)s with both [SProID](#secret-provisional-identity)s' private encryption keys when needed
12. The *Tanker SDK* decrypts the [GEK](#user-group-keys)s with both [SProID](#secret-provisional-identity)s' private encryption keys when needed
