# Glossary

- *application*: any application, web, desktop, or mobile, tat uses the *Tanker SDK* to protect *data* for its *user*s
- *application server*: server supporting the *application*, responsible for the *application*'s business logic, *data* storage, and user authentication
- *block*: an atomic element of a *Trustchain*, signed by its *device* author and linked to other *block*s
- *customer*: *Tanker*'s customer, the owner of the *application*, served to its *user*s
- *data*: any data (including files), owned by the *user*, stored by the *customer* to be used in the *application*
- *device*: a web browser, mobile device, or computer on which the *application* runs
- *group member*: a *user* being part of a *user group*
- *Tanker*: provider of the *Tanker SDK*, the *Trustchain*, and the *unlock service*
- *Tanker SDK*: a privacy SDK, integrated by the *customer* into the *application*, allowing to encrypt, decrypt, and share data between *user*s
- *Tanker server*: server hosting the *Trustchain*, and the *unlock service*
- *Trustchain*: a tamper-proof, append-only cryptographic log of chained and signed *block*s
- *unlock device*: a 'virtual' device associated with a *user*, it is the public part of an *unlock key* registered on the *Trustchain*
- *unlock key*: a key stored either by the *user* or by the *unlock service* to validate new *device* creations
- *unlock service*: an optional *Tanker* service involved in *device* management.
- *user*: a user of the *application*, owning one or more *device*s
- *user group*: a group of *user*s, the *group member*s, created and managed by *user*s. *Data* shared with a *user group* is accessible to all its *group member*s
- *resource*: same as *data*
