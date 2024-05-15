# Creating __Root certificate authority__ 

A certificate authority (CA) is an entity that signs digital certificates

Typically, the root CA does not sign server or client certificates directly. The root CA is only ever used to create one or more intermediate CAs, which are trusted by the root CA to sign certificates on their behalf. This is best practice. It allows the root key to be kept offline and unused as much as possible, as any compromise of the root key is disastrous.

### 1. Choose a directory (`/root/ca/root-CA`) to store all keys and certificates.

Switch to root using `sudo su`. Create directory structure and additionally needed files.

```bash
mkdir -p /root/ca/root-CA
cd /root/ca/root-CA
mkdir certs crl newcerts private csr
chmod 700 private
# The index.txt and serial files act as a flat file database to keep track of signed certificates
touch index
openssl rand -hex 16 > serial # Generate a 16 char long random hexadecimal number
openssl rand -hex 16 > crlnumber
```


| Directory | Purpose |
|--|--|
| ca/private | private keys |
| ca/certs | store certificates |
| ca/newcerts | new issued certificates |
| ca/csr | store signing requests |
| ca/clr | certificate revocation list |
| | |


### 2. Prepare the configuration file

Create a configuration file for OpenSSL to use `/root/ca/root-CA/root-CA-openssl.conf`

The [ ca ] section is mandatory. Here we tell OpenSSL to use the options from the [ CA_default ] section.

```conf
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = /root/ca/root-CA
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial

# The root key and root certificate.
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict
```

We’ll apply `policy_strict` for all root CA signatures, as the root CA is only being used to create intermediate CAs

```bash
[ policy_strict ]
# The root CA should only sign intermediate certificates that match of the field in the CA's DN.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

Options from the `[ req ]` section are applied when creating certificates or certificate signing requests.

```bash
[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca
```
<br/>

The `[ req_distinguished_name ]` section declares the information normally required in a certificate signing request. You can optionally specify some defaults.

```bash
[ req_distinguished_name ]

countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Optionally, specify some defaults.
countryName_default             = AT
stateOrProvinceName_default     = Austria
localityName_default            =
organizationName_default        = YourCompany
#emailAddress_default           =
```
<br/>

We’ll apply the `v3_ca` extension when we create the root certificate, by passing the `-extensions v3_ca` command-line argument.
```conf
[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```
<br/>

### 3. Create the root key
```bash
cd /root/ca/root-CA
openssl genrsa -out private/ca.key.pem 4096 # add "-aes256" option to protect your key with a password
chmod 400 private/ca.key.pem

# Result: 
# -r-------- 1 root root 3272 May 12 15:26 ca.key.pem

```

### 4. Create the root certificate

Create a certificate for 7300 days using provided openssl.cnf using the `v3_ca` extensions:

```bash
cd /root/ca/root-CA
openssl req -config root-CA-openssl.conf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -sha256 -extensions v3_ca \
      -out certs/ca.cert.pem
chmod 444 certs/ca.cert.pem # read by anybody
```


### 5. Verify the root certificate
```bash
openssl x509 -noout -text -in certs/ca.cert.pem

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            2d:2e:02:57:ca:a9:39:c8:70:ea:1e:b4:f7:8c:11:cf:1b:b6:cf:dc
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = AT, ST = Austria, L = Linz, O = MongoDB Root, OU = Root certification, CN = root
        Validity
            Not Before: May 12 15:34:23 2024 GMT
            Not After : May  7 15:34:23 2044 GMT
        Subject: C = AT, ST = Austria, L = Linz, O = MongoDB Root, OU = Root certification, CN = root
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:c4:7e:de:52:5d:85:de:11:1e:27:d2:54:92:67:
                    ...
                    b4:36:4d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                98:C4:4B:2B:45:09:7F:4D:6E:9A:91:12:02:FC:49:33:9A:4E:0E:1A
            X509v3 Authority Key Identifier:
                98:C4:4B:2B:45:09:7F:4D:6E:9A:91:12:02:FC:49:33:9A:4E:0E:1A
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        91:dd:a0:81:60:91:9d:59:08:50:8f:6b:25:e5:cf:b2:e9:b7:
        ...
        4f:5b:0b:e2:8e:9f:b3:ce


```



