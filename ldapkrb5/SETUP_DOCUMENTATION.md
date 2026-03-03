# Kerberos + LDAP + SSSD Lab Setup Documentation
**Server:** ldapkrb5.mydomain.lan (192.168.149.21)  
**Domain:** ldapkrb5.mydomain.lan  
**Kerberos Realm:** LDAPKRB5.MYDOMAIN.LAN  
**LDAP Base DN:** dc=ldapkrb5,dc=mydomain,dc=lan  
**Setup Date:** 2026-03-03  
**OS:** Ubuntu 24.04 LTS

---

## Overview

This document describes the complete setup of a Kerberos 5 authentication server backed by OpenLDAP, with SSSD configured as a client on the same host to simulate real-world domain-joined behavior. The environment is designed for testing authentication flows, permission levels, and SSH key distribution.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                  ldapkrb5.mydomain.lan (192.168.149.21)         │
│                                                                  │
│  ┌─────────────────┐    ┌──────────────────┐                    │
│  │  OpenLDAP (slapd)│    │  MIT Kerberos 5  │                    │
│  │  Port 389/636   │◄───│  KDC Port 88     │                    │
│  │  Base DN:       │    │  Admin Port 749  │                    │
│  │  dc=ldapkrb5,   │    │  Realm:          │                    │
│  │  dc=mydomain,   │    │  LDAPKRB5.       │                    │
│  │  dc=lan         │    │  MYDOMAIN.LAN    │                    │
│  └────────┬────────┘    └──────────────────┘                    │
│           │ stores principals                                    │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ LDAP Directory Structure                                 │    │
│  │ dc=ldapkrb5,dc=mydomain,dc=lan                          │    │
│  │ ├── ou=People (users: admin1, admin2, dev1, dev2,       │    │
│  │ │              user1, user2)                             │    │
│  │ ├── ou=Groups (admins, developers, users)               │    │
│  │ ├── ou=sudoers (sudo rules per group)                   │    │
│  │ ├── ou=Services                                         │    │
│  │ └── cn=krbContainer (Kerberos principals)               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                           ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ SSSD (System Security Services Daemon)                   │    │
│  │ • id_provider = ldap  (user/group lookup from LDAP)     │    │
│  │ • auth_provider = krb5 (authentication via Kerberos)    │    │
│  │ • sudo_provider = ldap (sudo rules from ou=sudoers)     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│              PAM → SSSD → Kerberos → Ticket Issued              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: System Preparation

### 1.1 Set Hostname and /etc/hosts

The fully qualified domain name (FQDN) must be set correctly because Kerberos and LDAP rely on DNS resolution.

```bash
hostnamectl set-hostname ldapkrb5.mydomain.lan
```

Edit `/etc/hosts`:
```
127.0.0.1 localhost
192.168.149.21 ldapkrb5.mydomain.lan ldapkrb5
```

**Why:** Kerberos uses the hostname to identify service principals. If the hostname doesn't match the KDC entry, ticket requests fail. LDAP clients use the FQDN when connecting over TLS.

### 1.2 Package Installation

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y \
  slapd ldap-utils \
  krb5-kdc krb5-admin-server krb5-kdc-ldap krb5-user \
  sssd sssd-krb5 sssd-ldap sssd-tools \
  libpam-sss libnss-sss libsss-sudo \
  sudo sudo-ldap \
  openssl ca-certificates \
  oddjob-mkhomedir \
  expect fail2ban
```

**Package explanation:**
- `slapd` – OpenLDAP server daemon
- `ldap-utils` – Command-line LDAP tools (ldapsearch, ldapadd, etc.)
- `krb5-kdc` – Kerberos Key Distribution Center
- `krb5-admin-server` – Kerberos admin server (kadmind)
- `krb5-kdc-ldap` – Plugin allowing KDC to store data in LDAP instead of flat files
- `krb5-user` – Kerberos client utilities (kinit, klist, kdestroy)
- `sssd` – SSSD daemon metapackage
- `sssd-krb5` – SSSD Kerberos authentication backend
- `sssd-ldap` – SSSD LDAP identity backend
- `sssd-tools` – SSSD utilities (sss_ssh_authorizedkeys)
- `libpam-sss` – PAM module for SSSD (handles login authentication)
- `libnss-sss` – NSS module for SSSD (user/group name resolution)
- `libsss-sudo` – SSSD sudo integration
- `sudo-ldap` – sudo with LDAP schema (provides sudoRole schema)
- `oddjob-mkhomedir` – Automatically creates home directories on first login
- `fail2ban` – Brute-force protection

**Pre-seed debconf to avoid interactive prompts:**
```bash
debconf-set-selections <<EOF
slapd slapd/domain string ldapkrb5.mydomain.lan
slapd slapd/password1 password 0607
slapd slapd/password2 password 0607
slapd slapd/backend select MDB
krb5-config krb5-config/default_realm string LDAPKRB5.MYDOMAIN.LAN
krb5-config krb5-config/kerberos_servers string ldapkrb5.mydomain.lan
krb5-config krb5-config/admin_server string ldapkrb5.mydomain.lan
EOF
```

---

## Step 2: TLS Certificate Authority Setup

A local Certificate Authority (CA) allows all services to use TLS without paying for public certificates. Clients only need to trust the CA cert.

### 2.1 Generate the CA

```bash
mkdir -p /etc/ssl/ldapkrb5/{ca,server,clients}

# Generate CA private key
openssl genrsa -out /etc/ssl/ldapkrb5/ca/ca.key 4096

# Generate self-signed CA certificate (valid 10 years)
openssl req -new -x509 -days 3650 -key /etc/ssl/ldapkrb5/ca/ca.key \
  -out /etc/ssl/ldapkrb5/ca/ca.crt \
  -subj "/C=US/ST=Lab/L=Lab/O=ldapkrb5 Lab CA/CN=ldapkrb5 Root CA"
```

### 2.2 Generate Server Certificate

```bash
openssl genrsa -out /etc/ssl/ldapkrb5/server/server.key 4096

# Create CSR with Subject Alternative Names (SANs)
cat > /tmp/server_san.cnf <<EOF
[req]
default_bits = 4096
distinguished_name = dn
req_extensions = req_ext

[dn]
CN=ldapkrb5.mydomain.lan

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = ldapkrb5.mydomain.lan
DNS.2 = ldapkrb5
IP.1  = 192.168.149.21
IP.2  = 127.0.0.1
EOF

openssl req -new -key /etc/ssl/ldapkrb5/server/server.key \
  -out /etc/ssl/ldapkrb5/server/server.csr \
  -config /tmp/server_san.cnf

# Sign with CA
openssl x509 -req -days 3650 \
  -in /etc/ssl/ldapkrb5/server/server.csr \
  -CA /etc/ssl/ldapkrb5/ca/ca.crt \
  -CAkey /etc/ssl/ldapkrb5/ca/ca.key \
  -CAcreateserial \
  -out /etc/ssl/ldapkrb5/server/server.crt \
  -extensions req_ext -extfile /tmp/server_san.cnf
```

### 2.3 Install CA into System Trust Store

```bash
cp /etc/ssl/ldapkrb5/ca/ca.crt /usr/local/share/ca-certificates/ldapkrb5-lab-ca.crt
update-ca-certificates
```

### 2.4 Distribute CA to Clients

To trust this setup on a client machine:
```bash
# On the client - copy the CA cert and update trust
scp root@ldapkrb5.mydomain.lan:/etc/ssl/ldapkrb5/ca/ca.crt /usr/local/share/ca-certificates/ldapkrb5-lab-ca.crt
update-ca-certificates
```

**Why SANs matter:** Modern TLS validation requires Subject Alternative Names. A cert with only a CN will be rejected by OpenSSL 1.1+ for hostname verification.

---

## Step 3: OpenLDAP Configuration

### 3.1 Verify Initial Setup

After installation, slapd starts with the base DN from the debconf seed:
```bash
ldapsearch -x -H ldapi:/// -LLL -b "" -s base namingContexts
# Returns: namingContexts: dc=ldapkrb5,dc=mydomain,dc=lan
```

### 3.2 Configure TLS

OpenLDAP uses the OLC (Online Configuration) system - configs are managed via LDAP itself using `ldapmodify` with SASL EXTERNAL auth (root-only):

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/ldapkrb5/ca/ca.crt
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/ldapkrb5/server/server.crt
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/ldapkrb5/server/server.key
-
add: olcTLSVerifyClient
olcTLSVerifyClient: never
EOF
```

Enable LDAPS in `/etc/default/slapd`:
```
SLAPD_SERVICES="ldap:/// ldaps:/// ldapi:///"
```

### 3.3 Load Required Schemas

**Core schemas** (cosine, nis, inetorgperson) are pre-loaded. Additional schemas needed:

**sudo schema** (from sudo-ldap package):
```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f /usr/share/doc/sudo-ldap/schema.olcSudo
```
This provides `sudoRole`, `sudoUser`, `sudoCommand`, etc. object classes.

**openssh-lpk schema** (SSH public keys in LDAP):
```bash
ldapadd -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=openssh-lpk,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: openssh-lpk
olcAttributeTypes: ( 1.3.6.1.4.1.24552.500.1.1.1.13
  NAME 'sshPublicKey'
  EQUALITY octetStringMatch
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
olcObjectClasses: ( 1.3.6.1.4.1.24552.500.1.1.2.0
  NAME 'ldapPublicKey'
  SUP top AUXILIARY
  MAY ( sshPublicKey $ uid ) )
EOF
```

**Kerberos schema** (for Kerberos principal storage in LDAP):
```bash
zcat /usr/share/doc/krb5-kdc-ldap/kerberos.openldap.ldif.gz > /tmp/kerberos_schema.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /tmp/kerberos_schema.ldif
```
This provides `krbPrincipal`, `krbRealmContainer`, `krbPrincipalAux`, etc.

### 3.4 Enable Overlays

**memberOf overlay** - automatically updates a `memberOf` attribute on user entries when they're added to a group:
```bash
# Load the module
ldapadd -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof
EOF

# Attach overlay to database
ldapadd -Y EXTERNAL -H ldapi:/// <<EOF
dn: olcOverlay=memberof,olcDatabase={1}mdb,cn=config
objectClass: olcMemberOf
olcOverlay: memberof
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
olcMemberOfMemberOfAD: memberOf
EOF
```

---

## Step 4: LDAP Directory Population

### 4.1 Organizational Units

The directory tree is organized into OUs (Organizational Units):

```bash
ldapadd -x -D "cn=admin,dc=ldapkrb5,dc=mydomain,dc=lan" -w 0607 <<EOF
dn: ou=People,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: organizationalUnit
ou: Groups

dn: ou=sudoers,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: organizationalUnit
ou: sudoers

dn: ou=Services,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: organizationalUnit
ou: Services
EOF
```

### 4.2 User Accounts

Users are created with these objectClasses:
- `posixAccount` – provides UID, GID, home directory, shell (required for Linux login)
- `inetOrgPerson` – standard person attributes (cn, sn, mail)
- `shadowAccount` – password aging support
- `ldapPublicKey` – allows `sshPublicKey` attribute (from openssh-lpk schema)
- `krbPrincRefAux` – reference link to Kerberos principal (added later)

**Generate password hashes:**
```bash
slappasswd -h {SSHA} -s "Admin1Pass!"
```

**UID/GID assignments:**
| User   | UID   | Primary GID | Group      |
|--------|-------|-------------|------------|
| admin1 | 10001 | 20001       | admins     |
| admin2 | 10002 | 20001       | admins     |
| dev1   | 10003 | 20002       | developers |
| dev2   | 10004 | 20002       | developers |
| user1  | 10005 | 20003       | users      |
| user2  | 10006 | 20003       | users      |

### 4.3 Groups

Groups use `posixGroup` with `memberUid` (RFC 2307 format - SSSD default):

```bash
ldapadd -x -D "cn=admin,..." -w 0607 <<EOF
dn: cn=admins,ou=Groups,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: posixGroup
cn: admins
gidNumber: 20001
memberUid: admin1
memberUid: admin2
EOF
```

**Note:** We chose `posixGroup` (with `memberUid`) over `groupOfNames` (with `member` DNs) because SSSD defaults to RFC 2307 schema which uses `memberUid`. Mixing both structural objectClasses on the same entry is not allowed by LDAP schema rules.

### 4.4 Sudo Rules

Sudo rules are stored in `ou=sudoers` using the `sudoRole` object class:

```
cn=defaults   - Global sudo defaults
cn=admins     - Full ALL access for %admins group
cn=developers - Limited access: systemctl, journalctl only
```

**Why no sudo for users group?** The `simple_allow_groups` in SSSD controls login access. No sudoRole entry for `users` group means they get no sudo commands.

### 4.5 SSH Public Keys

Keys are generated and stored in LDAP:
```bash
ssh-keygen -t ed25519 -f /etc/ldapkrb5/sshkeys/${user}_ed25519 -N ""

# Store in LDAP
ldapmodify -x -D "cn=admin,..." -w 0607 <<EOF
dn: uid=${user},ou=People,...
changetype: modify
add: sshPublicKey
sshPublicKey: ssh-ed25519 AAAA...
EOF
```

Private keys are stored in `/etc/ldapkrb5/sshkeys/` for client distribution.

---

## Step 5: Kerberos Configuration

### 5.1 /etc/krb5.conf

The global Kerberos client configuration. All systems (KDC and clients) need this:

```ini
[libdefaults]
    default_realm = LDAPKRB5.MYDOMAIN.LAN
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false

[realms]
    LDAPKRB5.MYDOMAIN.LAN = {
        kdc = ldapkrb5.mydomain.lan
        admin_server = ldapkrb5.mydomain.lan
        database_module = openldap_ldapconf
    }

[domain_realm]
    .ldapkrb5.mydomain.lan = LDAPKRB5.MYDOMAIN.LAN

[logging]
    default = SYSLOG:INFO:DAEMON

[dbmodules]
    openldap_ldapconf = {
        db_library = kldap
        ldap_kerberos_container_dn = cn=krbContainer,dc=ldapkrb5,dc=mydomain,dc=lan
        ldap_kdc_dn = cn=krbadmin,dc=ldapkrb5,dc=mydomain,dc=lan
        ldap_kadmind_dn = cn=krbadmin,dc=ldapkrb5,dc=mydomain,dc=lan
        ldap_service_password_file = /etc/krb5kdc/service.keyfile
        ldap_servers = ldaps://ldapkrb5.mydomain.lan
    }
```

**Key settings explained:**
- `rdns = false` – Don't use reverse DNS for canonicalization (prevents failures in lab without proper rDNS)
- `forwardable = true` – Tickets can be forwarded to remote services
- `database_module = openldap_ldapconf` – Points KDC to use LDAP backend instead of flat files

### 5.2 /etc/krb5kdc/kdc.conf

KDC-specific configuration:

```ini
[realms]
    LDAPKRB5.MYDOMAIN.LAN = {
        kadmind_port = 749
        max_life = 12h 0m 0s
        max_renewable_life = 7d 0h 0m 0s
        master_key_type = aes256-cts
        supported_enctypes = aes256-cts:normal aes128-cts:normal
        database_module = openldap_ldapconf
    }
```

**Important:** `default_principal_flags = +preauth` was intentionally removed because adding it requires clients to send pre-authentication data on all requests, which can cause issues with some older clients and SSSD configurations.

### 5.3 Kerberos LDAP Service Account

The KDC needs credentials to write to LDAP. A dedicated service account `cn=krbadmin` is created:

```bash
# Create the account
ldapadd -x -D "cn=admin,..." -w 0607 <<EOF
dn: cn=krbadmin,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: person
objectClass: simpleSecurityObject
cn: krbadmin
sn: krbadmin
userPassword: {SSHA}...
EOF

# Store password in encrypted keyfile for KDC use
kdb5_ldap_util stashsrvpw -f /etc/krb5kdc/service.keyfile \
  "cn=krbadmin,dc=ldapkrb5,dc=mydomain,dc=lan"
```

LDAP ACL for krbadmin to manage the krbContainer:
```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<EOF
dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to dn.subtree="cn=krbContainer,dc=ldapkrb5,dc=mydomain,dc=lan"
  by dn.exact="cn=krbadmin,dc=ldapkrb5,dc=mydomain,dc=lan" manage
  by dn.exact="cn=admin,dc=ldapkrb5,dc=mydomain,dc=lan" manage
  by * none
EOF
```

### 5.4 Initialize Kerberos Realm in LDAP

```bash
kdb5_ldap_util -D "cn=admin,dc=ldapkrb5,dc=mydomain,dc=lan" -w 0607 \
  -H ldaps://ldapkrb5.mydomain.lan \
  create -subtrees "dc=ldapkrb5,dc=mydomain,dc=lan" \
  -r LDAPKRB5.MYDOMAIN.LAN -s
# Enter master key: 0607kerbmaster
```

This creates `cn=krbContainer,dc=ldapkrb5,dc=mydomain,dc=lan` and `cn=LDAPKRB5.MYDOMAIN.LAN,cn=krbContainer,...` in LDAP.

### 5.5 Creating Principals

```bash
# Regular user principals
kadmin.local -q "addprinc -pw Admin1Krb! admin1"
kadmin.local -q "addprinc -pw Dev1Krb123 dev1"
kadmin.local -q "addprinc -pw User1Krb99 user1"

# Admin principals (with /admin instance for kadmin access)
kadmin.local -q "addprinc -pw Admin1Krb! admin1/admin"

# Service principals
kadmin.local -q "addprinc -randkey host/ldapkrb5.mydomain.lan"
kadmin.local -q "addprinc -randkey ldap/ldapkrb5.mydomain.lan"

# Export to keytab for services
kadmin.local -q "ktadd -k /etc/krb5.keytab host/ldapkrb5.mydomain.lan"
kadmin.local -q "ktadd -k /etc/krb5.keytab ldap/ldapkrb5.mydomain.lan"
```

### 5.6 Linking LDAP Users to Kerberos Principals

Each LDAP user entry is linked to its Kerberos principal using `krbPrincRefAux` objectclass:

```bash
ldapmodify -x -D "cn=admin,..." -w 0607 <<EOF
dn: uid=admin1,ou=People,dc=ldapkrb5,dc=mydomain,dc=lan
changetype: modify
add: objectClass
objectClass: krbPrincRefAux
-
add: krbPrincipalReferences
krbPrincipalReferences: krbPrincipalName=admin1@LDAPKRB5.MYDOMAIN.LAN,
  cn=LDAPKRB5.MYDOMAIN.LAN,cn=krbContainer,dc=ldapkrb5,dc=mydomain,dc=lan
EOF
```

**Why `krbPrincRefAux` not `krbPrincipalAux`?**
- `krbPrincipalAux` makes the KDC treat the LDAP user entry AS a Kerberos principal, causing duplicate principal entries (one in krbContainer, one in ou=People)
- `krbPrincRefAux` is a reference-only objectclass - it points to the principal in krbContainer without becoming one itself
- This gives users a dual identity (LDAP user attributes + Kerberos link) without confusing the KDC

---

## Step 6: SSSD Configuration

SSSD acts as an intermediary between the OS and identity/authentication services. On this host it acts as a **client** connecting to the LDAP and Kerberos servers (which happen to be on the same machine).

### 6.1 /etc/sssd/sssd.conf

```ini
[sssd]
services = nss, pam, sudo, ssh
domains = ldapkrb5.mydomain.lan

[domain/ldapkrb5.mydomain.lan]
# Who handles identity (user/group lookups)
id_provider = ldap
# Who handles authentication (password verification)
auth_provider = krb5
# Who handles sudo rules
sudo_provider = ldap
# Who controls access
access_provider = simple

# LDAP connection settings
ldap_uri = ldaps://ldapkrb5.mydomain.lan
ldap_search_base = dc=ldapkrb5,dc=mydomain,dc=lan
ldap_default_bind_dn = cn=sssduser,dc=ldapkrb5,dc=mydomain,dc=lan
ldap_default_authtok = sssdpass0607
ldap_tls_cacert = /etc/ssl/ldapkrb5/ca/ca.crt
ldap_tls_reqcert = demand
ldap_schema = rfc2307

# Kerberos settings
krb5_realm = LDAPKRB5.MYDOMAIN.LAN
krb5_server = ldapkrb5.mydomain.lan
krb5_renewable_lifetime = 7d
krb5_lifetime = 24h
krb5_use_fast = try

# Access control
simple_allow_groups = admins, developers, users

# Sudo from LDAP
ldap_sudo_search_base = ou=sudoers,dc=ldapkrb5,dc=mydomain,dc=lan

# Caching (allows offline login)
cache_credentials = true
krb5_store_password_if_offline = true
```

**Provider flow explanation:**

When a user tries to log in:
1. PAM calls SSSD
2. SSSD queries LDAP for the user's existence and attributes (`id_provider = ldap`)
3. SSSD sends the password to the Kerberos KDC (`auth_provider = krb5`)
4. KDC validates the password and issues a TGT (Ticket Granting Ticket)
5. If login succeeds, SSSD caches credentials for offline use

### 6.2 SSSD Service Account

A read-only LDAP account (`cn=sssduser`) is created for SSSD to query LDAP:

```bash
ldapadd -x -D "cn=admin,..." -w 0607 <<EOF
dn: cn=sssduser,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: person
objectClass: simpleSecurityObject
cn: sssduser
sn: sssduser
userPassword: {SSHA}...
EOF
```

LDAP ACLs grant sssduser read access to People, Groups, and sudoers OUs.

### 6.3 NSS Configuration (/etc/nsswitch.conf)

```
passwd:     files systemd sss
group:      files systemd sss
shadow:     files sss
sudoers:    files sss
```

**Order matters:** `files` first means local `/etc/passwd` and `/etc/group` take priority. `sss` is checked next from SSSD/LDAP.

### 6.4 PAM Configuration

`pam-auth-update` manages `/etc/pam.d/common-auth` and related files. After enabling `sss` and `mkhomedir`:

```
# common-auth
auth  [success=2 default=ignore]  pam_unix.so nullok
auth  [success=1 default=ignore]  pam_sss.so use_first_pass
auth  requisite                   pam_deny.so
auth  required                    pam_permit.so

# common-session  
session  optional  pam_sss.so
session  optional  pam_mkhomedir.so
```

The `[success=2 default=ignore]` on pam_unix means: if Unix auth succeeds, skip 2 modules (pam_sss and pam_deny); if it fails, try pam_sss.

### 6.5 SSH Integration

`/etc/ssh/sshd_config.d/10-ldap-keys.conf`:
```
AuthorizedKeysCommand /usr/bin/sss_ssh_authorizedkeys %u
AuthorizedKeysCommandUser nobody
GSSAPIAuthentication yes
GSSAPICleanupCredentials yes
```

`sss_ssh_authorizedkeys` queries SSSD which reads the `sshPublicKey` attribute from LDAP for the specified user.

---

## Step 7: Hardening

### 7.1 Disable Anonymous LDAP Binds

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=config
changetype: modify
replace: olcDisallows
olcDisallows: bind_anon
EOF
```

All LDAP access now requires authentication.

### 7.2 Enforce TLS 1.2+

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=config
changetype: modify
replace: olcTLSProtocolMin
olcTLSProtocolMin: 3.3
EOF
```

TLS 3.3 = TLS 1.2 in OpenSSL numbering.

### 7.3 Firewall (UFW)

```bash
ufw default deny incoming
ufw allow 22/tcp    # SSH
ufw allow 88/tcp    # Kerberos KDC TCP
ufw allow 88/udp    # Kerberos KDC UDP
ufw allow 749/tcp   # Kerberos Admin
ufw allow 636/tcp   # LDAPS (secure)
ufw allow from 192.168.149.0/24 to any port 389  # LDAP local subnet only
ufw enable
```

Plain LDAP (389) is restricted to the local network. All remote access should use LDAPS (636).

### 7.4 SSH Hardening

`/etc/ssh/sshd_config.d/20-hardening.conf`:
```
PermitRootLogin prohibit-password
MaxAuthTries 3
LoginGraceTime 30
PermitEmptyPasswords no
X11Forwarding no
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,...
MACs hmac-sha2-512-etm@openssh.com,...
KexAlgorithms curve25519-sha256,...
LogLevel VERBOSE
```

### 7.5 Sudo Hardening

`/etc/sudoers.d/hardening`:
```
Defaults    requiretty
Defaults    logfile="/var/log/sudo.log"
Defaults    log_input,log_output
Defaults    iolog_dir=/var/log/sudo-io
Defaults    timestamp_timeout=5
Defaults    passwd_tries=3
```

### 7.6 Kerberos Admin ACL

`/etc/krb5kdc/kadm5.acl`:
```
kadmin/admin@LDAPKRB5.MYDOMAIN.LAN    *
admin1/admin@LDAPKRB5.MYDOMAIN.LAN    *
admin2/admin@LDAPKRB5.MYDOMAIN.LAN    *
# Users can only change their own password
dev1@LDAPKRB5.MYDOMAIN.LAN           cpw dev1@LDAPKRB5.MYDOMAIN.LAN
```

### 7.7 Kernel Hardening (/etc/sysctl.d/99-hardening.conf)

```
net.ipv4.conf.all.rp_filter = 1           # Reverse path filtering
net.ipv4.conf.all.accept_redirects = 0    # Reject ICMP redirects
net.ipv4.conf.all.log_martians = 1        # Log packets from impossible sources
net.ipv4.tcp_syncookies = 1               # SYN flood protection
kernel.randomize_va_space = 2             # Full ASLR
kernel.dmesg_restrict = 1                 # Non-root can't read dmesg
kernel.kptr_restrict = 2                  # Kernel pointer hiding
fs.protected_hardlinks = 1                # Symlink/hardlink attack prevention
fs.protected_symlinks = 1
```

### 7.8 Fail2ban

Protects SSH from brute force:
```ini
[sshd]
enabled = true
maxretry = 5
findtime = 600
bantime = 3600
```

---

## Step 8: Testing and Verification

### 8.1 Test Kerberos Authentication

```bash
# Get a ticket
kinit admin1
# Enter password: Admin1Krb!

# Verify ticket
klist
# Shows: admin1@LDAPKRB5.MYDOMAIN.LAN, valid 24h, renewable 7d

# Destroy ticket
kdestroy
```

### 8.2 Test User Resolution

```bash
id admin1   # uid=10001(admin1) gid=20001(admins) groups=20001(admins)
id dev1     # uid=10003(dev1) gid=20002(developers) groups=20002(developers)
id user1    # uid=10005(user1) gid=20003(users) groups=20003(users)
```

### 8.3 Test Sudo Access

```bash
# As admin1 - should have ALL access
su - admin1 -c "sudo -l"
# Shows: (ALL : ALL) NOPASSWD: ALL

# As dev1 - should have limited access
su - dev1 -c "sudo -l"
# Shows: /usr/bin/systemctl restart *, /usr/bin/journalctl, etc.

# As user1 - should have NO sudo
su - user1 -c "sudo -l"
# Shows: Sorry, user user1 may not run sudo on ldapkrb5.
```

### 8.4 Test SSH Key Lookup

```bash
sss_ssh_authorizedkeys admin1
# Returns: ssh-ed25519 AAAA...
```

### 8.5 Test LDAPS Connection

```bash
ldapsearch -x -H ldaps://ldapkrb5.mydomain.lan \
  -D "cn=sssduser,dc=ldapkrb5,dc=mydomain,dc=lan" -w sssdpass0607 \
  -b "ou=People,dc=ldapkrb5,dc=mydomain,dc=lan" uid -LLL
```

### 8.6 Using kadmin Remotely

```bash
kadmin -p kadmin/admin
# Password: kadmin0607
kadmin> listprincs
kadmin> getprinc admin1
```

---

## Reference: User Accounts

See `/etc/ldapkrb5/user_passwords.txt` for all passwords.

| User   | LDAP Password | Kerberos Password | Group      | Sudo Access                          |
|--------|---------------|-------------------|------------|--------------------------------------|
| admin1 | Admin1Pass!   | Admin1Krb!        | admins     | FULL (ALL commands, no password)     |
| admin2 | Admin2Pass!   | Admin2Krb!        | admins     | FULL (ALL commands, no password)     |
| dev1   | Dev1Pass123   | Dev1Krb123        | developers | LIMITED (systemctl, journalctl)      |
| dev2   | Dev2Pass123   | Dev2Krb123        | developers | LIMITED (systemctl, journalctl)      |
| user1  | User1Pass99   | User1Krb99        | users      | NONE                                 |
| user2  | User2Pass99   | User2Krb99        | users      | NONE                                 |

**Service credentials:**
| Service            | Credentials                                   |
|--------------------|-----------------------------------------------|
| LDAP Admin         | cn=admin,dc=... / 0607                       |
| SSSD service acct  | cn=sssduser,dc=... / sssdpass0607            |
| KRB admin          | kadmin/admin@... / kadmin0607                |
| Kerberos master key| 0607kerbmaster                               |

---

## Reference: File Locations

| File/Directory                          | Purpose                              |
|-----------------------------------------|--------------------------------------|
| `/etc/ldap/slapd.d/`                   | slapd OLC config (don't edit directly) |
| `/etc/default/slapd`                   | slapd startup options (LDAPS enable) |
| `/etc/krb5.conf`                       | Kerberos client config (all systems) |
| `/etc/krb5kdc/kdc.conf`               | KDC server config                    |
| `/etc/krb5kdc/kadm5.acl`             | Kerberos admin ACL                   |
| `/etc/krb5kdc/service.keyfile`        | Encrypted LDAP service password      |
| `/etc/krb5.keytab`                    | Host/service keytab                  |
| `/etc/sssd/sssd.conf`                 | SSSD configuration                   |
| `/etc/ssl/ldapkrb5/ca/ca.crt`        | CA certificate (distribute to clients)|
| `/etc/ssl/ldapkrb5/ca/ca.key`        | CA private key (keep secret!)        |
| `/etc/ssl/ldapkrb5/server/server.crt` | Server TLS certificate              |
| `/etc/ssl/ldapkrb5/server/server.key` | Server TLS private key              |
| `/etc/ldapkrb5/sshkeys/`             | SSH key pairs for all users          |
| `/etc/ldapkrb5/user_passwords.txt`   | All user passwords reference         |
| `/etc/ldapkrb5/SETUP_DOCUMENTATION.md`| This document                       |

---

## Troubleshooting

### SSSD not resolving users
```bash
systemctl restart sssd
sssctl domain-status ldapkrb5.mydomain.lan
journalctl -u sssd -f
```

### kinit fails with "Generic preauthentication failure"
Check KDC logs:
```bash
journalctl -u krb5-kdc -n 50
# Check the principal exists:
kadmin.local -q "getprinc admin1"
```

### LDAP TLS connection refused
```bash
openssl s_client -connect ldapkrb5.mydomain.lan:636 -CAfile /etc/ssl/ldapkrb5/ca/ca.crt
```

### Sudo rules not applying
```bash
sssctl cache-expire -G  # Clear SSSD group cache
sudo -l -U admin1       # Check as root
```

### SSSD offline mode
```bash
sssctl domain-status ldapkrb5.mydomain.lan -o  # Show offline status
```

---

## Adding New Users (Procedure)

1. **Create LDAP entry:**
```bash
ldapadd -x -D "cn=admin,dc=ldapkrb5,dc=mydomain,dc=lan" -w 0607 <<EOF
dn: uid=newuser,ou=People,dc=ldapkrb5,dc=mydomain,dc=lan
objectClass: posixAccount
objectClass: inetOrgPerson
objectClass: shadowAccount
objectClass: ldapPublicKey
uid: newuser
uidNumber: 10007
gidNumber: 20003
cn: New User
sn: User
homeDirectory: /home/newuser
loginShell: /bin/bash
userPassword: {SSHA}...
EOF
```

2. **Add to group:**
```bash
ldapmodify -x -D "cn=admin,..." -w 0607 <<EOF
dn: cn=users,ou=Groups,dc=ldapkrb5,dc=mydomain,dc=lan
changetype: modify
add: memberUid
memberUid: newuser
EOF
```

3. **Create Kerberos principal:**
```bash
kadmin -p kadmin/admin -q "addprinc -pw TempPass123 newuser"
```

4. **Link to LDAP entry:**
```bash
ldapmodify -x -D "cn=admin,..." -w 0607 <<EOF
dn: uid=newuser,ou=People,...
changetype: modify
add: objectClass
objectClass: krbPrincRefAux
-
add: krbPrincipalReferences
krbPrincipalReferences: krbPrincipalName=newuser@LDAPKRB5.MYDOMAIN.LAN,cn=LDAPKRB5.MYDOMAIN.LAN,cn=krbContainer,dc=ldapkrb5,dc=mydomain,dc=lan
EOF
```

5. **Clear SSSD cache:**
```bash
sssctl cache-expire -u newuser
```

---

## Client Setup (Future Reference)

To configure another Ubuntu machine as a client:

1. Copy CA certificate:
```bash
scp root@ldapkrb5.mydomain.lan:/etc/ssl/ldapkrb5/ca/ca.crt \
    /usr/local/share/ca-certificates/ldapkrb5-lab-ca.crt
update-ca-certificates
```

2. Copy `/etc/krb5.conf` from server (remove `[dbmodules]` section)

3. Install packages: `sssd sssd-krb5 sssd-ldap libpam-sss libnss-sss libsss-sudo`

4. Configure `/etc/sssd/sssd.conf` (same as server but point to server's IP)

5. Configure NSS and PAM via `pam-auth-update`

6. Start SSSD: `systemctl enable --now sssd`

---

## Important: Critical ACL Fix for Kerberos LDAP Backend

When disabling anonymous LDAP binds (`olcDisallows: bind_anon`), the `cn=krbadmin` service account must have **read access to the base DN** (`dc=ldapkrb5,dc=mydomain,dc=lan`), not just to the `cn=krbContainer` subtree.

**Why:** The MIT Kerberos LDAP backend (kldap) first searches the root of the DIT to locate the `krbContainer`. If the service account cannot search the root entry, the KDC fails to start with "Cannot find master key record in database".

**Correct ACL order:**
```
{0} krbContainer subtree -> krbadmin: manage, admin: manage, *: none
{1} userPassword/shadowLastChange -> (auth rules)
{2} sudoers subtree -> sssduser: read, admin: write, *: none
{3} entire DIT subtree -> krbadmin: read, sssduser: read, admin: write, self: read, anonymous: auth, *: none
```

The key is that ACL {3} (catch-all for the whole tree) must include `krbadmin` in its allow list. ACL {0} only covers searches *within* the krbContainer subtree, not the initial search *for* the krbContainer from the root.

---

## OpenLDAP Accesslog Overlay

The accesslog overlay records all bind and write operations to a separate MDB database. This provides an audit trail for security monitoring.

### What was configured

**Module loaded** into `cn=module{0},cn=config`:
```
olcModuleLoad: accesslog
```

**Separate accesslog database** (`olcDatabase={2}mdb`):
- Suffix: `cn=accesslog`
- Directory: `/var/lib/ldap/accesslog` (create with `mkdir -p` and `chown openldap:openldap`)
- ACL: only EXTERNAL root (local socket) and `cn=admin` can read; everything else denied

**Overlay** on the main database (`olcDatabase={1}mdb`):
```
olcOverlay: accesslog
olcAccessLogDB: cn=accesslog
olcAccessLogOps: bind writes
olcAccessLogSuccess: TRUE
olcAccessLogOld: (objectClass=*)
olcAccessLogOldAttr: userPassword
olcAccessLogOldAttr: krbPrincipalKey
olcAccessLogPurge: 7+00:00 1+00:00
```

- `olcAccessLogOps: bind writes` — logs authentication attempts and any data modifications
- `olcAccessLogSuccess: TRUE` — logs both successful and failed operations
- `olcAccessLogOld` — captures the old value of sensitive attributes before changes (password auditing)
- `olcAccessLogPurge: 7+00:00 1+00:00` — auto-purge entries older than 7 days, checked every 24 hours

### How to apply (LDIF sequence)

**Step 1 — Load module:**
```bash
ldapmodify -Y EXTERNAL -H ldapi:/// <<EOF
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: accesslog
EOF
```

**Step 2 — Create accesslog database directory:**
```bash
mkdir -p /var/lib/ldap/accesslog
chown openldap:openldap /var/lib/ldap/accesslog
chmod 700 /var/lib/ldap/accesslog
```

**Step 3 — Create the accesslog MDB database:**
```bash
ldapadd -Y EXTERNAL -H ldapi:/// <<EOF
dn: olcDatabase=mdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: mdb
olcSuffix: cn=accesslog
olcDbDirectory: /var/lib/ldap/accesslog
olcDbMaxSize: 1073741824
olcAccess: {0}to *
  by dn.exact="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" manage
  by dn.exact="cn=admin,dc=ldapkrb5,dc=mydomain,dc=lan" read
  by * none
EOF
```

**Step 4 — Add accesslog overlay to the main database:**
```bash
ldapadd -Y EXTERNAL -H ldapi:/// <<EOF
dn: olcOverlay=accesslog,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcAccessLogConfig
olcOverlay: accesslog
olcAccessLogDB: cn=accesslog
olcAccessLogOps: bind writes
olcAccessLogSuccess: TRUE
olcAccessLogOld: (objectClass=*)
olcAccessLogOldAttr: userPassword
olcAccessLogOldAttr: krbPrincipalKey
olcAccessLogPurge: 7+00:00 1+00:00
EOF
```

**Note on `olcAccessLogOldAttr` syntax:** Each attribute must be on its own `olcAccessLogOldAttr:` line. Using a comma-separated value on a single line causes a syntax error and slapd will refuse the operation.

### Querying the accesslog

Read bind events (authentication attempts):
```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=accesslog" \
  "(objectClass=auditBind)" reqDN reqResult reqStart
```

Read write events (password changes, user modifications):
```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=accesslog" \
  "(objectClass=auditModify)" reqDN reqResult reqStart reqOld
```

Filter for a specific user's bind attempts:
```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=accesslog" \
  "(&(objectClass=auditBind)(reqDN=uid=admin1,ou=People,dc=ldapkrb5,dc=mydomain,dc=lan))" \
  reqResult reqStart reqEnd
```

Show failed binds only (reqResult != 0):
```bash
ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=accesslog" \
  "(&(objectClass=auditBind)(!(reqResult=0)))" reqDN reqResult reqStart
```

**Log entry fields:**
| Field | Meaning |
|---|---|
| `reqDN` | The DN that performed the operation |
| `reqResult` | LDAP result code (0 = success, 49 = invalid credentials) |
| `reqStart` / `reqEnd` | Timestamp of operation |
| `reqMethod` | SIMPLE or SASL |
| `reqOld` | Previous attribute value (for write ops, if captured by `olcAccessLogOldAttr`) |

### Dump via slapcat (offline)
```bash
slapcat -b "cn=accesslog"
```

---

## SSSD Access Filter — Secure Configuration

### Why the initial filter was insufficient

The original filter only checked `gidNumber`, which is just one attribute. A filter is only as strong as the conditions it checks — a single-attribute filter can pass accounts that should not have login access (e.g., service accounts that happen to share a GID, or accounts with locked shells).

### Layered filter approach

The final filter in `/etc/sssd/sssd.conf` requires **all** of the following to be true simultaneously:

```
ldap_access_filter = (&
  (objectClass=posixAccount)
  (uidNumber>=10001)
  (uidNumber<=10006)
  (homeDirectory=*)
  (loginShell=/bin/bash)
  (|(gidNumber=20001)(gidNumber=20002)(gidNumber=20003))
)
```

| Condition | Purpose |
|---|---|
| `objectClass=posixAccount` | Excludes service entries that don't carry this class |
| `uidNumber>=10001` | Excludes system UIDs (root=0, daemon accounts <1000) |
| `uidNumber<=10006` | Restricts to the provisioned user range only |
| `homeDirectory=*` | Must have a home directory attribute set |
| `loginShell=/bin/bash` | Accounts locked with `/sbin/nologin` or `/bin/false` are denied |
| `gidNumber=20001/20002/20003` | Must belong to admins, developers, or users group |

### Search base scoping

In addition to the filter, search bases were scoped per object type:

```ini
ldap_user_search_base  = ou=People,dc=ldapkrb5,dc=mydomain,dc=lan
ldap_group_search_base = ou=Groups,dc=ldapkrb5,dc=mydomain,dc=lan
ldap_sudo_search_base  = ou=sudoers,dc=ldapkrb5,dc=mydomain,dc=lan
```

This means SSSD never searches outside `ou=People` for user identities. Service accounts (`cn=sssduser`, `cn=krbadmin`) live in the root of the DIT, not under `ou=People`, so they are structurally excluded from ever being resolved as login users — even before the filter is evaluated.

### How to disable a user

Change their `loginShell` in LDAP to `/sbin/nologin`:

```bash
ldapmodify -D "cn=admin,dc=ldapkrb5,dc=mydomain,dc=lan" -W <<EOF
dn: uid=user1,ou=People,dc=ldapkrb5,dc=mydomain,dc=lan
changetype: modify
replace: loginShell
loginShell: /sbin/nologin
EOF
```

Then expire the SSSD cache for that user:
```bash
sssctl cache-expire -u user1
```

The user will be denied at next login attempt. Their Kerberos principal remains intact — re-enable by restoring `/bin/bash`.

### Production tuning

In a real environment, use different `ldap_access_filter` values per host to implement role-based host access:

```ini
# Production server — admins only
ldap_access_filter = (&(objectClass=posixAccount)(uidNumber>=10001)(uidNumber<=10006)(homeDirectory=*)(loginShell=/bin/bash)(gidNumber=20001))

# Dev server — admins and developers
ldap_access_filter = (&(objectClass=posixAccount)(uidNumber>=10001)(uidNumber<=10006)(homeDirectory=*)(loginShell=/bin/bash)(|(gidNumber=20001)(gidNumber=20002)))

# Bastion / general — all groups
ldap_access_filter = (&(objectClass=posixAccount)(uidNumber>=10001)(uidNumber<=10006)(homeDirectory=*)(loginShell=/bin/bash)(|(gidNumber=20001)(gidNumber=20002)(gidNumber=20003)))
```

This is the standard pattern for CCDC-style environments where different hosts should only be accessible to specific teams.
