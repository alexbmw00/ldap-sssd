# Autenticação com PAM + SSSD + OpenLDAP

## LDAP - OpenSUSE

Instale os pacotes referentes ao OpenLDAP e suas dependências:

```
zypper install openldap2 openldap2-client sssd-ldap
```

Crie um certificado TLS pois o SSSD exige:

```
mkdir -p /etc/openldap/tls/
openssl req -x509 -newkey rsa:4096 -nodes -keyout /etc/openldap/tls/key.pem -out /etc/openldap/tls/cert.pem -days 365
chown -R ldap: /etc/openldap/tls
```

Configure o arquivo **/etc/openldap/slapd.conf** para configurar os parâmetros iniciais do OpenLDAP:

```
pidfile         /run/slapd/slapd.pid
argsfile        /run/slapd/slapd.args

include /etc/openldap/schema/core.schema
include /etc/openldap/schema/cosine.schema
include /etc/openldap/schema/inetorgperson.schema
include /etc/openldap/schema/rfc2307bis.schema
include /etc/openldap/schema/yast.schema

modulepath /usr/lib64/openldap
moduleload back_mdb.la

access to dn.base=""
        by * read

access to dn.base="cn=Subschema"
        by * read

access to attrs=userPassword,userPKCS12
        by self write
        by * auth

access to attrs=shadowLastChange
        by self write
        by * read

access to *
        by * read

database     mdb
suffix       "dc=ldap,dc=example,dc=com"
rootdn       "cn=admin,dc=ldap,dc=example,dc=com"
rootpw       {SSHA}E7SBIzuFwXzf4AdYnWBhft9ynWnIWqww
directory    /var/lib/ldap
index        objectClass eq

TLSCACertificateFile /etc/openldap/tls/cert.pem
TLSCertificateFile /etc/openldap/tls/cert.pem
TLSCertificateKeyFile /etc/openldap/tls/key.pem
```

### Verificar conexão simples:

```
ldapsearch -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -b 'dc=ldap,dc=example,dc=com' -LLL
```

### Verificar conexão TLS com STARTTLS:

```
LDAPTLS_REQCERT=never ldapsearch -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -b 'dc=ldap,dc=example,dc=com' -LLL -ZZ
```

### Popular o OpenLDAP:

Popule o OpenLDAP com os domínios, grupos e usuários:

## Debian

Instale os pacotes necessários para a configuração do SSSD:

```
apt-get install sssd libpam-sss libnss-sss 
```

Verifique se o arquivo **/etc/nssswitch.conf** esteja da seguinte forma:

```
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         compat sss
group:          compat sss
shadow:         compat sss
gshadow:        files

hosts:          files dns
networks:       files

protocols:      db files
services:       db files sss
ethers:         db files
rpc:            db files

netgroup:       nis sss
sudoers:        files sss
```

Uma boa configuração do SSSD inicial, com cache e tentativas pode ser a seguinte em **/etc/sssd/sssd.conf**:

```
[nss] 
filter_groups = root 
filter_users = root 
reconnection_retries = 3 

[pam] 
reconnection_retries = 3 
offline_credentials_expiration = 3 

[sssd] 
config_file_version = 2 
reconnection_retries = 3 
sbus_timeout = 30 
services = nss, pam 
domains = ldap 
debug_level = 5 
certificate_verification = no_verification

[domain/ldap] 
chpass_provider = ldap 
auth_provider = ldap 
ldap_schema = rfc2307 
id_provider = ldap 
enumerate = true 
cache_credentials = true
offline_credentials_expiration = 3
# Necessário para autoassinados
ldap_tls_reqcert = never
# Opcional para limitar a base de busca
ldap_user_search_base = ou=users,dc=ldap,dc=example,dc=com

ldap_uri = ldap://ldap.example.com
```

Reconfigure o PAM para garantir que tudo esteja correto:

```
pam-auth-update
```

Caso seja utilizado um certificado auto-assinado, será preciso adicionar o seguinte parâmetro arquivo do client do OpenLDAP em **/etc/ldap/ldap.conf**:

```
TLS_REQCERT     never
```

Verifique se o Debian consegue encontrar os usuários e grupos através do comando **getent**:

```
getent passwd
getent group
```

## Centos
