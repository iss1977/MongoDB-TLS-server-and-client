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








client crt - private key + certificate
ca : rootCA  + intermediate ca







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


