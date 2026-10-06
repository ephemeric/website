### [[Simplicity]](simplicity.md) [[Complexity]](complexity.md) [[Neomania]](neomania.md) [[Networking]](networking.md) [[Security]](security.md) [[Miscellaneous]](miscellaneous.md)

# SSSD Sudo Policy from 389 Directory Server with mTLS

**Date:** 22 September 2026

## Purpose

This document records the working proof-of-concept for separating Unix
identity from centrally managed sudo policy.

The tested design uses:

-   a local Unix user as the identity source;
-   SSSD as the glue and sudo responder;
-   389 Directory Server as the sudo policy store;
-   LDAPS for transport;
-   a dedicated read-only LDAP bind identity for SSSD;
-   a TLS client certificate as an additional gate to the LDAPS service.

The important architectural result is that **389 Directory Server does
not need to contain the Unix user**. SSSD can resolve the user through
one identity provider while independently retrieving sudo policy from
LDAP.

## Tested systems

Directory server:

``` text
Host:       almabld10a.ephemeric.lan
LDAP name:  ds.ephemeric.lan
Address:    192.168.0.102
389 DS:     instance ds
```

SSSD client:

``` text
Host:       almabld10c.ephemeric.lan
Address:    192.168.0.109
Test user:  slob
UID/GID:    1002
```

389 DS listener arrangement:

``` text
127.0.0.1:389       LDAP plaintext - localhost administration/testing only
192.168.0.102:636   LDAPS - network clients / SSSD
```

The loopback LDAP listener is intentional and remains an administrative
escape hatch when changing LDAPS security.

------------------------------------------------------------------------

## 1. Sudo schema and policy in 389 DS

389 DS already had the sudo schema loaded from its packaged schema.

Verification:

``` bash
dsconf --pwdfile ~/.dspasswd ds schema objectclasses list | grep -i sudo
```

The server reported the `sudoRole` object class, including attributes
such as `sudoUser`, `sudoHost`, `sudoCommand`, `sudoRunAsUser`,
`sudoOption`, and `sudoOrder`.

The sudo policy subtree is:

``` text
ou=SUDOers,dc=ds,dc=ephemeric,dc=lan
```

The proven test rule is:

``` ldif
dn: cn=slob,ou=SUDOers,dc=ds,dc=ephemeric,dc=lan
objectClass: top
objectClass: sudoRole
cn: slob
sudoUser: slob
sudoHost: ALL
sudoCommand: /usr/bin/id
```

A complete SSSD database purge was used once as a controlled experiment
to eliminate stale state. After that clean test, **plain
`sudoUser: slob` was proven to work**. Storing `sudoUser: #1002` is not
required.

Normal SSSD cache invalidation remains:

``` bash
sss_cache -E
```

Deleting `/var/lib/sss/db/*` is not normal cache maintenance.

------------------------------------------------------------------------

## 2. Read-only LDAP service identity

Anonymous LDAP access could connect to the server but could not see the
sudo policy. This was demonstrated by comparing anonymous and Directory
Manager searches.

A dedicated LDAP service identity was therefore created:

``` text
cn=sssd-sudo,ou=Services,dc=ds,dc=ephemeric,dc=lan
```

The account is deliberately separate from Unix user identity.

A narrow ACI on the sudo subtree grants this identity only the access
required to retrieve sudo policy:

``` ldif
dn: ou=SUDOers,dc=ds,dc=ephemeric,dc=lan
changetype: modify
add: aci
aci: (targetattr="cn || objectClass || sudoUser || sudoHost || sudoCommand || sudoRunAs || sudoRunAsUser || sudoRunAsGroup || sudoOption || sudoNotBefore || sudoNotAfter || sudoOrder || description")(version 3.0; acl "SSSD sudo policy read"; allow (read,search,compare) userdn="ldap:///cn=sssd-sudo,ou=Services,dc=ds,dc=ephemeric,dc=lan";)
```

This grants read/search/compare access to selected sudo-policy
attributes, without granting write, add, or delete access.

------------------------------------------------------------------------

## 3. Working SSSD configuration before mTLS

The client exposes SSSD only as a sudo source:

``` text
sudoers: files sss
```

The working domain configuration is based on:

``` ini
[sssd]
services = sudo
domains = ephemeric.lan

[sudo]
debug_level = 9

[domain/ephemeric.lan]
debug_level = 9
id_provider = proxy
proxy_lib_name = files
auth_provider = none
local_auth_policy = only

sudo_provider = ldap
ldap_uri = ldaps://ds.ephemeric.lan
ldap_sudo_search_base = ou=SUDOers,dc=ds,dc=ephemeric,dc=lan

ldap_default_bind_dn = cn=sssd-sudo,ou=Services,dc=ds,dc=ephemeric,dc=lan
ldap_default_authtok_type = password
ldap_default_authtok = <service-account-password>
```

The proxy provider resolves the local `/etc/passwd` user. The LDAP
provider is used independently for sudo policy.

No `use_fully_qualified_names` setting is required.

The Easy-RSA root CA had already been installed in the AlmaLinux system
trust store. SSSD and the OpenLDAP client stack successfully verify
`ds.ephemeric.lan` through that system trust, so no application-specific
CA path is required in this configuration.

------------------------------------------------------------------------

## 4. Baseline sudo proof

SSSD successfully resolved:

``` text
slob -> UID 1002 / GID 1002
```

It authenticated to 389 DS as:

``` text
cn=sssd-sudo,ou=Services,dc=ds,dc=ephemeric,dc=lan
```

and fetched the sudo rule.

The end-to-end result was:

``` bash
sudo -lU slob
```

``` text
User slob may run the following commands on almabld10c:
    (root) /usr/bin/id
```

This proved the important separation:

``` text
/etc/passwd local identity
        |
        v
      SSSD
        |
        +---- identity via proxy/files
        |
        +---- sudo policy via LDAP
                  |
                  v
               389 DS
        |
        v
  SSSD sudo responder
        |
        v
       sudo
```

------------------------------------------------------------------------

## 5. Existing TLS PKI

The relevant PKI hierarchy is:

``` text
Easy-RSA CA
    |
    +-- Ephemeric CA
            |
            +-- ds.ephemeric.lan
            |
            +-- TLS client certificates
```

In the 389 DS NSS database:

``` text
Ephemeric CA    CT,,
Server-Cert     u,u,u
```

For NSS trust flags, the three comma-separated positions are:

``` text
SSL, S/MIME, JAR/XPI
```

For the CA:

-   `C` means trusted CA for TLS server certificates.
-   `T` means trusted CA for TLS client certificates.

Thus:

``` text
CT,,
```

means that `Ephemeric CA` is trusted for both server and client TLS
certificate issuance in the SSL trust category.

The same CA therefore appears when listing both normal TLS CAs and
client-authentication CAs.

------------------------------------------------------------------------

## 6. Existing test client certificate and final certmap behaviour

An existing Easy-RSA test certificate was used:

```text
Subject: CN=ldap-test
Issuer:  CN=Ephemeric CA
EKU:     TLS Web Client Authentication
```

389 DS certificate mapping is configured with:

```text
certmap ephemeric       CN=Ephemeric CA
ephemeric:DNComps
ephemeric:FilterComps   cn
```

Originally, an LDAP entry existed at:

```text
cn=ldap-test,ou=Hosts,dc=ds,dc=ephemeric,dc=lan
```

and SASL EXTERNAL successfully mapped the certificate to that entry.

For the final experiment, that LDAP entry was deliberately deleted:

```bash
ldapdelete -x -H ldap://localhost -D 'cn=Directory Manager' -W 'cn=ldap-test,ou=Hosts,dc=ds,dc=ephemeric,dc=lan'
```

The certificate remains trusted for TLS client authentication, but there is no longer a matching LDAP object for the certificate mapper to resolve.

The resulting SASL EXTERNAL test is:

```bash
LDAPTLS_CERT=/etc/sssd/tls/ldap-test.crt LDAPTLS_KEY=/etc/sssd/tls/ldap-test.key ldapwhoami -H ldaps://ds.ephemeric.lan -Y EXTERNAL
```

Result:

```text
SASL/EXTERNAL authentication started
ldap_sasl_interactive_bind: Invalid credentials (49)
```

This is intentional. The certificate is now usable as an mTLS credential but not as an LDAP identity.

No change to `certmap.conf` was required.

------------------------------------------------------------------------

## 7. SSSD TLS client certificate configuration

For the mTLS experiment, SSSD retained its normal simple-bind identity
and password and gained only these two LDAP TLS options:

``` ini
ldap_tls_cert = /etc/sssd/tls/ldap-test.crt
ldap_tls_key = /etc/sssd/tls/ldap-test.key
```

Conceptually:

``` text
TLS client certificate
        |
        v
permission to establish the LDAPS connection
        |
        v
LDAP simple bind
        |
        v
cn=sssd-sudo,ou=Services,...
        |
        v
ACI determines LDAP authorization
```

This is intentionally **not SASL EXTERNAL**. The certificate is an mTLS
gate; the LDAP simple bind establishes the LDAP authorization identity.

------------------------------------------------------------------------

## 8. Requiring client certificates in 389 DS

The 389 DS security configuration initially showed:

``` text
nssslclientauth: allowed
```

The relevant CLI option was discovered through the command hierarchy:

``` bash
dsconf --pwdfile ~/.dspasswd ds security set --help
```

which documents:

``` text
--tls-client-auth TLS_CLIENT_AUTH
    Configures client authentication requirement (nsSSLClientAuth)
```

Client authentication was changed to required with:

``` bash
dsconf --pwdfile ~/.dspasswd ds security set --tls-client-auth required
```

The resulting policy is:

``` text
nssslclientauth: required
```

------------------------------------------------------------------------

## 9. Direct LDAP A/B proof

With client certificates required, a normal LDAPS Directory Manager bind
without a client certificate failed:

``` bash
ldapsearch -H ldaps://ds.ephemeric.lan -D 'cn=Directory Manager' -W
```

Result:

``` text
ldap_result: Can't contact LDAP server (-1)
```

This demonstrates failure at the TLS connection layer before the LDAP
bind can be used.

Using the existing client certificate succeeded:

``` bash
LDAPTLS_CERT=ldap-test.crt LDAPTLS_KEY=ldap-test.key ldapsearch -H ldaps://ds.ephemeric.lan -Y EXTERNAL
```

The server accepted the TLS client certificate and SASL EXTERNAL used
the mapped `ldap-test` identity.

This independently proved that 389 DS was enforcing the required TLS
client certificate.

------------------------------------------------------------------------

## 10. SSSD positive mTLS proof

With these entries present:

``` ini
ldap_tls_cert = /etc/sssd/tls/ldap-test.crt
ldap_tls_key = /etc/sssd/tls/ldap-test.key
```

and 389 DS requiring client certificates, SSSD logged:

``` text
Option ldap_default_bind_dn has value cn=sssd-sudo,ou=services,dc=ds,dc=ephemeric,dc=lan
Executing simple bind as: cn=sssd-sudo,ou=services,dc=ds,dc=ephemeric,dc=lan
ldap simple bind sent, msgid = 2
Message type: [LDAP_RES_BIND]
Bind result: Success(0), no errmsg set
```

This is significant because SSSD could not have reached the LDAP
simple-bind stage unless the preceding TLS handshake, including required
client-certificate authentication, had succeeded.

The certificate therefore does not replace the LDAP bind identity.

The successful sequence is:

``` text
SSSD
 |
 +-- presents TLS client certificate
 |
 v
389 DS validates certificate
(nsSSLClientAuth = required)
 |
 v
TLS session established
 |
 v
SSSD simple-binds as cn=sssd-sudo,...
 |
 v
Bind result: Success
 |
 v
sudo policy retrieved
```

------------------------------------------------------------------------

## 11. SSSD negative mTLS proof

The two TLS client-certificate settings were then commented out:

``` ini
#ldap_tls_cert = /etc/sssd/tls/ldap-test.crt
#ldap_tls_key = /etc/sssd/tls/ldap-test.key
```

With 389 DS still requiring client certificates:

``` bash
sudo -lU slob
```

returned:

``` text
User slob is not allowed to run sudo on almabld10c.
```

Restoring the client-certificate configuration restores the working
path.

Together with the direct `ldapsearch` A/B test, this proves the
mechanism end-to-end rather than merely inferring it from configuration.

------------------------------------------------------------------------

## 12. Anonymous LDAP access disabled

With mTLS required, possession of a valid client certificate could still establish TLS. Initially, an LDAP client could then remain anonymous and read attributes permitted by existing `userdn="ldap:///anyone"` ACIs.

The relevant server setting was:

```text
nsslapd-allow-anonymous-access: on
```

It was changed to:

```text
nsslapd-allow-anonymous-access: off
```

The resulting server policy is:

```text
nssslclientauth: required
nsslapd-allow-anonymous-access: off
```

Certificate-only anonymous access was then tested directly:

```bash
LDAPTLS_CERT=/etc/sssd/tls/ldap-test.crt LDAPTLS_KEY=/etc/sssd/tls/ldap-test.key ldapsearch -x -H ldaps://ds.ephemeric.lan -b 'dc=ds,dc=ephemeric,dc=lan' '(objectClass=*)'
```

Result:

```text
ldap_bind: Inappropriate authentication (48)
        additional info: Anonymous access is not allowed
```

Thus the certificate passes TLS client authentication but cannot be used to browse LDAP anonymously.

A simple bind using the same TLS certificate still succeeds:

```bash
LDAPTLS_CERT=/etc/sssd/tls/ldap-test.crt LDAPTLS_KEY=/etc/sssd/tls/ldap-test.key ldapwhoami -x -H ldaps://ds.ephemeric.lan -D 'cn=sssd-sudo,ou=Services,dc=ds,dc=ephemeric,dc=lan' -W
```

Result:

```text
dn: cn=sssd-sudo,ou=services,dc=ds,dc=ephemeric,dc=lan
```

The final authentication matrix is therefore:

```text
No certificate + LDAP password
    -> TLS rejected

Certificate only + anonymous LDAP
    -> TLS accepted
    -> LDAP rejected: 48 Anonymous access is not allowed

Certificate + SASL EXTERNAL
    -> TLS accepted
    -> LDAP rejected: 49 Invalid credentials
       (no certificate-mapped LDAP entry)

Certificate + cn=sssd-sudo password
    -> TLS accepted
    -> LDAP simple bind succeeds
    -> sudo policy retrieved
```

The credentials have distinct jobs:

```text
client cert/key        -> admission to TLS
sssd-sudo DN/password  -> LDAP authentication identity
ACI                     -> LDAP authorization
```

------------------------------------------------------------------------

## 13. Final end-to-end sudo proof

With the complete configuration active:

```bash
sudo -lU slob
```

returned:

```text
User slob may run the following commands on almabld10c:
    (root) /usr/bin/id
```

This proves the complete path:

```text
slob (local Unix identity)
        |
        v
      SSSD
        |
        |  ldap-test cert/key
        v
389 DS required mTLS
        |
        |  simple bind:
        |  cn=sssd-sudo + password
        v
LDAP authentication
        |
        |  SUDOers ACI
        v
sudoRole: slob
        |
        v
SSSD sudo responder
        |
        v
(root) /usr/bin/id
```

------------------------------------------------------------------------

## 14. Remaining ACI hardening

The dedicated SUDOers ACI is narrow and working correctly. However, the directory also contains existing `userdn="ldap:///anyone"` ACIs on other branches. Because `anyone` includes authenticated LDAP identities, `cn=sssd-sudo` can still inherit some of those read grants outside `ou=SUDOers`.

Further confinement of `sssd-sudo` so that it can see only `ou=SUDOers` is therefore a future hardening task.

This was deliberately left for later: it is not an unresolved architectural question and is not required to prove the mTLS + simple-bind + SSSD sudo design.

Any future deny/exclusion ACIs should be tested carefully because a deny can take precedence over an allow.

------------------------------------------------------------------------

## 15. Proven architecture



The resulting design is:

``` text
Unix identity
(local files in this test)
        |
        v
      SSSD
        |
        |       shared TLS client certificate
        |                  |
        |                  v
        |            389 DS mTLS gate
        |                  |
        |                  v
        |       shared sssd-sudo bind DN/password
        |                  |
        |                  v
        |            narrow sudo ACI
        |                  |
        +-------> sudo policy in 389 DS
        |
        v
SSSD sudo responder
        |
        v
       sudo
```

The credentials have deliberately separate jobs:

``` text
TLS client certificate  -> permission to establish LDAPS
sssd-sudo credentials   -> LDAP authentication identity
389 DS ACI              -> authorization to read sudo policy
```

The design does not require the LDAP directory to contain the Unix user.

------------------------------------------------------------------------

## 16. Why use a shared certificate?

A per-machine client certificate could provide a unique machine
identity, but doing so properly would also introduce:

-   per-host certificate issuance and renewal;
-   per-host private-key management;
-   directory host objects;
-   certificate-to-host mapping;
-   per-host authorization policy;
-   revocation and lifecycle machinery.

That starts to resemble a machine-account/domain architecture.

For the current requirement, a shared certificate is intentionally
simpler:

``` text
shared certificate = approved SSSD LDAP client class
shared bind account = SSSD sudo reader
```

The certificate is not intended to identify an individual machine.

------------------------------------------------------------------------

## 17. Production cleanup

The `ldap-test` certificate is suitable for proving the mechanism
because it was already known to be a valid TLS client certificate issued
by `Ephemeric CA`.

For normal use, issue a purpose-specific shared certificate, for
example:

``` text
Subject: CN=sssd-sudo-client
Issuer:  CN=Ephemeric CA
EKU:     TLS Web Client Authentication
```

Then configure SSSD with that certificate and key:

``` ini
ldap_tls_cert = /etc/sssd/tls/sssd-sudo-client.crt
ldap_tls_key = /etc/sssd/tls/sssd-sudo-client.key
```

The existing simple-bind configuration remains:

``` ini
ldap_default_bind_dn = cn=sssd-sudo,ou=Services,dc=ds,dc=ephemeric,dc=lan
ldap_default_authtok_type = password
ldap_default_authtok = <service-account-password>
```

No matching LDAP entry is required merely to use the certificate as the
TLS client-authentication gate. For a purpose-specific shared certificate,
do not create a matching LDAP entry if SASL EXTERNAL identity is not desired.

The private key should be readable only by the SSSD process that needs
it; final ownership and permissions should be verified against the SSSD
execution model on the target AlmaLinux release.

------------------------------------------------------------------------

## 18. Final result

The experiment established all of the following:

1.  SSSD can resolve a Unix user independently of the LDAP sudo policy
    directory.
2.  389 DS can hold sudo policy without holding that Unix user's
    identity.
3.  SSSD can authenticate to the sudo-policy directory with a narrow
    read-only service account.
4.  AlmaLinux system PKI trust is sufficient for validating the 389 DS
    server certificate.
5.  389 DS can require a trusted TLS client certificate on LDAPS.
6.  SSSD can present that TLS client certificate while still using a
    normal LDAP simple bind.
7.  The TLS certificate and LDAP bind credentials therefore provide two
    independent gates.
8.  Removing the client certificate breaks sudo-policy retrieval when
    mTLS is required.
9.  Restoring it permits the simple bind and sudo-policy retrieval
    again.
10. The architecture works without per-machine LDAP identities or
    machine-account infrastructure.
11. Anonymous LDAP access can be disabled independently of required mTLS.
12. A trusted client certificate alone cannot browse the directory anonymously.
13. The client certificate can deliberately have no usable SASL EXTERNAL identity.
14. Certificate + `cn=sssd-sudo` simple bind succeeds and retrieves sudo policy.
15. The final end-to-end result remains `(root) /usr/bin/id` for `sudo -lU slob`.
16. Restricting `cn=sssd-sudo` from inheriting other `anyone` read ACIs is a
    separate future hardening task.

The practical target architecture can therefore remain:

``` text
Unix identity/authentication -> chosen identity provider
Privilege policy             -> 389 Directory Server
Glue                         -> SSSD
sudo                         -> sudoers: files sss
LDAP transport               -> LDAPS + required client certificate
Anonymous LDAP               -> disabled
TLS client certificate       -> transport admission only
LDAP policy identity         -> cn=sssd-sudo + password
LDAP authorization           -> SUDOers ACI
```

This keeps identity and privilege policy separate while retaining an
explicit, observable, and debuggable path through each layer.

### [[Please report any broken links via GitHub. Suggestions welcome. Polemics unwelcome.]](https://github.com/ephemeric/website)
