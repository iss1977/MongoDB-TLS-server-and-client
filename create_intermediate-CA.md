# Creating _intermediate CA_

An intermediate certificate authority (CA) is an entity that can sign certificates on behalf of the root CA. The root CA signs the intermediate certificate, forming a chain of trust.
The purpose of using an intermediate CA is primarily for security

### 1. Prepare the directory

The intermediate CA files will be kept in a different directory (`/root/ca/intermediate`) 

Create the same directory structure used for the root CA files. It’s convenient to also create a csr directory to hold certificate signing requests.

```bash
mkdir /root/ca/intermediate
cd /root/ca/intermediate
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
openssl rand -hex 16 > serial # Generate a 16 char long random hexadecimal number echo 1000 > serial
```

Add a `crlnumber` file to the intermediate CA directory tree. `crlnumber` is used to keep track of certificate revocation lists.
```bash
openssl rand -hex 16 > /root/ca/intermediate/crlnumber
```

Copy the intermediate CA configuration file from the Appendix to `/root/ca/intermediate/intermediate-CA-openssl.conf`. Five options have been changed compared to the root CA configuration file:

```conf
[ CA_default ]
dir             = /root/ca/intermediate
private_key     = $dir/private/intermediate.key.pem
certificate     = $dir/certs/intermediate.cert.pem
crl             = $dir/crl/intermediate.crl.pem
policy          = policy_loose
```

### 2. Create the intermediate key

```bash
cd /root/ca
openssl genrsa  -out intermediate/private/intermediate.key.pem 4096

chmod 400 intermediate/private/intermediate.key.pem
```

### 3. Create the intermediate certificate

#### 3.1 Create a certificate signing request (CSR)

Use the intermediate key to create a certificate signing request (CSR). The details should generally match the root CA. 
The __Common Name__, however, __must be different.__

> [!WARNING]  
> To create the CSR (certificate signing request) use the intermediate CA configuration file `/root/ca/intermediate/intermediate-CA-openssl.conf`

```bash
cd /root/ca
openssl req -config intermediate/intermediate-CA-openssl.conf -new -sha256 \
      -key intermediate/private/intermediate.key.pem \
      -out intermediate/csr/intermediate.csr.pem
```

#### 3.2 Sign the intermediate certificate signing request (CSR).

To create an intermediate certificate, use the root CA with the `v3_intermediate_ca` extension to sign the intermediate CSR


> [!WARNING]  
> specify the root CA configuration file `/root/ca/root-CA-openssl.conf`

```bash
cd /root/ca
openssl ca -config root-CA-openssl.conf -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 \
      -in intermediate/csr/intermediate.csr.pem \
      -out intermediate/certs/intermediate.cert.pem


chmod 444 intermediate/certs/intermediate.cert.pem
```

#### 3.3 Verify the intermediate certificate

Check that the details of the intermediate certificate are correct.

```bash
openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
```


Verify the intermediate certificate against the root certificate
```bash
openssl verify -CAfile certs/ca.cert.pem intermediate/certs/intermediate.cert.pem

# intermediate.cert.pem: OK
```

### 4. Create the certificate chain file
When an application (eg, a web browser) tries to verify a certificate signed by the intermediate CA, it must also verify the intermediate certificate against the root certificate. To complete the chain of trust, create a CA certificate chain to present to the application.

To create the CA certificate chain, concatenate the intermediate and root certificates together. We will use this file later to verify certificates signed by the intermediate CA.

```bash
cat intermediate/certs/intermediate.cert.pem \
      certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

<br/>

> [!NOTE]  
> Our certificate chain file must include the root certificate because no client application knows about it yet. A better option is to install your root certificate on every client that needs to connect. In that case, the chain file need only contain your intermediate certificate.