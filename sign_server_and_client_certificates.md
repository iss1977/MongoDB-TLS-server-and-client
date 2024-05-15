# Sign server and client certificates

We will be signing certificates using our intermediate CA. You can use these signed certificates in a variety of situations, such as to secure connections to a web server or to authenticate clients connecting to a service.

We will use the server certificate to secure a standalone MongoDB server and the client certificate to authenticate a user in the MongoDB database.

## Create MongoDB server certificate

### 1. Create a private key for mongo server certificate 

Server and client certificates normally expire after one year, so we can safely use 2048 bits instead.

```bash
cd /root/ca
mkdir server-certs
openssl genrsa -out server-certs/mongo-server.key.pem 2048
chmod 400 server-certs/mongo-server.key.pem 
```

### 2. Create a server certificate

#### 2.1 Create a certificate signing request (CSR).

For MongoDB servers, the `Common Name (CN)` will be only used, when the `subjectAltName` is not present.
The Mongo clients will check if the host name ( or IP ) from the connection string is present in the `subjectAltName` list, if not, the connection will be generally refused. ( For exceptions see MongoDB documentation. )

`subjectAltName` will be added in the signing request using the `client-or-server-certificate.conf` configuration file.

Create the [`server-certificate-csr.conf` ](./server-certificate-csr.conf) file inside the `/root/ca/server-certs` folder. Modify the `[alt_names]` according to your needs. This section will be the `subjectAltName` extension in the server certificate.

> [!NOTE]  
> As default, the CA will not add the extensions to the certificate unless `copy_extensions = copy` is added to the `intermediate-CA-openssl.conf` configuration file


Use the private key to create a certificate signing request (CSR):

```bash
cd /root/ca
openssl req -config server-certs/server-certificate-csr.conf \
    -key server-certs/mongo-server.key.pem -new \
    -out server-certs/mongo-server.csr
      
```

#### 2.2 Verify certificate signing request (CSR).

Make sure that `subjectAltName` is present

```bash
openssl req -in server-certs/mongo-server.csr -noout -text

Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = AT, ST = Austria, O = Your-Company, OU = MongoDB, CN = MongoDB-server
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e8:c7:68:23:65:6a:73:ae:83:b8:88:72:8e:6c:
                    ...
                    e8:57
                Exponent: 65537 (0x10001)
        Attributes:
            Requested Extensions:
                X509v3 Subject Alternative Name:
                    DNS:example.com, DNS:localhost, DNS:builder, IP Address:141.144.231.71, IP Address:10.0.0.164, IP Address:127.0.0.1
```

#### 2.3 Sign the server certificate

To create a certificate, use the intermediate CA to sign the CSR. If the certificate is going to be used on a server, use the `server_cert` extension. If the certificate is going to be used for user authentication, we will use the `usr_cert` extension.


We’ll apply the `server_cert` extension from the `intermediate-CA-openssl.conf` file when signing the mongo server certificates, such as those used for web servers.

```conf
[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```

```bash
cd /root/ca
openssl ca -config intermediate-CA/intermediate-CA-openssl.conf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in server-certs/mongo-server.csr \
      -out server-certs/mongo-server.crt

chmod 444 server-certs/mongo-server.crt
```

#### 2.4 Verify the server certificate.

```bash
openssl x509 -in server-certs/mongo-server.crt -noout -text
```

X509v3 Subject Alternative Name must be present.


## Create MongoDB client certificate

### 1. Create a private key for mongo client certificate 

Server and client certificates normally expire after one year, so we can safely use 2048 bits instead.

```bash
cd /root/ca
mkdir client-certs
openssl genrsa -out client-certs/mongo-client.key.pem 2048
chmod 400 client-certs/mongo-client.key.pem 
```

### 2. Create a client certificate

#### 2.1 Create a certificate signing request (CSR).

Client certificates must contain the following fields:

```conf
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
```
This will be stored in the `[ usr_cert ]` section of the `intermediate-CA-openssl.conf` file.

At least one of the following client certificate attributes __must be different__ than the attributes of the server certificates:
```conf
Organization (O)
Organizational Unit (OU)
Domain Component (DC)
```

Use the private key to create a certificate signing request (CSR):

```bash
cd /root/ca
openssl req -config client-certs/client-certificate-csr.conf -key client-certs/mongo-client.key.pem -new -out client-certs/mongo-client.csr 
      
```

#### 2.2 Verify certificate signing request (CSR).

```bash
openssl req -in client-certs/mongo-client.csr -noout -text

Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C = AT, ST = Austria, O = Your-Company, OU = MongoDB, CN = MongoDB-server
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e8:c7:68:23:65:6a:73:ae:83:b8:88:72:8e:6c:
                    ...
                    e8:57
                Exponent: 65537 (0x10001)
        Attributes:
            Requested Extensions:
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
```

#### 2.3 Sign the server certificate

To create a certificate, use the intermediate CA to sign the CSR. If the certificate is going to be used on a server, use the `server_cert` extension. If the certificate is going to be used for user authentication, use the `usr_cert` extension.

We’ll apply the `usr_cert` extension when signing __client certificates__  from the `intermediate-CA-openssl.conf` file.
```conf
[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection
```


```bash
cd /root/ca
openssl ca -config intermediate-CA/intermediate-CA-openssl.conf \
      -extensions usr_cert -days 375 -notext -md sha256\
      -in client-certs/mongo-client.csr -out client-certs/mongo-client.crt

chmod 444 client-certs/mongo-client.crt
```

#### 2.4 Verify the server certificate.

```bash
openssl x509 -in server-certs/mongo-server.crt -noout -text
```
