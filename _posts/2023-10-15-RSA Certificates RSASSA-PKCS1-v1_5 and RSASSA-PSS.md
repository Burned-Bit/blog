---
title: RSA Certificates (RSASSA-PKCS1-v1_5 and RSASSA-PSS)
published: true
---

In this post we will see how to create RSA certificates using OpenSSL

But first 2 good videos, one about Public Key Cryptography and another about X.509 certificates

https://www.youtube.com/watch?v=GSIDS_lvRv4

https://www.youtube.com/watch?v=kAaIYRJoJkc


## [](#header-2) Understanding RSA and Signature Schemes
RSA is a widely used asymmetric encryption algorithm that involves generating a pair of keys: a public key for encryption and a private key for decryption. Additionally, RSA can be used for digital signatures, where the private key signs the data and the public key verifies the signature.

Signature schemes define the algorithms and processes used to generate and verify signatures. RSASSA-PKCS1-v1_5 and RSASSA-PSS are two common signature schemes used with RSA. RSASSA-PKCS1-v1_5 is based on the PKCS#1 v1.5 standard, while RSASSA-PSS is based on the Probabilistic Signature Scheme (PSS).


### [](#header-3)Generating an RSASSA-PKCS1-v1_5 certificate

```bash
openssl genrsa -out private_key.pem [-aes256] 3072
openssl req -new -key private_key.pem -out csr.pem
openssl x509 -req -in csr.pem -signkey private_key.pem -out certificate.pem

```
The [-aes256] command serves to protect the private key with a passphrase and is optional. 

We can then verify that the generated certificate is a RSASSA-PKCS1-v1_5 cert with the following command:

```bash
 openssl x509 -in certificate.pem -noout -text
```

In the output we will see the following relevant points

![image](https://github.com/Burned-Bit/blog/assets/93063449/2a2fb55a-f8a9-4bd3-a930-47fc881d4ff8)

![image](https://github.com/Burned-Bit/blog/assets/93063449/9e3fd3ff-011d-442a-8797-c9f799d48727)

### [](#header-3)Generating an RSASSA-PSS certificate

To generate an  certificate we just need to change the one command from the previous cert creation.


```bash
openssl genpkey -algorithm rsa-pss -out private_key.pem [-aes256] -pkeyopt rsa_keygen_bits:3072
```


Again if we verify it with Openssl we will see that a RSASSA-PSS certificate has been generated this time:

```bash
 openssl x509 -in certificate.pem -noout -text
```

![image](https://github.com/Burned-Bit/blog/assets/93063449/e60e2e68-b901-409d-9b1f-073302d06ac3)

![image](https://github.com/Burned-Bit/blog/assets/93063449/be6b3f26-5d53-41bb-8849-0afe8e1ba086)




## [](#header-2)Automating the creation of certificates

We can make a bash script to automate this process. Here is a very raw script to do it which ask if you want to use a password for the private key, and also ask for the Common Name (CN) for the certificate. Feel free to improve it.

```bash
#!/bin/bash

generate_private_key() {
    local algorithm="$1"
    local key_file="$2"
    local aes_flag="$3"
    
    echo "Generating a private key for RSA algorithm $algorithm..."
    if [ "$algorithm" == "1" ]; then
        openssl genrsa -out "$key_file" "$aes_flag" 3072
    elif [ "$algorithm" == "2" ]; then
        openssl genpkey -algorithm rsa-pss -out "$key_file" "$aes_flag" -pkeyopt rsa_keygen_bits:3072
    else
        echo "Invalid choice. Exiting..."
        exit 1
    fi
}

generate_certificate() {
    local key_file="$1"
    local csr_file="$2"
    local cert_file="$3"
    
    # Create a CSR
    openssl req -new -key "$key_file" -subj "/CN=$common_name" -out "$csr_file"
    
    # Sign the CSR and create the certificate
    openssl x509 -req -in "$csr_file" -signkey "$key_file" -out "$cert_file"
}

echo "Select the RSA algorithm:"
echo "1. RSASSA-PKCS1-v1_5"
echo "2. RSA-PSS"
read -p "Enter your choice (1 or 2): " choice

# Prompt for AES-256 encryption for private key
read -p "Do you want to use AES-256 encryption for the private key? (y/n): " aes_choice

if [ "$aes_choice" == "y" ]; then
    aes_flag="-aes256"
else
    aes_flag=""
fi

generate_private_key "$choice" "private_key.pem" "$aes_flag"

# Prompt the user for the Common Name (CN)
read -p "Enter the Common Name (CN) for the certificate: " common_name

generate_certificate "private_key.pem" "csr.pem" "certificate.pem"

echo "Certificate created successfully."

# Verify the certificate and display details
echo "Verifying the created certificate..."
openssl x509 -in certificate.pem -noout -text

# Ask if user wants to create a Root-CA certificate and sign the generated certificate
read -p "Do you want to create a Root-CA certificate and sign the generated certificate? (y/n): " ca_choice

if [ "$ca_choice" == "y" ]; then
    generate_private_key "$choice" "ca_private_key.pem" "$aes_flag"
    
    # Create a CSR for the Root-CA
    openssl req -new -key "ca_private_key.pem" -subj "/CN=Root-CA" -out "ca_csr.pem"
    
    # Sign the CSR to create the Root-CA certificate
    openssl x509 -req -in "ca_csr.pem" -signkey "ca_private_key.pem" -out "ca_cert.pem"
    
    # Finally sign the certificate with the CA certificate
    openssl x509 -req -in "csr.pem" -CA "ca_cert.pem" -CAkey "ca_private_key.pem" -out "certificate_signed.pem" -CAcreateserial
    echo "Certificate signed with Root-CA certificate."
fi

echo "Finished!"


```


## [](#header-2)Verifying the certificates with OpenSLL and Wireshsark

We could also check that in 

For that we can quickly set up a local TLS server with the following command:

```bash
openssl s_server --accept 1234 -key private_key.pem  -cert certificate.pem -tls1_2 -debug

```

Then we can fire up Wireshark on the any or loopback addresss
and connect with the OpenSSL client:

```bash
openssl s_client -connect 127.0.0.1:1234

```
This is the output from the TLS client where we can see the RSA certificate being used

![image](https://github.com/Burned-Bit/blog/assets/93063449/e691c1a6-382a-416b-8bd7-d528037d3ab0)

![image](https://github.com/Burned-Bit/blog/assets/93063449/4799dd6b-e1c0-4657-8776-e86d8f7d70d1)

And this is the Wireshark Server Hello packet where we can also see the info

![image](https://github.com/Burned-Bit/blog/assets/93063449/85ebf28b-461a-4831-8f83-8d90c94627bb)


WIP


