mkdir -p /root/ca/{certs,crl,newcerts,private}
> /root/ca/index.txt
echo 1000 > /root/ca/serial
cp /etc/ssl/openssl.cnf /root/ca/

 - Descomentar a linha 71, 242
 - Comentar linha 58, 107, 109

```
dir             = /root/ca                  # Where everything is kept
...
certificate     = $dir/certs/ca-cert.pem    # The CA certificate
...
crl             = $dir/crl/ca-crl.pem       # The current CRL
private_key     = $dir/private/ca-key.pem   # The private key
...
policy          = policy_match

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = BR
countryName_min                 = 2
countryName_max                 = 2

stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = São Paulo

localityName                    = Locality Name (eg, city)

0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = Dragonfly Creative Ltda.

# we can do this but it is not needed normally :-)
#1.organizationName             = Second Organization Name (eg, company)
#1.organizationName_default     = World Wide Web Pty Ltd

organizationalUnitName          = Organizational Unit Name (eg, section)
#organizationalUnitName_default = Dragonfly Certificate Authority

commonName                      = Common Name (e.g. server FQDN or YOUR name)
commonName_max                  = 64

emailAddress                    = Email Address
emailAddress_max                = 64

# SET-ex3                       = SET extension number 3
```

```
[ v3_intermediate_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical,CA:false,pathlen:0 # indica que esta CA não poderá ter outras CAs abaixo
```

cd /root/ca
openssl genrsa -aes256 -out private/ca-key.pem 4096
chmod 400 private/ca-key.pem
openssl req -config openssl.cnf -key private/ca-key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca-cert.pem

```
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:São Paulo
Locality Name (eg, city) []:São Paulo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Dragonfly Creative Ltda.
Organizational Unit Name (eg, section) []:Dragonfly Certificate Authority                                                        
Common Name (e.g. server FQDN or YOUR name) []:example.com
Email Address []:
```

openssl x509 -in certs/ca-cert.pem -noout -text

mkdir -p /root/ca/intermediate/{certs,crl,csr,newcerts,private}
> /root/ca/intermediate/index.txt
echo 1000 > /root/ca/intermediate/serial
echo 1000 > /root/ca/intermediate/crlnumber
cp /root/ca/openssl.cnf /root/ca/intermediate/

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

cd /root/ca/intermediate
openssl genrsa -aes256 -out private/intermediate-key.pem
chmod 0400 private/intermediate-key.pem

openssl req -config openssl.cnf -new -sha256 -key private/intermediate-key.pem -out csr/intermediate-csr.pem

```
Country Name (2 letter code) [BR]:
State or Province Name (full name) [São Paulo]:
Locality Name (eg, city) [São Paulo]:
Organization Name (eg, company) [Dragonfly Creative Ltda.]:
Organizational Unit Name (eg, section) [Dragonfly Certificate Authority]:Dragonfly Intermediate Certificate Authority                               
Common Name (e.g. server FQDN or YOUR name) []:intermediate.example.com
Email Address []:
```

cd /root/ca
openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 365 -notext -md sha256 -in intermediate/csr/intermediate-csr.pem -out intermediate/certs/intermediate-cert.pem
openssl x509 -in intermediate/certs/intermediate-cert.pem -noout -text

```
opensuse:~/ca # openssl verify -CAfile certs/ca-cert.pem intermediate/certs/intermediate-cert.pem
intermediate/certs/intermediate-cert.pem: OK
```

cat intermediate/certs/intermediate-cert.pem certs/ca-cert.pem > intermediate/certs/ca-chain-cert.pem
chmod 444 intermediate/certs/ca-chain-cert.pem

### Criando um certificado

openssl genrsa -aes256 -out intermediate/private/ldap.example.com-key.pem 2048
chmod 0400 intermediate/private/ldap.example.com-key.pem

openssl req -config intermediate/openssl.cnf -key intermediate/private/ldap.example.com-key.pem -new -sha256 -out intermediate/csr/ldap.example.com-csr.pem

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

openssl ca -config intermediate/openssl.cnf -days 365 -notext -md sha256 -in intermediate/csr/ldap.example.com-csr.pem -out intermediate/certs/ldap.example.com-cert.pem
