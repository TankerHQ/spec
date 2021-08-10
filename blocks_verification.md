# Block Verifications

Blocks are stored on Tanker's infrastructure and distributed to the different clients.

The tree structure of the Trustchain can be read to reconstruct the state of each user's devices at any given time, and verified to make sure every action taken was legitimate at the time. Such a verification is done by every Tanker client receiving a new block, before extracting any information from their payload. Each block’s signature is verified, then their author’s block’s signature recursively until the root block is reached.

## Definitions

- *a zero-array* is an array filled with zeros
- *the user has a user key*: this means that the user has at least one block of type DeviceCreation v3 or DeviceRevocation v2

## Blocks

- The author field must be a TrustchainCreation or a DeviceCreation (except for TrustchainCreation)
- The signature field must contain the signature of the hash of the block by the private key of the author (except for TrustchainCreation and DeviceCreation)

## Trustchain Creation

Verification:

- The author of the block must be a zero-array
- The signature of the block must be a zero-array
- The hash of the block must match the Trustchain ID

Server side:

- The Trustchain signature key must be unique

## Device Creations

Verification:

- The user_id must be the same as the author's one (in case of a secondary device)
- The delegation_signature field must contain the signature of the delegation (user_id + ephemeral public key) by the author's key
- The block's signature field must contain the signature of the block by the ephemeral key

Server-side verifications:

- The user_id must not already exist on the Trustchain (in case of a first device)
- The device public signature and encryption key must be unique
- The first device must be a virtual device, additional ones must be physical devices

### DeviceCreation v2

- The user must not have a user key

### DeviceCreation v3

- The user PublicEncryptionKey must be unique
- It cannot change the user PublicEncryptionKey (in case of a secondary device)

## Device Revocations

Verification:

- The author must be a DeviceCreation block
- The device_id field must contain the hash of a DeviceCreation block
- The revoked device must belong to the author
- The revoked device must not already be revoked
- The revoked device must not be a virtual device

Note: A device can revoke itself, even if it is the last of a user's devices.

### DeviceRevocation v1

- The user must not have a user key
- Block's author can be TrustchainCreation or DeviceCreation

### DeviceRevocation v2

- The user PublicEncryptionKey must be unique (and thus must be different from the previous one)
- If there was a previous key
  - The previous_user_encryption_key must match the user's last public_encryption_key
- Else
  - The previous_user_encryption_key must be a zero-array
  - The encrypted_key_for_previous_user_key must be a zero-array
- Endif
- The encrypted_keys_for_devices field must have exactly one element per remaining active device
- It should not contain any extra encrypted_keys_for_devices
- encrypted_keys_for_devices should target the user's devices

This blocks supports the case where the user does not have a user key because it
is the block that is used to upgrade a user without user key to a user with one.

## User groups

### UserGroupCreation v1 & v2

Verification:

- The author must be a DeviceCreation block
- The group id must not already exist
- The block must be extra-signed with the public_signature_key contained in it

Server side:

- The public encryption and signature keys must be unique
- The keys in encrypted_group_private_encryption_keys_for_users.public_user_encryption_key should be non-obsolete keys (this is a best effort)

### UserGroupAddition v1 & v2

Verification:

- The author must be a DeviceCreation block
- The block must be extra-signed with public_signature_key

Server side:

- The author must be a member of the group
- The previous group block must be the hash of the last group modification of the group referenced by group_id
- The keys in encrypted_group_private_encryption_keys_for_users.public_user_encryption_key should be non-obsolete keys

### UserGroupRemoval

This block is only verified server-side.

Verification:

- The author must be a DeviceCreation block
- The block must be extra-signed with public_signature_key
- The author must be a member of the group
- The removed members must be present in the group
- The removed provisional members must not be claimed

## Key publishes

Only server-side verification is needed for key publishes. Client-side verification is useless as key publishes are leaves of the Trustchain tree → either the key can be decrypted or it cannot.

Server side:

- The block author must be a DeviceCreation

### KeyPublishToDevice

Server side:

- The recipient field must contain the hash of a device creation block

- The recipient must not have a user key

- The recipient must not be a revoked device (at the moment the block is created)
- The key for this resource must not be already shared before to the same recipient

### KeyPublishToUser

Server side:

- The recipient field must contain a valid user public key (one that has not been supersede, this is a best effort)

### KeyPublishToUserGroup

Server side:

- The recipient_group_public_encryption_key must be the last group public key of a group (this is a best effort)

## Claims

### ProvisionalIdentityClaim

Verification:

- The author_signature_by_app_key must be the signature of the (device id + app_public_signature_key + tanker_public_signature_key) by the app_public_signature_key
- The author_signature_by_tanker_key must be the signature of (device_id + app_public_signature_key + tanker_public_signature_key) by the tanker_public_signature_key
- The user_id must match the user_id of the author

Server side:

- The recipient_user_public_key must not be obsolete (best effort)
- The key pair must be already claimed
