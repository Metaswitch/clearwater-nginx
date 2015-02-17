# Managing Nginx HTTPS Certificates

We use Nginx to help provide secure HTTP communication between clients and servers. Nginx achieves this through HTTPS (HTTP over TLS/SSL). We have two requirements from HTTPS. We require a client to be able to verify the identity of a server, and we then require the client and the server to be able to communicate securely. We achieve the first of these two requirements through HTTPS certificates.

In this document we'll refer to the following terms.
* Public key - a public cryptographic key.
* Private key - the corresponding private cryptographic key.
* Certificate signing request (CSR) - a public key with a service provider's information embedded in it.
* Certificate Authority (CA) - a body that signs a CSR with the private key of a known certificate.
* Certificate - a signed CSR.
* Certificate chain - a chain of certificates, where each certificate has been signed by the next certificate in the chain.

In order for the client to verify the identity of the server, the server sends a signed certificate to the client, who trusts the signature. The client may also request the certificate chain which was used to sign the certificate. Once the client has verified the certificate, it can use it to securely send a means of encryption for subsequent messages to the server. The server can read this using the private key, and the server and client can now exchange information securely. There are three options, outlined below, for generating the trusted certificate.

## Self-signed certificate

This requires the client accepting un-trusted certificates, and is the default option for the clearwater-nginx package. It involves the server signing the CSR with its own private key. The post-install script for clearwater-nginx handles this using the Openssl library.

    openssl req -nodes -sha256 -newkey rsa:2048 -keyout nginx.key -out nginx.csr -config nginx_openssl_config
    openssl x509 -sha256 -req -in nginx.csr -signkey nginx.key -out nginx.crt

The first command creates a CSR (using a very basic default set of config) and a corresponding private key. The second command creates a certificate by signing the CSR with its own private key. Clients won't be able to verify the self-signed certificate, and so will have to accept un-trusted certificates. This is not completely secure, since whilst it means the client and server can form a secure communication, the client cannot be sure of the identity of the server.

## Certificate signed by the service provider as a CA

This involves the service provider acting as a CA and signing the CSR with the private key of a certificate they own. This certificate needs to be uploaded to all clients that need to contact the Nginx server. It also needs to be uploaded securely to these clients since it is a critical part of the security chain. The clients will then be able to verify the server's certificate. This is the recommended option, but it does require access to the clients.

The service provider can use Openssl to create a self-signed certificate and private key as in the first option above. The config file should have the format of [nginx_openssl_config](https://github.com/Metaswitch/clearwater-nginx/blob/master/clearwater-nginx/etc/nginx/ssl/nginx_openssl_config), with the default fields replaced. They can then run both commands again on their server, but rather than self-signing the new CSR they should use their first private key.

For example, a service provider (e.g. Clearwater) would create their own certificate on a private box.
    openssl req -nodes -sha256 -newkey rsa:2048 -keyout clearwater.key -out clearwater.csr -config clearwater_openssl_config
    openssl x509 -sha256 -req -in clearwater.csr -signkey clearwater.key -out clearwater.crt

They would then embed clearwater.crt into their clients, and upload clearwater.key onto their server in order to sign their nginx certificate.
    openssl req -nodes -sha256 -newkey rsa:2048 -keyout nginx.key -out nginx.csr -config clearwater_openssl_config
    openssl x509 -sha256 -req -in nginx.csr -signkey clearwater.key -out nginx.crt

The clients would now be able to verify nginx.crt.

There are tools other than Openssl that can be used for generating signed public keys. However, if you do choose to use your own method, note that the first bullet of section 5.3 of the [3GPP specification TS 33.222](http://www.3gpp.org/DynaReport/33222.htm) requires that the client should check that the FQDN in the certificate is the same as the FQDN used to set up the HTTPS connection.

## Certificate signed by a well known CA

This involves paying an existing trusted CA to sign your certificates. The steps required vary from CA to CA, but in general you will need to produce a CSR (see above) which you will send, along with proof of identity to your CA who will send you back a signed certificate along with their certificate chain. The certificate chain will lead back to a certificate that the clients already trust, and so they will be able to verify the certificate. This can be a timely and costly procedure. Popular CAs include Symantec, Comodo SSL and GlobalSign.
