# Mongo with TLS/SSL encryption and X.509 Certificates to Authenticate Clients (Ubuntu 22.04)

Objectives :

- install MongoDB & Nginx
- create a root certification authority ( root CA )
- create an intermediate certification authority to sign MongoDB standalone server and MongoDB client certificates. 
- Connect to MongoDB Instances Using TLS Encryption.
- Connect to MongoDB Instances that Require Client Certificates.
- proxy MongoDB connection with Nginx.

## 1. Install MongoDB and nginx

To Install MongoDB, the steps in the documentation:
https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/

for nginx,
https://www.nginx.com/resources/wiki/start/topics/tutorials/install/

To test functionality from a remote, you can install only `mongosh`

```bash
sudo apt-get install -y mongodb-org-shell
```

### Verify that MongoDB is running and identify the configuration file.

```bash
ps aux | grep mongo
mongodb  1368919  2.5 14.1 2639876 137824 ?      Ssl  19:14   0:02 /usr/bin/mongod --config /etc/mongod.conf
```
## 2. Create root-CA and intermediate-CA

Root CA will be used to sign an intermediate CA, which will sign the server and client certificates.

 - [Creating __Root certificate authority__ ](./create_root-CA.md)
 - [Creating __intermediate CA__](./create_intermediate-CA.md)


## 3. Create a Mongo server certificate and a Mongo client certificate

Sign server and client certificates with the intermediate CA.

 - [Creating __server and client certificates__ ](./sign_server_and_client_certificates.md)


## 4. Prepare private keys and certificates to be used by MongoDB


### 4.1 Prepare private keys and certificates to be used by MongoDB

Concatenate client private key and client certificate. This file will be used for client authentication.

```bash
cat mongo-client.key.pem mongo-client.crt > mongo-client-bundle.pem
cat mongo-client-bundle.pem

-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

For MongoDB server we will need:
  - the CA bundle, containing root CA and intermediate CA `mongo-server-bundle.pem`(will be created by concatenating root CA and intermediate CA)
  - a server certificate, signed by the intermediate CA `ca-chain.pem` ( will be created by concatenating the server private key and the server certificate)

```bash
cat mongo-server.key.pem mongo-server.crt > mongo-server-bundle.pem
cat mongo-server-bundle.pem

-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

```bash
cat root-CA/certs/ca.cert.pem intermediate-CA/certs/intermediate-CA.cert.pem > ca-chain.pem
cat ca-chain.pem

-----BEGIN CERTIFICATE-----
... 
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

Move the file `mongo-server-bundle.pem` and `ca-chain.pem` to a directory accessible by the process `mongodb` where the MongoDB server is running.

Move the file `mongo-client-bundle.pem` to the machine where the MongoDB client ( here `mongosh`) is located.

### 4.2 Create user in MongoDB

To enable X.509 authentication in MongoDB, we will need to create a user from the `subject` in `RFC2253` format present in the client certificate.

Retrieve the `RFC2253` formatted subject from the client certificate:
```bash
openssl x509 -in <pathToClientPEM> -inform PEM -subject -nameopt RFC2253

subject=CN=your-mongo-CN,OU=your-mongo-OU,O=your-organisation # other fields may be present
-----BEGIN CERTIFICATE-----

```

Login to MongoDB server using `mongosh`. This is possible while the server is not protected after installation.

Run `mongosh` from the same machine running MongoDB server. Following warning should be present:
`Access control is not enabled for the database. Read and write access to data and configuration is unrestricted`

```bash
mongosh
```

After successful login, we will create the user for our client certificate:

```js
db.getSiblingDB("$external").runCommand(
  {
    createUser: "CN=your-mongo-CN,OU=your-mongo-OU,O=your-organisation", // change according to your RFC2253 subject
    roles: [
         { role: "readWrite", db: "test" },
         { role: "userAdminAnyDatabase", db: "admin" }
    ],
    writeConcern: { w: "majority" , wtimeout: 5000 }
  }
)
// on success, result will be "{ ok: 1 }"
```


subject=

db.getSiblingDB("$external").runCommand(
  {
    createUser: "DC=Mongo-Client,CN=Mongo-Client,OU=MongoDB,O=iss-dev", // change according to your RFC2253 subject
    roles: [
         { role: "readWrite", db: "test" },
         { role: "userAdminAnyDatabase", db: "admin" }
    ],
    writeConcern: { w: "majority" , wtimeout: 5000 }
  }
)






### 4.3 Secure MongoDB

To secure MongoDB, we will edit the `/etc/mongod.conf` file. Use the files created in step "4.1": 

```conf
net:
  port: 27017
  bindIp: 127.0.0.1
  tls:
    mode: requireTLS
    certificateKeyFile: <your ca-chain.pem file>
    CAFile: <your mongo-server-bundle.pem file>

security:
  authorization: enabled
```

Restart and check `MongoDB` status:

```bash
sudo systemctl restart mongod
sudo systemctl status mongod
```

Check if status is `Active: active (running)`. On error, chech MongoDB logs `/var/log/mongodb/mongod.log`


### 4.4 Connect to MongoDB using `mongosh`

```bash
mongosh --tls --host localhost --tlsCertificateKeyFile < your mongo-client-bundle.pem file > --tlsCAFile < your ca-chain.pem file >
```

You will connect securely with TLS to MongoDB Shell. 

The session is not authenticated, commands will fail:

```js
test> use admin
switched to db admin
admin> db.getUsers()
MongoServerError[Unauthorized]: Command usersInfo requires authentication
admin>
```

To authenticate, run:

```js

db.getSiblingDB("$external").auth(
  {
    mechanism: "MONGODB-X509"
  }
)


// to test if authentication is working:

test> use $external
switched to db $external

$external> db.getUsers()

{
  users: [
    {
      _id: '$external.CN=Mongo-Client,OU=MongoDB,O=iss-dev',
      userId: UUID('56eabf48-6180-40e6-8f48-a5209715b3e6'),
      user: 'CN=Mongo-Client,OU=MongoDB,O=your-company',
      db: '$external',
      roles: [
        { role: 'readWrite', db: 'test' },
        { role: 'userAdminAnyDatabase', db: 'admin' }
      ],
      mechanisms: [ 'external' ]
    },
  ],
  ok: 1
}
```

Or, connecting with `mongosh` and authentication in one command:
```bash
mongosh --tls --tlsCertificateKeyFile < your mongo-client-bundle.pem file > --tlsCAFile < your ca-chain.pem file >
    --authenticationDatabase '$external' \
    --authenticationMechanism MONGODB-X509
```


Connection to MongoDB Instances Using Encryption with SCRAM authentication is also possible.

```js
use admin
db.createUser(
  {
    user: "root",
    pwd: "admin123",
    roles: [
      { role: "root", db: "admin" }
    ]
  }
)

db.getUsers({
    showCredentials: true
})

```




Add 

security:
    authorization: enabled

to mongo.conf


Authenticate:

mongosh --port 27017  --authenticationDatabase \
    "admin" -u "myUserAdmin" -p

or in mongosh

use admin
db.auth("myUserAdmin", passwordPrompt()) // or cleartext password




https://www.prisma.io/dataguide/mongodb
https://jamielinux.com/docs/openssl-certificate-authority/index.html


