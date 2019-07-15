# CA

Este exemplo tem como objetivo demonstrar a criação de uma Autoridade Certificadora através do **openssl** alterando o mínimo possível de diretivas com a distribuição GNU/Linux **openSUSE**.
Não possuí nenhuma intenção de tornar o acesso a esses arquivos seguro.

Uma CA - Certificate Authority - Autoridade Certificadora - é uma entidade confiável para informar se algum certificado é válido ou não. Quando criamos certificados autoassinados estamos dizendo que nosso certificado é válido, mas uma informação advinda de nós mesmos não tem validade para terceiros. Essa confiança então é delegada a alguma entidade pública, conhecida como CA.

Os computadores ou browsers possúem uma base de CAs - */var/lib/ca-certificates/*, e através dela conseguem saber se um certificado é valido ou não. Para que um certificado autoassinado possa ser considerado válido, podemos adicionar a nossa CA nesta base, e é isso o que faremos aqui.

Para que o exemplo possa ser o mais próximo da estrutura real possível de uma CA pública, utilizarei uma CA raíz, uma CA intermediaria e com ela assinarei o certificado final do usuário.

 - Root CA
   - Intermediate CA
     - Certificado LDAP

### Root CA

A CA raíz é utilizada para gerar as CAs intermediárias. É altamente recomendável que a chave utilizada pela CA raíz não fique nos servidores de produção. Isso garante que a CA raíz não seja corrompida, comprometendo toda a cadeia.

Cria a estrutura básica com os arquivos necessários para começar a assinar outros certificados:

```
mkdir -p /root/ca/{certs,crl,newcerts,private}
> /root/ca/index.txt
echo 1000 > /root/ca/serial
cp /etc/ssl/openssl.cnf /root/ca/
```

Faça algumas pequenas modificação no arquivo **/root/ca/openssl.cnf**:

 - Descomentar a linha 241 - **keyUsage** - especifica para que este certificado será utilizado
 - Comentar linha 58 - **x509_extensions** - não queremos essas diretivas em nossos certificados de CA

 **/root/ca/openssl.cnf**:
```
dir             = /root/ca                  # Where everything is kept
...
certificate     = $dir/certs/ca-cert.pem    # The CA certificate
...
crl             = $dir/crl/ca-crl.pem       # The current CRL
private_key     = $dir/private/ca-key.pem   # The private key
...
policy          = policy_match
...
[ v3_intermediate_ca ]
subjectKeyIdentifier=hash                       # hash segue RFC3280, a outra opção não é aconselhada
keyUsage = cRLSign, keyCertSign                 # Para que o certificado será utilizado
authorityKeyIdentifier=keyid:always,issuer      # keyid - copia o key identifier do certificado pai, 
                                                # issuer se keyid falha copia issuer e serial number do pai
basicConstraints = CA:true,pathlen:0   # indica que será uma CA e não poderá ter outras CAs abaixo
```

Vamos gerar a chave para criptografar o **CSR** - Certificate Sign Request:

```
cd /root/ca
openssl genrsa -aes256 -out private/ca-key.pem 4096
chmod 400 private/ca-key.pem
openssl req -config openssl.cnf -key private/ca-key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca-cert.pem
```

Ao gerar o CSR algumas perguntas são feitas. Neste caso é importante se atentar ao **countryName**, **stateOrProvinceName** e **organizationName**. Devido a política especificada - *policy_match* - estes valores precisarão ser os mesmos no certificado da CA intermediária:

```
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:Sao Paulo
Locality Name (eg, city) []:Sao Paulo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Dragonfly Creative Ltda.
Organizational Unit Name (eg, section) []:Dragonfly Root CA
Common Name (e.g. server FQDN or YOUR name) []:example.com
Email Address []:
```

Vamos verificar o certificado:

```
openssl x509 -in certs/ca-cert.pem -noout -text
```

### Intermediate CA

A CA intermediária é quem assinará os certificados de nosso serviços. Possuí a mesma estrutura da CA raíz:

```
mkdir -p /root/ca/intermediate/{certs,crl,csr,newcerts,private}
> /root/ca/intermediate/index.txtCountry Name (2 letter code) [BR]:
State or Province Name (full name) [Sao Paulo]:
Locality Name (eg, city) [Sao Paulo]:intermediate/certs/intermediate-cert.pem
Organization Name (eg, company) [Dragonfly Creative Ltda.]:
echo 1000 > /root/ca/intermediate/serial
cp /root/ca/openssl.cnf /root/ca/intermediate/
```

Apenas os locais, o nome dos arquivos e a política devem ser alterados:

**/root/ca/intermediate/openssl.cnf**:
```
dir             = /root/ca/intermediate             # Where everything is kept
...
certificate     = $dir/certs/intermediate-cert.pem  # The CA certificate
...
crl             = $dir/crl/intermediate-crl.pem     # The current CRL
private_key     = $dir/private/intermediate-key.pem # The private key
...
policy          = policy_anything
```

Assim como na CA raíz, preciso criar uma chave, e um pedido de assinatura. Mas deste vez, quem assinará será a CA raíz, e não eu mesmo. É possível notar que o comando não possuí o parâmetro **-x509** que é utilizado quando já se deseja assinar o pedido:

```
cd /root/ca/intermediate
openssl genrsa -aes256 -out private/intermediate-key.pem
chmod 0400 private/intermediate-key.pem
openssl req -config openssl.cnf -new -sha256 -key private/intermediate-key.pem -out csr/intermediate-csr.pem

```

Lembre-se de repetir os parâmetros **countryName**, **stateOrProvinceName** e **organizationName** utilizados anteriormente. Isso não será preciso para os certificados assinados pela CA intermediária, já que a política utilizada por ela será **policy_anything**:

```
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:Sao Paulo
Locality Name (eg, city) []:Sao Paulo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Dragonfly Creative Ltda.
Organizational Unit Name (eg, section) []:Dragonfly Intermediate CA
Common Name (e.g. server FQDN or YOUR name) []:intermediate.example.com
Email Address []:
```

Volte para o diretório da CA raíz e assine o certificado. É possível ver nos detalhes do certificado assinado que, diferente do anterior, existe um **issuer**:

```
cd /root/ca
openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 365 -notext -md sha256 -in intermediate/csr/intermediate-csr.pem -out intermediate/certs/intermediate-cert.pem
openssl x509 -in intermediate/certs/intermediate-cert.pem -noout -text
```

Para validar se o certificado faz parte de uma CA, podemos executar o seguinte comando:

```
openssl verify -CAfile certs/ca-cert.pem intermediate/certs/intermediate-cert.pem
intermediate/certs/intermediate-cert.pem: OK
```

Agora que temos mais de uma CA em uma mesma cadeia, precisamos criar um arquivo com essa cadeia, para facilitar as futuras comparações:

```
cat intermediate/certs/intermediate-cert.pem certs/ca-cert.pem > intermediate/certs/ca-chain-cert.pem
chmod 444 intermediate/certs/ca-chain-cert.pem
```

### Criando um certificado

Finalmente vamos criar nosso certificado para o LDAP - poderia ser Apache. Caso fosse um pedido não controlado por nós, poderíamos pular o passo de gerar a chave e gerar o CSR, mas este não é o caso:

```
openssl genrsa -aes256 -out intermediate/private/ldap.example.com-key.pem 2048
chmod 0400 intermediate/private/ldap.example.com-key.pem
openssl req -config intermediate/openssl.cnf -key intermediate/private/ldap.example.com-key.pem -new -sha256 -out intermediate/csr/ldap.example.com-csr.pem
```

Os nomes utilizados aqui não precisam mais coincidir com aqueles utilizados na CA intermediária:

```
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:São Paulo
Locality Name (eg, city) []:São Paulo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Dragonfly Creative Ltda.
Organizational Unit Name (eg, section) []:Dragonfly Lightweight Access Protocol
Common Name (e.g. server FQDN or YOUR name) []:ldap.example.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

Agora, utilizamos o certificado da CA intermediária para assinar este certificado, e verificamos a autenticidade com o arquivo que contém toda a cadeia:

```
openssl ca -config intermediate/openssl.cnf -days 365 -notext -md sha256 -in intermediate/csr/ldap.example.com-csr.pem -out intermediate/certs/ldap.example.com-cert.pem
openssl verify -CAfile intermediate/certs/ca-chain-cert.pem intermediate/certs/ldap.example.com-cert.pem
```

### Copiando a CA

Uma vez que nossa CA esteja funcionando perfeitamente, precisamos copiar o arquivo **intermediate/certs/ldap.example.com-cert.pem** para as máquinas desejadas:

#### openSUSE

```
cp ldap.example.com-cert.pem /usr/share/pki/trust/anchors/
update-ca-certificates -v
```

#### Debian

```
cp ldap.example.com-cert.pem /usr/local/share/ca-certificates/ldap.crt
update-ca-certificates -v
```

#### CentOS

```
cp ldap.example.com-cert.pem /usr/share/pki/ca-trust-source/anchors/ldap.pem
update-ca-trust -v
```
