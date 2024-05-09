https://rajanmaharjan.medium.com/secure-your-mongodb-connections-ssl-tls-92e2addb3c89



1. creating own certificate authority

1.1 Create CA private key

```
openssl genrsa -out rootCA.key 2048
```

1.2 The next step is to self-sign this certificate.

openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem

Once done, this will create an SSL certificate called rootCA.pem, signed by itself, valid for 1024 days, and it will act as our root certificate

Get certificate info :
openssl x509 -in certificate.crt -text -noout


2. Create A Certificate (Done Once Per Device)

2.1 Private key
openssl genrsa -out mongodb.key 2048

2.2 certificate signing request

openssl req -new -key mongodb.key -out mongodb.csr

Verify 
openssl req -in mongodb.csr -text -noout

3. 

Once that’s done, you’ll sign the CSR, which requires the CA root key.

openssl x509 -req -in mongodb.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out mongodb.crt -days 500 -sha256

3. Create .pem file

cat mongodb.key mongodb.crt > mongodb.pem

Verify 

openssl verify -verbose -CAfile rootCA.pem mongodb.pem
# Response : mongodb.pem: OK



4. Enabling TLS on a self-hosted or self-managed MongoDB server

Edit /etc/mongod.conf

net:
  port:27017
  bindIp: 127.0.0.1
  ssl:                                !!! replace with tls
    mode: requireSSL
    PEMKeyFile: <route-to-cert-file>
    CAFile: <route-to-ca-file>


sudo service mongodb restart


5. Create client certificates

A single Certificate Authority (CA) must issue the certificates for both the client and the server.

Client certificates must contain the following fields:

keyUsage = digitalSignature

extendedKeyUsage = clientAuth

Each unique MongoDB user must have a unique certificate.



5.1 Create signing request

openssl req -new -key myPrivate.key  -out my.csr -config openssl_mongo.cfg


5.2 Sign client certificate request using CA

openssl x509 -req -CA rootCA.pem -CAkey rootCA.key -CAcreateserial  -days 1000 -in  mongo-user.csr -out mongo-user.crt -sha256

5.3 Concatenate certificate with private key

cat myPrivate.key mongo-user.crt > mongo-user.pem

5.4 Get subject from certificate for further use 

openssl x509 -in user.pem -inform PEM -subject -nameopt RFC2253

subject=emailAddress=emailaddress@myemail.com,CN=mycommname.com,OU=MyOrgUnit,O=MyOrg,L=EFG_HIJ,ST=CD,C=AU
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----









[ req ]
prompt                 = no
days                   = 365
distinguished_name     = req_distinguished_name
req_extensions         = v3_req


[ req_distinguished_name ]
countryName            = AB
stateOrProvinceName    = CD
localityName           = EFG_HIJ
organizationName       = MyOrg
organizationalUnitName = MyOrgUnit
commonName             = mycommname.com
emailAddress           = emailaddress@myemail.com


[ v3_req ]
basicConstraints       = CA:false
extendedKeyUsage       = clientAuth, serverAuth
subjectAltName         = @sans

[ sans ]
DNS.0 = localhost





END:
MongoServerSelectionError: Hostname/IP does not match certificate's altnames: Host: localhost. is not cert's CN: iss-dev.xyz

