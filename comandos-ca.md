mkdir -p /root/ca/{certs,crl,newcerts,private}
> /root/ca/index.txt
echo 1000 > /root/ca/serial
cp /etc/ssl/openssl.cnf /root/ca/

Descomentar a linha 71

[ v3_intermediate_ca ]
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical,CA:false,pathlen:0 # indica que esta CA não poderá ter outras CAs abaixo

mkdir -p /root/ca/intermediate/{certs,crl,csr,newcerts,private}
> /root/ca/intermediate/index.txt
echo 1000 > /root/ca/intermediate/serial
echo 1000 > /root/ca/intermediate/crlnumber

vim /etc/ssl/openssl.cnf
