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

The *Tanker* column corresponds to the use of the currently released *Tanker SDK*.

The *Tanker identity verification* column corresponds to the use of a future version of the *Tanker SDK*, in which we would have implemented the client-side verification of identities.

The *Tanker device management* column corresponds to the use of a future version of the *Tanker SDK*, in which we would have implemented a more usable device management API (revocation, devices listing, etc). The application uses these APIs to provide device management actions to its users.

The *Tanker split-view protection* column corresponds to the use of a future version of the *Tanker SDK*, in which we would have implemented a gossiping scheme preventing split view attacks.

## Tanker risk factors


*Attack*: An attacker gains control to all of *Tanker*'s APIs, *Trustchain* blocks, their distribution and the unlock service.


*Impact*: The attacker **cannot access any application data in any way.**

## Application attacks / risk factors

| Threat                                             | No Tanker                           | Tanker                     | Tanker with identity verification |
| -------------------------------------------------- | ----------------------------------- | -------------------------- | --------------------------------- |
| Data leak                                          | Full access to past data            | No access                  | No access                         |
| Repeatable data leak                               | Full access to past and future data | No access                  | No access                         |
| Identities leak + repeatable data leak             | Full access to past and future data | No access                  | No access                         |
| Trustchain private key leak + repeatable data leak | Full access to past data            | No access                  | No access                         |
| Identities modification + data leak                | Full access to past data            | No access                  | No access                         |
| Identities modification + repeatable data leak     | Full access to past and future data | Full access to future data | No access                         |
| Application Code control                           | Full access to future data          | Full access to future data | Full access to future data        |

### Detailed attacks analysis

**Data leak**:

*Attack*: An attacker gains access to all user data stored by the *application*. This attack is not repeatable.   

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. With *Tanker*, they have no access to Tanker-encrypted data. 

**Repeatable data leak**:

*Attack*: An attacker gains access to all user data stored by the *application*. This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. By repeating the same attack, they also have access to all future user data. With *Tanker*, they have no access to Tanker-encrypted data. 

**Identities leak + repeatable data leak**:

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to all stored *Tanker Identities*. This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. By repeating the same attack, they also have access to all future user data. With *Tanker*, they have no access to Tanker-encrypted data. 

**Trustchain private key leak + repeatable data leak**:

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to the private [TSK](#trustchain-keys). This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. By repeating the same attack, they also have access to all future user data. With *Tanker*, they have no access to Tanker-encrypted data. 

**Identity modification + data leak** :

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to all stored *Tanker Identities* and are able to change them. They replace all *user* *Identities* by their own. This attack is not repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. With *Tanker*, they have no access to Tanker-encrypted data. All future user data will however be encrypted for the attacker's *Tanker* account.

**Identity modification + repeatable data leak** :

*Attack*: An attacker gains access to all user data stored by the *application*. They also gain access to all stored *Tanker Identities* and are able to change them. This attack is repeatable. 

*Impact:* Without *Tanker*, the attacker immediately gains access to all user data existing at this point in time. With *Tanker*, all future user data will be encrypted for the attacker's *Tanker* account. If the attacker repeats the data leak attack in the future, they will gain access to all data since the initial attack.
*Fix*: This is fixed with *Tanker identity verification*, which prevents an attacker from impersonating someone else.

**App code control**:   

*Attack*: An attacker is able to deploy arbitrary code instead of the *application*. They deploy a version of the *application* which sends all user data in clear to a server they control.

*Impact:* Whatever the level of protection, the attacker has access to all past and future data. 

*Fix:* *Application* verification like Apple does for its App Store could mitigate that risk.

## User leaks / risks factors

| Threat                  | No Tanker                               | Tanker                                  | Tanker device management     |
| ----------------------- | --------------------------------------- | --------------------------------------- | ---------------------------- |
| Unlocked device         | Targeted access to past and future data | Targeted access to past and future data | Targeted access to past data |
| Locked device           | Targeted access to past data            | No access                               | No access                    |
| Password / Email access | Targeted access to past and future data | Targeted access to past and future data | Targeted access to past data |

### Detailed attacks analysis

**Unlocked device**:   

*Attack*: An attacker gains access to a *user*'s devi*ce. The *user* is logged in the *application*.

*Impact:* The attacker has access to all the *user*'s existing data. If the attacker maintains an open session, they will also have access to future data. 

*Fix:* With Tanker device management, if the *user* revokes the stolen/lost device, the attacker won't have access to future data.

**Locked device**:   

*Attack*: An attacker gains access to a *user*'s *device*. The *user* is not logged in the application.

*Impact:* Depending on the type of *device*, the attacker can have access to part or all of the *user*'s existing data. With Tanker, the attacker won't have access to any data.

**Password / Email access**:

*Attack*: An attacker gains access to a *user*'s password or email address.

*Impact*: As the attacker can use the password or the email to open a session, impact is the same as the *Unlocked device* attack.

## Composite attacks with unlock

Composite attacks are attacks requiring two parties to be compromised by an attacker. They can provide more access than simple attacks described above.

Combinations not listed here have the same impact as the combination of their parts.

| Threat 1                                           | Threat 2              | Tanker                               | Tanker with split-view protection    |
| -------------------------------------------------- | --------------------- | ------------------------------------ | ------------------------------------ |
| Identities leak + data leak                        | Tanker control        | Full access to past data             | Full access to past data             |
| Identities leak + repeatable data leak             | Tanker control        | Full access to past and future  data | Full access to past and future data |
| Trustchain private key leak + repeatable data leak | Tanker control        | Full access to future information    | No access                            |
| Identities leak                                    | Locked revoked device | Targeted access to past information  | Targeted access to past information  |

### Detailed attacks analysis

**Identities leak + data leak + Tanker control**

*Attack*: The attacker gets the [US](#user-secret) from the *identities*. They get the encrypted [ULK](#unlock-key) from *Tanker*'s unlock service. Using both of those, they are able to decrypt the [ULK](#unlock-key). Using the [DEK](#device-keys) contained in the [ULK](#unlock-key) and the encrypted [UEK](#user-keys) in the *unlock device*'s creation block, they get the [UEK](#user-keys). Using the [UEK](#user-keys) and the key publish blocks, they can get the [REK](#resource-keys) needed to decrypt the encrypted user data.

*Impact*: The attacker gains access to all past encrypted data.

**Identities leak + repeatable data leak + Tanker control**

*Attack*: Same as *Identities leak + data leak + Tanker control*, but the attacker can repeat the data leak, and so gain access to future encrypted data as well.

*Impact*: The attacker gains access to all past encrypted data, and to all future encrypted data.

**Trustchain private key leak + repeatable data leak + Tanker control**

*Attack*: The attacker uses the [TSK](#trustchain-keys) to create new valid *devices* for every *user* in the *Trustchain*, creating an alternate version where they control every single *device*. They then perform a split view attack, showing each *user* a different view of the *Trustchain*: each *user* still sees their own *devices* correctly but every other *users*' *devices* are replaced by the attacker's *devices*. New shared data will now be shared with the attacker's controlled *device*.

*Impact*: The attacker gains access to all future shared data undetected as long as they are able to maintain the split view.
*Note*: The split view attack prevents the attacked *users* to realize they are being attacked. The attack causes some discrepancies in the way the application works, but those are hard to notice and link to an actual attack. The split view attack also requires a very dedicated attacker, having full control and knowledge of the *Tanker* server
*Note 2*: this attack is much more difficult to perform, and has less potential impact than *Identities leak + repeatable data leak + Tanker control*.

**Identities leak + locked revoked device**

*Attack*: The attacker gets the [US](#user-secret) from the *identities*. They get the encrypted [LES](#device-id) from the *device*. Using the [US](#user-secret), they decrypt the [LES](#device-id) and gain access to all past [REK](#resource-keys)s the *user* had access to. They then use the [REK](#resource-keys)s on the encrypted data stored on the *device* to access clear data.

*Impact*: Access to targeted past data. This attack can be mitigated if the *device* is wiped when revoked.

## Composite attacks without unlock 

Combinations not listed here have the same impact as the combination of their parts.

| Threat 1                                           | Threat 2              | Tanker                              | Tanker split-view protection        |
| -------------------------------------------------- | --------------------- | ----------------------------------- | ----------------------------------- |
| Trustchain private key leak + repeatable data leak | Tanker control        | Full access to future information   | No access                           |
| Identities leak                                    | Locked revoked device | Targeted access to past information | Targeted access to past information |

### Detailed attacks analysis

See the corresponding sections in *Composite attacks with unlock* for detailed attack and impacts analysis



