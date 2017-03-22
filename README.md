# slapd-contrib

The purpose is to be able to bind simple to the LDAP directory while having the userPassword stored in Kerberos

## Kerberos
### Installation
### Configuration
```
[libdefaults]
    default_realm = <OU>.<DOMAIN>.<TLD>
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    forwardable = true
    udp_preference_limit = 1000000
#    default_tkt_enctypes = des-cbc-md5 des-cbc-crc des3-cbc-sha1
#    default_tgs_enctypes = des-cbc-md5 des-cbc-crc des3-cbc-sha1
#    permitted_enctypes = des-cbc-md5 des-cbc-crc des3-cbc-sha1

[realms]
    <OU>.<DOMAIN>.<TLD> = {
        kdc            = kdc.<ou>.<domain>.<tld>:88
        admin_server   = kdm.<ou>.<domain>.<tld>:749
        default_domain = <ou>.<domain>.<tld>
    }

[domain_realm]
    .<ou>.<domain>.<tld> = <OU>.<DOMAIN>.<TLD>
     <ou>.<domain>.<tld> = <OU>.<DOMAIN>.<TLD>

[logging]
    kdc          = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmin.log
    default      = FILE:/var/log/krb5lib.log
```
## PAM
### Configuration
#### /etc/pam.d/ldap 
```
#%PAM-1.0
auth       include      password-auth
account    include      password-auth
```
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
BASE	      dc=<ou>,dc=<domain>,dc=<tld>
URI	      ldap://ldap1.<ou>.<domain>.<tld> ldap://ldap2.<ou>.<domain>.<tld>
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
authz-regexp "^uid=([^/,]*)/admin,cn=<OU>.<DOMAIN>.<TLD>,cn=GSSAPI,cn=auth" "cn=admin,dc=<ou>,dc=<domain>,dc=<tld>"
# when authenticated with SASL/PLAIN
authz-regexp "^uid=([^/,]*),cn=plain,cn=auth" "uid=$1,ou=users,dc=<ou>,dc=<domain>,dc=<tld>"

access to *
        by dn.exact="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
        by dn.exact="cn=admin,dc=<ou>,dc=<domain>,dc=<tld>" manage
        by * none
```
### user.ldif
Using the {SASL} prefix in the userPassword attribute let us use simple-bind for compatibility purpose
```
dn: uid=<uid>,ou=users,ou=<ou>,dc=<domain>,dc=<tld>
uid: <uid>
userPassword: {SASL}<uid>@<REALM>
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
# kinit <uid>@LABOS.<DOMAIN>.<TLD>
Password for <uid>@LABOS.<DOMAIN>.<TLD>:
# ldapvi -Z -Y GSSAPI
SASL/GSSAPI authentication started
SASL username: <uid>@LABOS.<DOMAIN>.<TLD>
SASL SSF: 56
SASL data security layer installed.
    162 entries read                                                                                                                                                                                                     
No changes.
```
### SASL/PLAIN
```
# ldapvi -Z -Y PLAIN -U <uid>
SASL/PLAIN authentication started

--- SASL login
Type M-h for help on key bindings.

 authorization name: 
authentication name: <uid>
           password: **********
SASL username: <uid>
SASL SSF: 0
      162 entries read                                                                                                                                                                                                     
No changes.
```
### SIMPLE-BIND
```
# ldapvi -Z -D uid=<uid>,ou=users,ou=<ou>,dc=<domain>,dc=<tld>

--- Login
Type M-h for help on key bindings.

Filter or DN: uid=<uid>,ou=users,ou=<ou>,dc=<domain>,dc=<tld>
    Password: **********
    162 entries read                                                                                                                                                                                                     
No changes.
```
