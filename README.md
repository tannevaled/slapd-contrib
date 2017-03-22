# slapd-contrib

The purpose is to be able to bind simple to the LDAP directory while having the userPassword stored in Kerberos

## Kerberos
### Installation
## OpenLDAP
### Installation
#### 
```
make base
make init
make conf
```
### Configuration
#### /etc/openldap/ldap.conf
```
BASE	      dc=labos,dc=<DOMAIN>,dc=<TLD>
URI	      ldap://ldap1.labos.<DOMAIN>.<TLD> ldap://ldap2.labos.<DOMAIN>.<TLD>
TLS_CACERTDIR /etc/openldap/certs
TLS_REQCERT   demand
SASL_MECH     PLAIN
# Turning this off breaks GSSAPI used with krb5 when rdns = false
SASL_NOCANON	on
```
#### /etc/openldap/slapd.conf
```


# to get administration rights
# when authenticated with SASL/GSSAPI
authz-regexp "^uid=([^/,]*)/admin,cn=LABOS.<DOMAIN>.<TLD>,cn=GSSAPI,cn=auth" "cn=admin,dc=labos,dc=<DOMAIN>,dc=<TLD>"
# when authenticated with SASL/PLAIN
authz-regexp "^uid=([^/,]*),cn=plain,cn=auth" "uid=$1,ou=users,dc=labos,dc=<DOMAIN>,dc=<TLD>"

access to *
        by dn.exact="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
        by dn.exact="cn=admin,dc=labos,dc=<DOMAIN>,dc=<TLD>" manage
        by * none
```
### user.ldif
Using the {SASL} prefix in the userPassword attribute let us use simple-bind for compatibility purpose
```
dn: uid=<UID>,ou=users,ou=labos,dc=<DOMAIN>,dc=<TLD>
uid: <UID>
userPassword: {SASL}<UID>@<REALM>
```
## SASL
### Installation
### Configuration
#### /etc/sasl2/ldap.conf
```
mech_list: PLAIN
pwcheck_method: saslauthd
saslauthd_path: /run/saslauthd/mux
```
### 
```
# systemctl enable saslauthd
# systemctl start  saslauthd
```
## Tests
### SASL/GSSAPI
```
# kinit <UID>@LABOS.<DOMAIN>.<TLD>
Password for <UID>@LABOS.<DOMAIN>.<TLD>:
# ldapvi -Z -Y GSSAPI
SASL/GSSAPI authentication started
SASL username: <UID>@LABOS.<DOMAIN>.<TLD>
SASL SSF: 56
SASL data security layer installed.
    162 entries read                                                                                                                                                                                                     
No changes.
```
### SASL/PLAIN
```
# ldapvi -Z -Y PLAIN -U <UID>
SASL/PLAIN authentication started

--- SASL login
Type M-h for help on key bindings.

 authorization name: 
authentication name: <UID>
           password: **********
SASL username: <UID>
SASL SSF: 0
      162 entries read                                                                                                                                                                                                     
No changes.
```
### SIMPLE-BIND
```
# ldapvi -Z -D uid=<UID>,ou=users,dc=labos,dc=<DOMAIN>,dc=<TLD>

--- Login
Type M-h for help on key bindings.

Filter or DN: uid=<UID>,ou=users,dc=labos,dc=<DOMAIN>,dc=<TLD>
    Password: **********
    162 entries read                                                                                                                                                                                                     
No changes.
```
