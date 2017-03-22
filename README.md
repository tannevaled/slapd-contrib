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
security ssf=1 update_ssf=112 simple_bind=64

# to get administration rights when authenticated with SASL/GSSAPI
authz-regexp "^uid=([^/,]*)/admin,cn=<ou>.<domain>.<tld>,cn=GSSAPI,cn=auth" "cn=admin,dc=<ou>,dc=<domain>,dc=<tld>"

# to get administration rights when authenticated with SASL/PLAIN
authz-regexp "^uid=([^/,]*),cn=plain,cn=auth" "uid=$1,ou=users,dc=<ou>,dc=<domain>,dc=<tld>"

access to *
        by dn.exact="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
        by dn.exact="cn=admin,dc=<ou>,dc=<domain>,dc=<tld>" manage
        by * none
```
### user.ldif
Using the {SASL} prefix in the userPassword attribute let us use simple-bind for compatibility purpose.

cf http://www.lichteblau.com/ldapvi/cyrus-sasl/sysadmin.html
Realms will be passed to saslauthd as part of the saslauthd protocol, however the way each saslauthd module deals with the situation is different (for example, the LDAP plugin allows you to use the realm to query the server, while the rimap and PAM plugins ignore it entirely).

cf https://web.mit.edu/kerberos/krb5-1.5/krb5-1.5.4/doc/krb5-user/What-is-a-Kerberos-Principal_003f.html
Traditionally, a principal is divided into three parts: the primary, the instance, and the realm. The format of a typical Kerberos V5 principal is primary/instance@REALM. 

```
dn: uid=<uid>,ou=users,ou=<ou>,dc=<domain>,dc=<tld>
uid: <uid>
userPassword: {SASL}<primary>
```
## SASL
### Installation
### Configuration
#### /etc/sysconfig/saslauthd
```
# Directory in which to place saslauthd's listening socket, pid file, and so
# on.  This directory must already exist.
SOCKETDIR=/run/saslauthd

# Mechanism to use when checking passwords.  Run "saslauthd -v" to get a list
# of which mechanism your installation was compiled with the ablity to use.
MECH=pam

# Additional flags to pass to saslauthd on the command line.  See saslauthd(8)
# for the list of accepted flags.
#FLAGS="-O /etc/saslauthd.conf"
```
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
### saslauthd authentication mechanisms
```
# saslauthd -v
saslauthd 2.1.26
authentication mechanisms: getpwent kerberos5 pam rimap shadow ldap httpform
```
### slapd supported SASL Mechanisms
```
# ldapsearch -x -H ldapi:/// -b '' -LLL -s base supportedSASLMechanisms
dn:
supportedSASLMechanisms: GSS-SPNEGO
supportedSASLMechanisms: GSSAPI
supportedSASLMechanisms: EXTERNAL
supportedSASLMechanisms: LOGIN
supportedSASLMechanisms: PLAIN
```
### SASL / GSSAPI
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
### SASL / PLAIN
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
### SASL / SIMPLE-BIND PASSTHRU
```
# ldapvi -Z -D uid=<uid>,ou=users,ou=<ou>,dc=<domain>,dc=<tld>

--- Login
Type M-h for help on key bindings.

Filter or DN: uid=<uid>,ou=users,ou=<ou>,dc=<domain>,dc=<tld>
    Password: **********
    162 entries read                                                                                                                                                                                                     
No changes.
```
