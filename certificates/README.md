# Certificates lab

## Setup

For this lab exercise you need [the same setup as the SSH lab](https://github.ugent.be/security/docs/blob/main/setup-vms.md). You can use the snapshots to revert te VMs. Make sure all VMs are still in working condition by running the "[testing out the VMs](https://github.ugent.be/security/docs/blob/main/setup-vms.md#testing-out-the-vms)" steps at the end. If these fail, you might need to recreate the VMs.

## OpenSSL introduction

The OpenSSL Project develops and maintains the OpenSSL software - a robust, commercial-grade, full-featured toolkit for general-purpose cryptography and secure communication. At the core of this project is the OpenSSL C library. It implements a lot of cryptographic operations such as symmetric encryption, public-key encryption, digital signature, hash functions and so on. OpenSSL also implements the Secure Socket Layer (SSL) protocol used for HTTPS and VPNs like OpenVPN.

The `openssl` CLI application is a command-line interface to this library that allows you to perform virtually any cryptographic operation. You can run `openssl help` to see most common commands and supported ciphers for encryption and decryption.

OpenSSL CLI has [extensive reference documentation](https://www.openssl.org/docs/man3.0/man1/). However, make sure to click on the `openssl-<command-name>` links instead of `<command-name>` itself, as the former contain the actual information about the command. If this confuses you, you are not alone! Even Google in its infinite wisdom doesn't understand how this documentation works and refuses to rank anything higher than the old OpenSSL 1.0.0 documentation, which was saner.

> How do you encrypt the file `legalities.txt` using the `aes-256-cbc` cipher? You can choose a password yourself.

```shell
# <fill in answer>
```

> How do you decrypt this file?

```shell
# <fill in answer>
```

## SSL Ciphers

As the name implies, OpenSSL is mainly used for Secure Sockets Layer (SSL) cryptography. For backwards-compatibility reasons, OpenSSL supports a lot of ciphers, but many of them are insecure. Although most distributions removed the most insecure ciphers from the default OpenSSL installation, some of them are still supported because they are still widely used. Nevertheless, the program can help you pick a good cipher using the [`openssl ciphers`](https://www.openssl.org/docs/man3.1/man1/openssl-ciphers.html) command. Run `openssl ciphers -s -stdname` to see all supported ciphers ordered by preference. The first column is the official name of the cipher. The second column is how it's often represented in OpenSSL.

A cipher is a chain of encryption algorithms which are used together for ssl. Specifically, there are four algorithms specified:

* **Key exchange algorithm**. Although SSL/TLS uses asymmetric encryption for authentication, the actual data encryption is done using symmetric encryption. This means that symmetric cryptographic session keys, which are generated and kept by the server and client, encrypt and decrypt information. This, of course, occurs after the initial authentication process is completed using asymmetric public and private keys.
* **Symmetric encryption algorithm**. This is used for the actual encryption of the data.
* **Block cipher mode of operation**. This defines how the encryption algorithm will be used to encrypt subsequent blocks of the data.
* **Hash function**. To protect the integrity of the message, data is converted into a hash value of a particular length. This process, known as “hashing,” is considered a one-way function in cryptography. Even the smallest of change in the data will result in a totally different hash. Such hashing is a very fundamental part of the SSL/TLS data transmission, and it’s done using a hashing algorithm.

Let's take a look at `TLS_AES_256_GCM_SHA384`:

* TLS: Since it's a TLS v1.3 algorithm, it uses Diffie Hellman key exchange, so the key exchange is not shown explicitly.
* AES_256: It uses the AES encryption algorithm with 256 bits.
* GCM: It uses Galois/counter mode (GCM)
* SHA384: It uses the SHA384 hash function

> Take a look at the least recommended option. What algorithms are used? Why is this cipher not recommended anymore? Try to find this out using the theory course materials.

```txt
<fill in answer>
```

> Search for this cipher on the internet to find the full explanation of why it's insecure. You can use a website such as [CipherSuite](https://ciphersuite.info) to have concrete information about every cipher's security.

Since cryptography is ever-evolving and vulnerabilities to existing ciphers are constantly found, it's important to keep checking the security of the ciphers you use and phase out vulnerable ciphers. Although distributions such as Ubuntu often disable the oldest and most broken ciphers, they often still support some insecure ciphers for backwards compatibility reasons.

## Checking SSL Certificates

SSL does more than simply encription. It also uses X.509 certificates to authenticate peers. Server authentication is mandatory in HTTPS and client authentication is optional and rarely used.

You can use `opensssl` to verify the certificate of a server and show information about it. For example

```shell
echo | openssl s_client -connect redhat.com:443 -brief
```

* `echo` gives a response to the server, so that the connection is released
* `-connect` specifies the server and port to connect to.
* `-brief` shows short information about the certificate.

> Is this server using a secure ciphersuite?

```txt
<fill in answer>
```

[BadSSL](https://badssl.com/) contains a collection of broken or insecure SSL certificates and configurations.

> Take a look at the following servers and explain what is wrong with them. Specifically show the corresponding lines in the output that show the issues.
>
> * `https://tls-v1-0.badssl.com:1010/`
> * `https://expired.badssl.com/`
> * `https://untrusted-root.badssl.com/`
> * `https://self-signed.badssl.com/`
>

```txt
<fill in answer>
```

You can also use `openssl` to download the SSL certificate of a server, for example of Google.

```shell
echo | openssl s_client -connect google.com:443 -servername google.com | openssl x509 > /tmp/google.com.cert
```

* `-servername` is used to select the correct certificate when multiple are presented, in the case of SNI.
* `openssl x509` extracts the server certificate from the output. It removes information about the certificate chain and connection details. This is the preferred format to import the certificate into other keystores. Remove this part to see more information about the chain.

> Take a look at the certificate with the entire chain present. What company provides the root of trust of the google.com certificate?

```txt
<fill in answer>
```

## Checking certificate extensions

X509 extensions allow for additional fields to be added to a certificate. One of the most common is the subject alternative name (SAN). The SAN of a certificate allows multiple values (e.g., multiple FQDNs) to be associated with a single certificate. The SAN is even used when there aren’t multiple values because the use of a certificate’s common name for verification is deprecated. You can show an extension of a certificate using `openssl x509 -noout -ext <extensionName>`

* `-noout` instructs the command to not output the entire certificate after the output. Without this flag, the command will show the extensions and then output the entire certificate.

Piping output between multiple OpenSSL commands makes it easy to inspect specific certificate extensions. However, don't forget to redirect stderr away if you pipe from `openssl s_client`.

> Download the certificate of `redhat.com` and show the contents of the `subjectAltName` extension using a one-liner.

```shell
<fill in answer>
```

Another common set of extensions include the basic constraints and key usage of a certificate. Specifically, you might want to check if a certificate is allowed to be used as a certificate authority. Again, this can be done in the same way that you can check for a SAN: `openssl x509 -ext basicConstraints,keyUsage`

> Show the contents of `basicConstraints` and `keyUsage` extensions from the Google certificate and the GlobalSign Root CA, which you can find in `/usr/share/ca-certificates/mozilla/GlobalSign_Root_CA.crt`.

```shell
<fill in answer>
```

> What does the `critical` stand for? You can use the [Certificate Extension Reference](https://access.redhat.com/documentation/en-us/red_hat_certificate_system/9/html/administration_guide/standard_x.509_v3_certificate_extensions) to get a lot of information about X.509 extensions.

```shell
<fill in answer>
```

Finally, if you want to see the entire contents of a certificate, you can use `openssl x509 -text`

> Show the entire contents of the Google certificate.

```shell
<fill in answer>
```

## Testing cipher support of servers

If you bring a webserver with HTTPS support to production it's highly recommended to use an SSL server test such as [Qualis SSL Labs](https://www.ssllabs.com/ssltest/). This provides you with a full report on the security of your TLS configuration. It will alert you when the server uses insecure cipher suites or configuration.

If a server is not publicly accessible, you can still manually test configuration parameters such as supported cypher suites. You can do this by specifying the cypher suite to the `s_client` command using `-cipher`

As an example, attempting to use the weak TLS_PSK_WITH_AES_128_CBC_SHA suite against a server that doesn’t support it will result in an error:

```shell
$ echo | openssl s_client -connect redhat.com:443 -cipher ECDHE-RSA-AES128-SHA -no_tls1_3 -brief
40177367437F0000:error:0A000410:SSL routines:ssl3_read_bytes:sslv3 alert handshake failure:../ssl/record/rec_layer_s3.c:1584:SSL alert number 40
```

However, it will work for servers that are less concerned about the security of their users.

```shell
$ echo | openssl s_client -connect argenta.be:443 -cipher ECDHE-RSA-AES128-SHA -no_tls1_3 
CONNECTED(00000003)
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root G2
verify return:1
depth=1 C = US, O = DigiCert Inc, CN = DigiCert EV RSA CA G2
verify return:1
depth=0 jurisdictionC = BE, businessCategory = Private Organization, serialNumber = 0404.453.574, C = BE, L = Antwerp, O = Argenta Spaarbank nv, CN = argenta.be
verify return:1
```

Similarly, you can specify the version of the TLS protocol used in the connection with `-tls1_2`, for example.

> How can you check if `redhat.com` supports older tls1_2 ciphers?

```shell
<fill in answer>
```

> Now try this with the server at `tls13.1d.pw:443`

```shell
<fill in answer>
```

Note that, for these tests to work, your operating system must still support the SSL/TLS versions you are testing for. Otherwise, the connection will fail client-side.

To get a full list of all ciphers supported by a domain, you can use the tool `nmap`.

First, install it using `sudo apt install nmap`. Then execute the following command to get a list of supported ciphers.

```shell
nmap --script ssl-enum-ciphers -p 443 redhat.com
```

`nmap` gets this list by trying every single cipher that is known with the server. Note that `nmap` is an advanced penetration testing tool. Be careful with using it on websites you don't own. The `ssl-enum-ciphers` script, however, is harmless.

> What ciphers does `rc4-md5.badssl.com` support? Are all of these safe?

```shell
<fill in answer>
```

Note: the `nmap` version on Ubuntu 22.04 does not yet support TLS 1.3.

## Certificate formats

* The **PEM** format (originally "Privacy Enhanced Mail") is the most common format used for certificates. Extensions used for PEM certificates are `.cer`, `.crt`, and `.pem`. These files contain a line `-----BEGIN CERTIFICATE-----`, followed by Base64-encoded certificate and end with `-----END CERTIFICATE-----`. Most tools ignore the file contents before the start and after the end of the certificate.
* The **DER** format is the binary form of the certificate and uses the `.der` format.
* The **PKCS#7** or **P7B** format is stored in Base64 ASCII format and has a file extension of `.p7b` or `.p7c`. A P7B file only contains certificates and chain certificates (Intermediate CAs), not the private key. The most common platforms that support P7B files are Microsoft Windows and Java Tomcat.
* The **PKCS#12** or **PFX** format is a binary format for storing the server certificate, intermediate certificates, and the private key in one encryptable file. PFX files usually have extensions such as `.pfx` and `.p12`. PFX files are typically used on Windows machines to import and export certificates and private keys.

You can use `openssl` to switch between formats.

```shell
# Convert PEM to DER
<fill in answer>
# Convert DER to PEM
<fill in answer>
# Convert PEM to P7B
<fill in answer>
# Convert P7B to PEM
<fill in answer>
# Convert pfx to PEM
<fill in answer>
```

See [How to convert a certificate into the appropriate format](https://knowledge.digicert.com/solution/SO26449.html) for more information on how to switch between formats.

## Configuring certificates in webservers

> As the last exercise, we'll do another man-in-the-middle attack, but this time, we'll try to attack SSL traffic. This is an advanced exercise with loose instructions. The exercise is optional, but highly recommended since you will learn a lot about the certificates, certificate authorities, and the dangers of trusting a random certificate authority.

To do the attack, we will create our own Certificate Authority and generate a malicious certificate. Then we trick the client to trust the Certificate Authority we just created. For this exercise, we skip the social engineering part and just install the CA certificate on the client ourselves.

* Create the CA and the webserver on the Server VM.
* Trust the CA and add the domain name on the Client VM.

1. [The first step is to use `easy-rsa` to setup a certificate authority](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-debian-10) Follow this up to and including Step 4. Step 4 explains how to distribute the CA certificate we just created, so other computers can trust that CA. This is the step that an attacker might trick you into doing.
1. The second step is to generate and sign a keypair for your server. To do this, do the `(Optional) — Signing a CSR` step in the previous tutorial. This gives you a private key and certificate for your webserver that wil intercept traffic. You can use a domain name of your chosing. This will be the domain name that you perform the man-in-the-middle attack on.
1. The next step is to install the apache webserver and install the certificate you just created. You can do this by following the guide [How To Create a Self-Signed SSL Certificate for Apache in Ubuntu 22.04](https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-22-04), but skip step 2, as you just created the certificate in the previous step.
1. Finally, we add the domain name to the hosts file of the Client VM, so that when you surf to that domain name, you end up on the server.

## Copyright

You can use and modify this lab as part of your education, but you are not allowed to share this lab, your modifications, and your solutions. Please contact the teaching staff if you want to use (part of) this lab for teaching other courses.

Copyright © teaching staff Ghent University
