<!-- note: duplicated from concepts.md for now -->
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

# Threat model

Like all security solutions, *Tanker* protects against some but not all attacks, at different degrees. Here is an attempt to summarize the impact *Tanker* has on an *application*'s security.

## Preface

Security is not only a matter of encrypting data properly, it must also be reflected in the user experience of the *application* itself. Proper notifications, like new device registration emails and/or in-app notifications must be implemented. Email verification should be enforced for account creation and password recovery. 

Adding Tanker to an *application* will add a few additional security steps in the user experience, like additional identity verification.

Applications should also implement the best in class security recommendations: passwords should be hashed and stored properly, proper development practices should be enforced internally to prevent code changes detrimental to security (like the deterioration of the aforementioned best practices).

## Glossary

**Full access**: The attacker gains access to every *user*'s data  
**Targeted access**: The attacker gains access to one *user*'s data  
**Past data**: The attacker gains access to data uploaded to the application before the attack and up until appropriate actions are taken by the *user* (revoking the device)  
**Future data**: The attacker gains access to data uploaded to the application after the attack, even if appropriate actions are taken by the *user* (revoking the device)  

## Tanker risk factors

*Attack*: An attacker gains control to all of *Tanker*'s APIs, *Trustchain* blocks, their distribution and the identity verification service.

*Impact*: The attacker **cannot access any application data in any way.**

## Application risk factors

| Threat                                               | No Tanker                           | Tanker                              |
| ---------------------------------------------------- | ----------------------------------- | ----------------------------------- |
| One-time data access                                 | Full access to past data            | No access                           |
| Repeatable data access                               | Full access to past and future data | No access                           |
| Identities + repeatable data access                  | Full access to past and future data | No access                           |
| App secret + repeatable data access                  | Full access to past data            | No access                           |
| Identities modification + one-time data access       | Full access to past data            | No access                           |
| Identities modification + repeatable data access     | Full access to past and future data | Full access to future data          |
| Application Code control                             | Full access to past and future data | Full access to past and future data |

### Detailed attacks analysis

**One-time data access**:

*Attack*: An attacker gains access to all user data stored by the *application*. This attack is not repeatable.   

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. With *Tanker*, they have no access to Tanker-encrypted data. 

**Repeatable data access**:

*Attack*: An attacker gains access to all user data stored by the *application*. This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. By repeating the same attack, they also have access to all future user data. With *Tanker*, they have no access to Tanker-encrypted data. 

**Identities + repeatable data access**:

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to all stored *Tanker Identities*. This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. By repeating the same attack, they also have access to all future user data. With *Tanker*, they have no access to Tanker-encrypted data. 

**App secret + repeatable data access**:

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to the private [Trustchain Signature Key Pair] contained in the app secret. This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. By repeating the same attack, they also have access to all future user data. With *Tanker*, they have no access to Tanker-encrypted data. 

**Identity modification + one-time data access** :

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to all stored *Tanker Identities* and are able to change them. They replace all *user* *Identities* by their own. This attack is not repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. With *Tanker*, they have no access to Tanker-encrypted data. All future user data will however be encrypted for the attacker's *Tanker* account.

**Identity modification + repeatable data access** :

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to all stored *Tanker Identities* and are able to change them. This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. With *Tanker*, all future user data will be encrypted for the attacker's *Tanker* account. If the attacker accesses the data again in the future, they will gain access to all data since the initial attack.

**App code control**:   

*Attack*: An attacker is able to deploy arbitrary code instead of the *application*. They deploy a version of the *application* which sends all user data in clear-text to a server they control.

*Impact:* Whatever the level of protection, the attacker has access to all past and future data. 

*Fix:* *Application* verification and signature could help mitigate that risk. This technology however is difficult to implement in browsers.

## User risks factors

| Threat                  | No Tanker                               | Tanker                                  |
| ----------------------- | --------------------------------------- | --------------------------------------- |
| Unlocked device         | Targeted access to past and future data | Targeted access to past and future data |
| Locked device           | Targeted access to past data            | No access                               |
| User app credentials    | Targeted access to past and future data | No access                               |

### Detailed attacks analysis

**Unlocked device**:   

*Attack*: An attacker gains access to a *user*'s devi*ce. The *user* is logged in the *application*.

*Impact:* The attacker has access to all the *user*'s existing data. If the attacker maintains an open session, they will also have access to future data. 

*Note:* If the *user* revokes the stolen/lost device, the attacker won't have access to future data.

**Locked device**:   

*Attack*: An attacker gains access to a *user*'s *device*. The *user* is not logged in the application.

*Impact:* Depending on the type of *device*, the attacker can have access to part or all of the *user*'s existing data. With Tanker, the attacker won't have access to any data.

**User app credentials**:

*Attack*: An attacker gains access to the credentials used to authenticate with the application.

*Impact*: Without Tanker, the attacker immediately gains access to all user data. With Tanker, granted that the application's credentials cannot be used to verify the user's identity with Tanker, the attacker gains no access.

## Composite attacks with identity verification

Composite attacks are attacks requiring two parties to be compromised by an attacker. They can provide more access than simple attacks described above.

Combinations not listed here have the same impact as the combination of their parts.

| Threat 1                                           | Threat 2              | Tanker                               |
| -------------------------------------------------- | --------------------- | ------------------------------------ |
| Identities + data access                           | Tanker control        | Full access to past data             |
| Identities + repeatable data access                | Tanker control        | Full access to past and future data  |
| App secret + repeatable data access                | Tanker control        | Full access to future data    |
| Identities access                                  | Locked device         | Targeted access to past data  |
| User app credentials                               | Locked device         | Targeted access to past data  |


### Detailed attacks analysis

**Identities + data access + Tanker control**

*Attack*: The attacker gets the [User Secret] from the *identities*. They get the encrypted [Verification Key] from *Tanker*'s identity verification service. Using both of those, they are able to decrypt the [Verification Key]. Using the [Device Encryption Key Pair] contained in the [Verification Key] and the encrypted [User Encryption Key Pair] in the *ghost device*'s creation block, they get the [User Encryption Key Pair]. Using the [User Encryption Key Pair] and the key publish blocks, they can get the [Resource Encryption Key] needed to decrypt the encrypted user data.

*Impact*: The attacker gains access to all past encrypted data.

**Identities + repeatable data access + Tanker control**

*Attack*: Same as *Identities + data access + Tanker control*, but the attacker can repeatedly access data, and so gain access to future encrypted data as well.

*Impact*: The attacker gains access to all past encrypted data, and to all future encrypted data.

**App secret + repeatable data access + Tanker control**

*Attack*: The attacker uses the [Trustchain Signature Key Pair] contained in the app secret to create new valid *devices* for every *user* in the *Trustchain*, creating an alternate version where they control every single *device*. They then perform a split view attack, showing each *user* a different view of the *Trustchain*: each *user* still sees their own *devices* correctly but every other *users*' *devices* are replaced by the attacker's *devices*. New shared data will now be shared with the attacker's controlled *device*.

*Impact*: The attacker gains access to all future shared data, and remains undetected as long as they are able to maintain the split view.
*Note*: The split view attack prevents the attacked *users* to realize they are being attacked. The attack causes some discrepancies in the way the application works, but those are hard to notice and link to an actual attack. The split view attack also requires a very dedicated attacker, having full control and knowledge of the *Tanker* server.
*Note 2*: this attack is much more difficult to perform, and has less potential impact than *Identities + repeatable data access + Tanker control*.

**Identities access + locked device**

*Attack*: The attacker uses the identity and the locked device to start a Tanker session. They can then decrypt any encrypted data stored locally on the device.

*Impact*: Access to targeted past data. This attack can be mitigated if the *device* is revoked or if encrypted data is not stored locally.

**User app credentials + locked device**

Same as *Identities access + locked device*


## Composite attacks without identity verification 

Combinations not listed here have the same impact as the combination of their parts.

| Threat 1                            | Threat 2              | Tanker                              |
| ----------------------------------- | --------------------- | ----------------------------------- |
| App secret + repeatable data access | Tanker control        | Full access to future data   |
| Identities access                   | Locked device         | Targeted access to past data |

### Detailed attacks analysis

See the corresponding sections in *Composite attacks identity verification* for detailed attack and impacts analysis



