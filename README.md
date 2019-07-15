# Autenticação com PAM + SSSD + OpenLDAP

Existem três máquinas com três sistemas diferentes:

| Servidor | IP          | Papel               |
|----------|-------------|---------------------|
| opensuse | 27.11.90.10 | LDAP Server, Client |
| debian   | 27.11.90.20 | Client              |
| centos   | 27.11.90.30 | Client              |

## openSUSE

Instale os pacotes referentes ao OpenLDAP, SSSD e suas dependências:

```
zypper install -y openldap2 openldap2-client sssd-ldap
```

Caso não utilize DNS:

```
echo '27.11.90.10 ldap.example.com' >> /etc/hosts
```

### TLS

Certificados TLS/SSL são obrigatórios para o funcionamento do SSSD, pois a conexão com o LDAP acontece apenas de forma segura. Você pode criar um simples certificado para testes ou uma CA inteira caso precise deste tipo de exemplo:

#### Certificado Simples

Neste caso apenas os comandos abaixo são necessários:

```
mkdir -p /etc/openldap/tls/
openssl req -x509 -newkey rsa:4096 -nodes -keyout /etc/openldap/tls/key.pem -out /etc/openldap/tls/cert.pem -days 365
chown -R ldap: /etc/openldap/tls
```

#### Criando uma CA

Criar uma CA é um processo relativamente trabalhoso, caso deseje você pode seguir os exemplos no arquivo **ca.md** deste repositório.

#### OpenLDAP

Modifique o arquivo **/etc/openldap/slapd.conf** para configurar os parâmetros iniciais do OpenLDAP:

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

Inicie o openLDAP:

```
systemctl start slapd
```

#### Verificar conexão simples:

```
opensuse:~ # ldapsearch -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -LLL
No such object (32)
```

#### Verificar conexão TLS com STARTTLS:

Caso seja utilizado um certificado auto-assinado, você tem três opções:

Copiar e importar a CA - gerada no exemplo **ca.md** - para a máquina:

```
cp ldap.example.com-cert.pem /usr/share/pki/trust/anchors/
update-ca-certificates -v
```

Alterar o arquivo de parâmetros padrões do client ldap **/etc/openldap/ldap.conf** - isto é necessário por conta do PAM e do SSSD:

```
echo -e 'TLS_REQCERT\tnever' > /etc/openldLDAPTLS_REQCERT=neverap/ldap.conf
```

Ou sempre especificar a variável **LDAPTLS_REQCERT=never** antes do comando:

```
LDAPTLS_REQCERT=never ldapsearch -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -LLL -ZZ
No such object (32)
```

Em ambos os casos procuramos a saída *No such object (32)*. Apensar do susto, isso é um sucesso, pois não tivemos problemas em fechar a conexão TLS.

#### Popular o OpenLDAP:

Popule o OpenLDAP com os domínios, grupos e usuários iniciais disponíveis no diretório **/vagrant/files/ldap**:

```
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f base.ldif
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f users.ldif
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f groups.ldif
# Este arquivo adiciona um membro a um grupo já existente
ldapadd -h localhost -D 'cn=admin,dc=ldap,dc=example,dc=com' -w 123 -f groups-users.ldif
```

#### SSSD

Edite o arquivo **/etc/sssd/sssd.conf**:

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

[domain/ldap] 
chpass_provider = ldap 
auth_provider = ldap 
ldap_schema = rfc2307 
id_provider = ldap 
enumerate = true 
cache_credentials = true
# Necessário para autoassinados
ldap_tls_reqcert = never
# Opcional para limitar a base de busca
ldap_user_search_base = ou=users,dc=ldap,dc=example,dc=com

ldap_uri = ldap://ldap.example.com

```

Reinicie o sssd:

```
systemctl restart sssd
```

Verifique se no arquivo **/etc/nsswitch.conf** ao menos passwd, group e shadow utilizem **sss**:

```
passwd: compat sss
group:  compat sss
shadow: compat sss
```

Verifique se o openSUSE consegue encontrar os usuários e grupos através do comando **getent**:

```
getent passwd
getent group
```

Configure o PAM para utilizar SSS e criar a home dos usuários que se logarem:

```
pam-config -a --sss --mkhomedir --mkhomedir-skel=/etc/skel/
```

Verifique se é possível se logar com o usuário:

```
# Local
su - hector.vido
# SSH
ssh hector.vido@127.0.0.1
```

## Debian

Instale os pacotes necessários para a configuração do SSSD:

```
apt-get install sssd libpam-sss libnss-sss vim ldap-utils
```

Caso não utilize DNS:

```
echo '27.11.90.10 ldap.example.com' >> /etc/hosts
```

Verifique se no arquivo **/etc/nsswitch.conf** ao menos passwd, group e shadow utilizem **sss**:

```
passwd: compat sss
group:  compat sss
shadow: compat sss
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

[domain/ldap] 
chpass_provider = ldap 
auth_provider = ldap 
ldap_schema = rfc2307 
id_provider = ldap 
enumerate = true 
cache_credentials = true
# Necessário para autoassinados
ldap_tls_reqcert = never
# Opcional para limitar a base de busca
ldap_user_search_base = ou=users,dc=ldap,dc=example,dc=com

ldap_uri = ldap://ldap.example.com
```

Garanta que o arquivo seja acessado apenas pelo usuário root e reinicie o serviço:

```
chmod 0600 /etc/sssd/sssd.conf
systemctl restart sssd
```

Caso você queira que os usuários tenham um diretório **home**, você pode criar o arquivo */usr/share/pam-configs/mkhomedir* com o seguinte conteúdo:

```
Name: Create home directory during login
Default: yes
Priority: 900
Session-Type: Additional
Session:
        required        pam_mkhomedir.so umask=0022 skel=/etc/skel
```

Reconfigure o PAM para garantir que tudo esteja correto:

```
pam-auth-update
```

Caso seja utilizado um certificado auto-assinado você tem as mesmas opções do openSUSE, mas neste caso vou demonstrar apenas a adição da cadeia da CA, pois é o método que mais se diferencia:


```
cp ldap.example.com-cert.pem /usr/local/share/ca-certificates/ldap.example.com-cert.crt
update-ca-certificates -v
```

Verifique se o Debian consegue encontrar os usuários e grupos através do comando **getent**:

```
getent passwd
getent group
```

Tente se logar com o usuário:

```
# Local
su - hector.vido
# SSH - liberar "PasswordAuthentication yes" em /etc/ssh/sshd_config
ssh hector.vido@127.0.0.1
```

## Centos

Instale os pacotes necessários para a configuração do SSSD:

```
yum install -y sssd vim openldap-clients
```

Caso não utilize DNS:

```
echo '27.11.90.10 ldap.example.com' >> /etc/hosts
```

Verifique se no arquivo **/etc/nsswitch.conf** ao menos passwd, group e shadow utilizem **sss**:

```
passwd: compat sss
group:  compat sss
shadow: compat sss
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

[domain/ldap] 
chpass_provider = ldap 
auth_provider = ldap 
ldap_schema = rfc2307 
id_provider = ldap 
enumerate = true 
cache_credentials = true
# Necessário para autoassinados
ldap_tls_reqcert = never
# Opcional para limitar a base de busca
ldap_user_search_base = ou=users,dc=ldap,dc=example,dc=com

ldap_uri = ldap://ldap.example.com
```

Garanta que o arquivo seja acessado apenas pelo usuário root e reinicie o serviço:

```
chmod 0600 /etc/sssd/sssd.conf
systemctl restart sssd
```

Configure o PAM para utilizar SSS e criar a home dos usuários que se logarem:

```
authconfig --enablesssd --enablesssdauth --enablelocauthorize --enablemkhomedir --update
```

Caso seja utilizado um certificado auto-assinado você tem as mesmas opções do openSUSE, mas neste caso vou demonstrar apenas a adição da cadeia da CA, pois é o método que mais se diferencia:


```
cp ldap.example.com-cert.pem /usr/share/pki/ca-trust-source/anchors/ldap.example.com-cert.pem
update-ca-trust -v
```

Verifique se o CentOS consegue encontrar os usuários e grupos através do comando **getent**:

```
getent passwd
getent group
```

Tente se logar com o usuário:

```
# Local
su - hector.vido
# SSH - liberar "PasswordAuthentication yes" em /etc/ssh/sshd_config
ssh hector.vido@127.0.0.1
```
