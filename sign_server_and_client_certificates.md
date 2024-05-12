# Sign server and client certificates

We will be signing certificates using our intermediate CA. You can use these signed certificates in a variety of situations, such as to secure connections to a web server or to authenticate clients connecting to a service.

We will use the server certificate to secure a standalone MongoDB server and the client certificate to authenticate a user in the MongoDB database.

### 1. Create a key

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

`subjectAltName` will be added in the signing request using the `` configuration file.


> [!NOTE]  
> As default, the CA will not add the extensions to the certificate unless `copy_extensions = copy` is added to the configuration file


Use the private key to create a certificate signing request (CSR):

```bash
cd /root/ca
openssl req -config server-certs/server-certificate.conf -key server-certs/mongo-server.key.pem -new -out server-certs/mongo-server.csr 
      
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

To create a certificate, use the intermediate CA to sign the CSR. If the certificate is going to be used on a server, use the `server_cert` extension. If the certificate is going to be used for user authentication, use the `usr_cert` extension.


cd /root/ca
openssl ca -config intermediate/intermediate-CA-openssl.conf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in server-certs/mongo-server.csr \
      -out server-certs/mongo-server.crt

chmod 444 intermediate/certs/www.example.com.cert.pem

#### 2.4 Verify the server certificate.

```bash
openssl x509 -in server-certs/mongo-server.crt -noout -text
```

X509v3 Subject Alternative Name must be present.
