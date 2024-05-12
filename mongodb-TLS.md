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

We will create our own custom openssl.cnf to add extensions to the Certificate Signing Request

```conf
[req]
# Used by the req command
default_bits            = 2048
distinguished_name      = dn
prompt                  = no
req_extensions          = v3_req

[dn]
# distinguished_name
C="AT"
ST="Linz"
L="Linz"
O="MongoDB"
OU="MongoDB Server"
emailAddress="yourmail@mail.com"
CN="alt-names-will-be-used" #see mongodb documentation

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = iss-dev.us
DNS.2 = localhost
DNS.3 = builder
IP.1 = 141.144.238.71
IP.2 = 10.0.0.164
IP.3 = 127.0.0.1
~                     
```

openssl req -new -key mongodb.key  -out mongo-server.csr -config server-csr-mongodb.conf

Verify that the signing request contains the "subjectAltName"

```bash
$ openssl req -text -in mongo-server.csr -noout | grep -A 2 "Requested Extensions:"

Requested Extensions:
                X509v3 Subject Alternative Name:
                    DNS:iss-dev.us, DNS:localhost, DNS:builder, IP Address:141.144.238.71, IP Address:10.0.0.164, IP Address:127.0.0.1
```

$ openssl x509 -req -in mongo-server.csr -out mongodb.crt -CA ../mongo-rootCA/rootCA.pem -CAkey ../mongo-rootCA/rootCA.key -extfile server-csr-mongodb.conf -extensions v3_req -CAcreateserial -days 1000 -sha256


or


 openssl x509 -req -in mongo-server.csr -out mongodb.crt -CA ../mongo-rootCA/rootCA.pem -CAkey ../mongo-rootCA/rootCA.key server-csr-mongodb.conf -CAcreateserial -days 1000 -sha256  -copy_extensions copyall 


Verify :
$ openssl x509 -text -noout -in mongodb.crt


5.3 Concatenate certificate with private key

cat myPrivate.key mongo-user.crt > mongo-user.pem

$ cat mongo-user.pem

-----BEGIN PRIVATE KEY----
...
-----END PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----




5.2 Sign client certificate request using CA


Client certificates must contain the following fields:

keyUsage = digitalSignature
extendedKeyUsage = clientAuth

Generate private key:

openssl genrsa -out mongo-client.key 2048


$ sudo openssl req -new -key mongo-client.key  -out mongo-user.csr -config mongo-client-ssl-conf.conf


$ sudo openssl x509 -req -CA ../mongo-rootCA/rootCA.pem -CAkey ../mongo-rootCA/rootCA.key -CAcreateserial  -days 1000 -in  mongo-user.csr -out mongo-client.crt -sha256 -extfile mongo-client-ssl-conf.conf -extensions extensions


Verify: 
openssl x509 -text -noout -in mongo-user.crt
Note: must be present

X509v3 extensions:
            X509v3 Key Usage:
                Digital Signature
            X509v3 Extended Key Usage:
                TLS Web Client Authentication



Verify coonection to mongodb

$ mongosh --tls --host localhost --tlsCertificateKeyFile mongo-client.pem --tlsCAFile ../mongo-rootCA/rootCA.pem




5.4 Get subject from certificate for further use 

openssl x509 -in user.pem -inform PEM -subject -nameopt RFC2253

subject=emailAddress=emailaddress@myemail.com,CN=mycommname.com,OU=MyOrgUnit,O=MyOrg,L=EFG_HIJ,ST=CD,C=AU
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----




Create user

db.getSiblingDB("$external").runCommand(
  {
    createUser: "CN=mongo-client-01,emailAddress=mongo-client@mail.com,OU=MongoDB Client,O=MongoDB,L=Linz,ST=Linz,C=AT",
    roles: [
         { role: "readWrite", db: "test" },
         { role: "userAdminAnyDatabase", db: "admin" }
    ],
    writeConcern: { w: "majority" , wtimeout: 5000 }
  }
)



Authenticate:

mongosh --tls --host localhost --tlsCertificateKeyFile mongo-client.pem --tlsCAFile ../mongo-rootCA/rootCA.pem --authenticationDatabase '$external' --authenticationMechanism MONGODB-X509


After connection:

db.getSiblingDB("$external").auth(
  {
    mechanism: "MONGODB-X509"
  }
)


> [!TIP]
> Urgent info that needs immediate user attention to avoid problems.

Appendix

- Install the ca-certificates package. `sudo apt-get install -y ca-certificates`
- Copy the CA.pem file to the /usr/local/share/ca-certificates directory. `sudo cp CA.pem /usr/local/share/ca-certificates/CA.crt`
- Update the certificate store. `sudo update-ca-certificates`




mongosh --tls --host iss-dev.us --port 27020 --tlsCertificateKeyFile ~/mongo-ssl-client/mongo-user.pem --tlsUseSystemCA





END:
MongoServerSelectionError: Hostname/IP does not match certificate's altnames: Host: localhost. is not cert's CN: iss-dev.xyz

https://www.golinuxcloud.com/add-x509-extensions-to-certificate-openssl/

https://github.com/openssl/openssl/blob/baba1545105131fa34068f62928322e99d695ab1/apps/openssl.cnf#L219


https://jamielinux.com/docs/openssl-certificate-authority/index.html