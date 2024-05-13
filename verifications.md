# Verifying that a Private Key Matches a Certificate

The `modulus' and the `public exponent' portions in the key and the Certificate must match

openssl x509 -noout -modulus -in server.crt 
openssl rsa -noout -modulus -in server.key 


# Verifying that a Certificate is issued by a CA

openssl verify -verbose -CAfile cacert.pem  server.crt
server.crt: OK



https://www.allanbank.com/blog/security/tls/x.509/2014/10/13/tls-x509-and-mongodb/
