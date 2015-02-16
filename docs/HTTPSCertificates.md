# Managing nginx HTTPS Certificates

nginx needs to provide secure HTTP communication between clients and servers. This means that the server needs to be able to verify itself to the client so that the client is confident it's sending messages to the correct place, and that the client can communicate in a secure fashion with this server. nginx achieves this through HTTPS (HTTP over SSL). This involves the server sending a signed certifcate (or public key) to the client, who is able to verify the signature. The client may also request the certificate authority (CA) certificate signing request (CSR) which was used to sign the certificate. Once the client has verified the signed certificate, it can use it to securely send a means of encryption for subsequent messages to the server. The server can read this using the private key, and the server and client can now exchange information securely. There are three options for generating the secure certificate.

## Self-signed certificate

This requires the client accepting un-trusted certificates, and is the default option for the clearwater-nginx package. It involves the server creating its own CSR, which it uses to sign its certificate. Currently the post-install script for clearwater-nginx handles this using the openssl library.

    openssl req -nodes -sha256 -newkey rsa:2048 -keyout nginx.key -out nginx.csr -config nginx_openssl_config
    openssl x509 -sha256 -req -in nginx.csr -signkey nginx.key -out nginx.crt

The first command creates a private key and a CSR from a very basic default set of config. The second command creates a signed certificate using the private key and the CSR. Since this is all done at start of life, clients won't be able to verify the certificate, and so will have to accept un-trusted certificates, which is not secure.

## Certificate signed by the service provider as a CA

This involves generating the CSR before deploying nginx. This CSR then needs to be uploaded to all clients that need to contact the nginx server, and the nginx server itself. It can then be used to create a signed certificate on the nginx server which the clients can verify. This is the recommended option, but it does require being able to upload the certificate to your clients.

The CSR can be generated using openssl, as in the first option above. The config file should have the format of [nginx_openssl_config](https://github.com/Metaswitch/clearwater-nginx/blob/master/clearwater-nginx/etc/nginx/ssl/nginx_openssl_config), with the default fields replaced. The CSR and private key can then be uploaded onto the nginx server and used to generate a signed certificate, again as in the first option above. The CSR then needs to be uploaded to your clients.

There are other means of generating your own CSR. However, note that bullet 1) of section 5.3.0 of the 3GPP specification 33.222 suggests that the client should check that the FQDN in the certificate is the same as the FQDN used to set up the HTTPS connection.

## Certificate signed by a well known CA

This involves paying an existing trusted CA to sign your certificates.
