mkdir -p /root/ca/{certs,crl,newcerts,private}
> /root/ca/index.txt
echo 1000 > /root/ca/serial
cp /etc/ssl/openssl.cnf /root/ca/

 - Descomentar a linha 71

```
dir             = /root/ca                  # Where everything is kept
certs           = $dir/certs                # Where the issued certs are kept
crl_dir         = $dir/crl                  # Where the issued crl are kept
database        = $dir/index.txt            # database index file.
#unique_subject = no                        # Set to 'no' to allow creation of
                                            # several certs with same subject.
new_certs_dir   = $dir/newcerts             # default place for new certs.

certificate     = $dir/certs/ca-cert.pem    # The CA certificate
serial          = $dir/serial               # The current serial number
crlnumber       = $dir/crlnumber            # the current crl number
                                            # must be commented out to leave a V1 CRL
crl             = $dir/crl/ca-crl.pem       # The current CRL
private_key     = $dir/private/ca-key.pem   # The private key
```

```
[ v3_intermediate_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical,CA:false,pathlen:0 # indica que esta CA não poderá ter outras CAs abaixo
```

cd /root/ca
openssl genrsa -aes256 -out private/cakey.pem 4096
chmod 400 private/cakey.pem
openssl req -config openssl.cnf -key private/cakey.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out cacert.pem

```
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:São Paulo
Locality Name (eg, city) []:São Paulo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Dragonfly Creative Ltda.
Organizational Unit Name (eg, section) []:Dragonfly Certificate Authority                                                        
Common Name (e.g. server FQDN or YOUR name) []:example.com
Email Address []:
```

openssl x509 -in cacert.pem -noout -text

mkdir -p /root/ca/intermediate/{certs,crl,csr,newcerts,private}
> /root/ca/intermediate/index.txt
echo 1000 > /root/ca/intermediate/serial
echo 1000 > /root/ca/intermediate/crlnumber

```
dir             = /root/ca/intermediate             # Where everything is kept
certs           = $dir/certs                        # Where the issued certs are kept
crl_dir         = $dir/crl                          # Where the issued crl are kept
database        = $dir/index.txt                    # database index file.
#unique_subject = no                                # Set to 'no' to allow creation of
                                                    # several certs with same subject.
new_certs_dir   = $dir/newcerts                     # default place for new certs.

certificate     = $dir/certs/intermediate-cert.pem  # The CA certificate
serial          = $dir/serial                       # The current serial number
crlnumber       = $dir/crlnumber                    # the current crl number
                                                    # must be commented out to leave a V1 CRL
crl             = $dir/crl/intermediate-crl.pem     # The current CRL
private_key     = $dir/private/intermediate-key.pem # The private key
```

cd /root/ca/intermediate
openssl genrsa -aes256 -out private/intermediate-key.pem
chmod 0400 private/intermediate-key.pem

openssl req -config openssl.cnf -new -sha256 -key private/intermediate-key.pem -out csr/intermediate-csr.pem

```
Country Name (2 letter code) [AU]:BR
State or Province Name (full name) [Some-State]:São Paulo
Locality Name (eg, city) []:São Paulo
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Dragonfly Creative Ltda.
Organizational Unit Name (eg, section) []:Dragonfly Certificate Authority
Common Name (e.g. server FQDN or YOUR name) []:intermediate.example.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
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
