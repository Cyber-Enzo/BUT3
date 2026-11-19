# Installation minimale d'un serveur LDAPS sur Debian 12

Variables d'exemple (à adapter) : FQDN `ldap.example.local`, IP `192.168.10.10`, base `dc=example,dc=local`.

## 1. Installation

```bash
apt update && apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```

Réponses : **No** (omit config) → DNS domain `example.local` → organisation `Example` → mot de passe admin → backend **MDB** → purge **No** → move old DB **Yes**.

## 2. Certificat (auto-signé avec SAN)

```bash
mkdir -p /etc/ldap/tls && cd /etc/ldap/tls

openssl req -x509 -newkey rsa:2048 -nodes -days 825 -sha256 \
  -keyout ldap.key -out ldap.crt \
  -subj "/CN=ldap.example.local" \
  -addext "subjectAltName=DNS:ldap.example.local,IP:192.168.10.10" \
  -addext "basicConstraints=CA:TRUE"

chown root:openldap ldap.key ldap.crt
chmod 640 ldap.key
chmod 644 ldap.crt
```

> `ldap.crt` sera aussi le certificat à importer comme autorité côté client (Stormshield).

## 3. Activation de LDAPS

```bash
cat > tls.ldif <<'EOF'
dn: cn=config
changetype: modify
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/tls/ldap.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/tls/ldap.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/tls/ldap.key
EOF

ldapmodify -Y EXTERNAL -H ldapi:/// -f tls.ldif
```

Dans `/etc/default/slapd` :

```bash
SLAPD_SERVICES="ldaps:/// ldapi:///"
```

```bash
systemctl restart slapd
ss -tlnp | grep 636
```

Client local, dans `/etc/ldap/ldap.conf` :

```
BASE       dc=example,dc=local
URI        ldaps://ldap.example.local
TLS_CACERT /etc/ldap/tls/ldap.crt
```

Ajouter le FQDN dans `/etc/hosts` si pas de DNS :

```bash
echo "192.168.10.10 ldap.example.local" >> /etc/hosts
```

## 4. Contenu de l'annuaire

Hash du mot de passe du compte de service :

```bash
slappasswd -h {SSHA}
```

Fichier `base.ldif` (remplacer `{SSHA}xxxx`) :

```ldif
dn: ou=users,dc=example,dc=local
objectClass: organizationalUnit
ou: users

dn: ou=groups,dc=example,dc=local
objectClass: organizationalUnit
ou: groups

dn: cn=svc-reader,dc=example,dc=local
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: svc-reader
userPassword: {SSHA}xxxx

dn: uid=jdupont,ou=users,dc=example,dc=local
objectClass: inetOrgPerson
uid: jdupont
cn: Jean Dupont
sn: Dupont
mail: jdupont@example.local

dn: cn=users-all,ou=groups,dc=example,dc=local
objectClass: groupOfNames
cn: users-all
member: uid=jdupont,ou=users,dc=example,dc=local
```

```bash
ldapadd -x -H ldaps://ldap.example.local -D "cn=admin,dc=example,dc=local" -W -f base.ldif

# Mot de passe de l'utilisateur
ldappasswd -x -H ldaps://ldap.example.local -D "cn=admin,dc=example,dc=local" -W -S \
  "uid=jdupont,ou=users,dc=example,dc=local"
```

## 5. Tests

```bash
# Certificat valide ?
openssl s_client -connect ldap.example.local:636 -CAfile /etc/ldap/tls/ldap.crt </dev/null 2>/dev/null | grep "Verify return code"

# Bind du compte de service
ldapwhoami -x -H ldaps://ldap.example.local -D "cn=svc-reader,dc=example,dc=local" -W

# Recherche d'un utilisateur
ldapsearch -x -H ldaps://ldap.example.local -D "cn=svc-reader,dc=example,dc=local" -W \
  -b "ou=users,dc=example,dc=local" "(uid=jdupont)"
```

## Paramètres à fournir au client (ex. Stormshield)

| Paramètre | Valeur |
|---|---|
| Serveur / port | `ldap.example.local` / `636` (SSL) |
| Base DN | `dc=example,dc=local` |
| Compte de lecture | `cn=svc-reader,dc=example,dc=local` |
| Branche utilisateurs | `ou=users,dc=example,dc=local` |
| Branche groupes | `ou=groups,dc=example,dc=local` |
| Certificat à importer | `/etc/ldap/tls/ldap.crt` |
