# Certificates lab

## Setup

For this lab exercise you need [the same setup as the SSH lab](https://github.ugent.be/security/docs/blob/main/setup-vms.md). You can use the snapshots to revert te VMs. Make sure all VMs are still in working condition by running the "[testing out the VMs](https://github.ugent.be/security/docs/blob/main/setup-vms.md#testing-out-the-vms)" steps at the end. If these fail, you might need to recreate the VMs.

## OpenSSL introduction

The OpenSSL Project develops and maintains the OpenSSL software - a robust, commercial-grade, full-featured toolkit for general-purpose cryptography and secure communication. At the core of this project is the OpenSSL C library. It implements a lot of cryptographic operations such as symmetric encryption, public-key encryption, digital signature, hash functions and so on. OpenSSL also implements the Secure Socket Layer (SSL) protocol used for HTTPS and VPNs like OpenVPN.

The `openssl` CLI application is a command-line interface to this library that allows you to perform virtually any cryptographic operation. You can run `openssl help` to see most common commands and supported ciphers for encryption and decryption.

OpenSSL CLI has [extensive reference documentation](https://www.openssl.org/docs/man3.0/man1/). However, make sure to click on the `openssl-<command-name>` links instead of `<command-name>` itself, as the former contain the actual information about the command. If this confuses you, you are not alone! Even Google in its infinite wisdom doesn't understand how this documentation works and refuses to rank anything higher than the old OpenSSL 1.0.0 documentation, which was saner.

> How do you encrypt the file `legalities.txt` using the `aes-256-cbc` cipher? You can choose a password yourself.

```shell
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 -in legalities.txt -out legalities.txt.enc
```

- `enc`: stand for encode/decode, used for encrypting files
- `-aes-256-cbc`: spefocoes the encryption algorithm
    - `AES-256`: Advanced Encryption Standard with a 256-bit key.
    - `CBC`: Cipher Block Chaining mode, which makes each block depend on the previous one for better security.
- `salt`: adds a random salt before encryption. This ensure that encrypting the same file with the same password twice gives different results. This prevents rainbow attacks.
- `pbkdf2`: uses the modern PBKDF2 (Paswword-Based Key Derivation Function 2) to turn your password into a cryptographic key. This makes brute-force attacks harder.
- `iter 100000`: specifies 100000 iterations for the PBKDF2 process. The higher this number, the longer brute forcing it is gonna take.

> How do you decrypt this file?

```shell
openssl enc -d -aes-256-cbc -pbkdf2 -iter 100000 -in legalities.txt.enc -out legalities.txt
```

- `enc -d`: the `-d` makes it decrypt insteand of encypting it

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
TLS_RSA_WITH_3DES_EDE_CBC_SHA

What algorithms are used:
- Key exchange/authentication: RSA (server certificate signs and authenticates, no ephemeral Diffie-Hellman, so no forward secrecy)
- Symmetric enryption: 3DES (EDE) in CBC mode, "Triple DES, encryption-decrypt-encrypt with DES keying"
- Mode of operation: CBC (Cipher Block Chaining)
- Hash/intigrity: SHA-1 (or SHA)

Why this cypher is not recommended anymore:
- 3DES has a small block size of 64-bits (even though key size is bigger). That means after enough data encrypted under the same  key you can get birthday collisions (the Sweet32 attack) and real-world weaknesses.
- SHA-1 is deprecated/considered weak for many integrity/authenitcation uses.
- Using RSA without epgeeral jey exchanfe means you lack forward secrecy, if the server key is compromised, past sessions van be decrypted.
- CBC mode for TLS has known weaknesses.
- Because of these, many security scanners report suck cipher suites as "weak" or "insecure".
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

```shell
student@security-client:~$ echo | openssl s_client -connect redhat.com:443 -brief
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Ciphersuite: TLS_AES_256_GCM_SHA384
Peer certificate: C = US, ST = North Carolina, L = Raleigh, O = "Red Hat, Inc.", CN = redhat.com
Hash used: SHA256
Signature type: RSA-PSS
Verification: OK
Server Temp Key: X25519, 253 bits
DONE
```

```txt
Yes and more specific:
- Using the protocol TLS 1.3, the newest and most secure version of the TLS protocol.
- ciphersuite: TLS_AES_256_GCM_SHA384
```

[BadSSL](https://badssl.com/) contains a collection of broken or insecure SSL certificates and configurations.

> Take a look at the following servers and explain what is wrong with them. Specifically show the corresponding lines in the output that show the issues.
>
> * `https://tls-v1-0.badssl.com:1010/`
> * `https://expired.badssl.com/`
> * `https://untrusted-root.badssl.com/`
> * `https://self-signed.badssl.com/`
>

```shell
student@security-client:~$ echo | openssl s_client -connect tls-v1-0.badssl.com:443 -briefCONNECTION ESTABLISHED
Protocol version: TLSv1.2                                                                    
Ciphersuite: ECDHE-RSA-AES128-GCM-SHA256                                                     
Peer certificate: CN = *.badssl.com                                                          
Hash used: SHA512                                                                            
Signature type: RSA                                                                          
Verification: OK                                                                             
Supported Elliptic Curve Point Formats: uncompressed:ansiX962_compressed_prime:ansiX962_compressed_char2                                                                                  
Server Temp Key: ECDH, prime256v1, 256 bits                                                  
DONE 

#This server si configured to support the obsolete TLSv1.0 protocol. Older clients would connect uring TLSv1.0, which is insecure and deprecated. Newer clients (like this OpenSSl version) automatically upgraded to TLSv1.2, so the actual connection appears secure, but the server itself still allows outdated TLSv1.0 connections and should disable them.

student@security-client:~$ echo | openssl s_client -connect expired.badssl.com:443 -brief   
depth=0 OU = Domain Control Validated, OU = PositiveSSL Wildcard, CN = *.badssl.com          
verify error:num=10:certificate has expired                                                  
notAfter=Apr 12 23:59:59 2015 GMT                                                            
notAfter=Apr 12 23:59:59 2015 GMT                                                            
CONNECTION ESTABLISHED                                                                       
Protocol version: TLSv1.2                                                                    
Ciphersuite: ECDHE-RSA-AES128-GCM-SHA256
Peer certificate: OU = Domain Control Validated, OU = PositiveSSL Wildcard, CN = *.badssl.com
Hash used: SHA512
Signature type: RSA
Verification error: certificate has expired
Supported Elliptic Curve Point Formats: uncompressed:ansiX962_compressed_prime:ansiX962_compressed_char2
Server Temp Key: ECDH, prime256v1, 256 bits
DONE

# Certifactes were expired in 2015, no more need to be said.

student@security-client:~$ echo | openssl s_client -connect untrusted-root.badssl.com:443 -brief
depth=1 C = US, ST = California, L = San Francisco, O = BadSSL, CN = BadSSL Untrusted Root Certificate Authority
verify error:num=19:self-signed certificate in certificate chain
CONNECTION ESTABLISHED
Protocol version: TLSv1.2
Ciphersuite: ECDHE-RSA-AES128-GCM-SHA256
Peer certificate: C = US, ST = California, L = San Francisco, O = BadSSL, CN = *.badssl.com
Hash used: SHA512
Signature type: RSA
Verification error: self-signed certificate in certificate chain
Supported Elliptic Curve Point Formats: uncompressed:ansiX962_compressed_prime:ansiX962_compressed_char2
Server Temp Key: ECDH, prime256v1, 256 bits
DONE

#The protocol/cipher themselves look fine, but trust fails, so browser/clients must reject it despite strong crypto parameters, since the certificates are self-signed and your system does not trust it.

student@security-client:~$ echo | openssl s_client -connect self-signed.badssl.com:443 -briefdepth=0 C = US, ST = California, L = San Francisco, O = BadSSL, CN = *.badssl.com
verify error:num=18:self-signed certificate
CONNECTION ESTABLISHED
Protocol version: TLSv1.2
Ciphersuite: ECDHE-RSA-AES128-GCM-SHA256
Peer certificate: C = US, ST = California, L = San Francisco, O = BadSSL, CN = *.badssl.com
Hash used: SHA512
Signature type: RSA
Verification error: self-signed certificate
Supported Elliptic Curve Point Formats: uncompressed:ansiX962_compressed_prime:ansiX962_compressed_char2
Server Temp Key: ECDH, prime256v1, 256 bits
DONE

#Yet again self signe certificates, so it is not trusted by your system and refuses.
```
### **Self signed certificates**
- A self-signed certificate is a digital certificate that is signed by the same entity that created it, instead of being signed by a trusted **Certificate Authority (CA)**.

You can also use `openssl` to download the SSL certificate of a server, for example of Google.

```shell
echo | openssl s_client -connect google.com:443 -servername google.com | openssl x509 > /tmp/google.com.cert
```

* `-servername` is used to select the correct certificate when multiple are presented, in the case of SNI.
* `openssl x509` extracts the server certificate from the output. It removes information about the certificate chain and connection details. This is the preferred format to import the certificate into other keystores. Remove this part to see more information about the chain.

> Take a look at the certificate with the entire chain present. What company provides the root of trust of the google.com certificate?

```shell
student@security-client:~$ echo | openssl s_client -showcerts -servername google.com -connect google.com:443 < /dev/null
CONNECTED(00000003)                                                                                                                                                                          
depth=2 C = US, O = Google Trust Services LLC, CN = GTS Root R1                                                                                                                              
verify return:1                                                                                                                                                                              
depth=1 C = US, O = Google Trust Services, CN = WR2                                                                                                                                          
verify return:1                                                                                                                                                                              
depth=0 CN = *.google.com                                                                                                                                                                    
verify return:1                                                                                                                                                                              
---                                                                                                                                                                                          
Certificate chain                                                                                                                                                                            
 0 s:CN = *.google.com                                                                                                                                                                       
   i:C = US, O = Google Trust Services, CN = WR2                                                                                                                                             
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256                                                                                                                                     
   v:NotBefore: Oct 13 08:37:33 2025 GMT; NotAfter: Jan  5 08:37:32 2026 GMT                                                                                                                 
-----BEGIN CERTIFICATE-----                                                                                                                                                                  
MIIOSDCCDTCgAwIBAgIRAPi0qjqwVy5/EBqcNxr+g2swDQYJKoZIhvcNAQELBQAw                                                                                                                             
OzELMAkGA1UEBhMCVVMxHjAcBgNVBAoTFUdvb2dsZSBUcnVzdCBTZXJ2aWNlczEM                                                                                                                             
MAoGA1UEAxMDV1IyMB4XDTI1MTAxMzA4MzczM1oXDTI2MDEwNTA4MzczMlowFzEV                                                                                                                             
MBMGA1UEAwwMKi5nb29nbGUuY29tMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE                                                                                                                             
uU/3qwsIt0mW7oNqUIfYBw/AANxAFNG5msmMQ9pkgBn9V0bnTgixSKoE5DG++jST
JjgRQuk+nYVrbYhfSiUd9qOCDDQwggwwMA4GA1UdDwEB/wQEAwIHgDATBgNVHSUE
DDAKBggrBgEFBQcDATAMBgNVHRMBAf8EAjAAMB0GA1UdDgQWBBSQpTBwlrqS5dnT
1ZaSyN89rRHoCzAfBgNVHSMEGDAWgBTeGx7teRXUPjckwyG77DQ5bUKyMDBYBggr
BgEFBQcBAQRMMEowIQYIKwYBBQUHMAGGFWh0dHA6Ly9vLnBraS5nb29nL3dyMjAl
BggrBgEFBQcwAoYZaHR0cDovL2kucGtpLmdvb2cvd3IyLmNydDCCCgsGA1UdEQSC
CgIwggn+ggwqLmdvb2dsZS5jb22CFiouYXBwZW5naW5lLmdvb2dsZS5jb22CCSou
YmRuLmRldoIVKi5vcmlnaW4tdGVzdC5iZG4uZGV2ghIqLmNsb3VkLmdvb2dsZS5j
b22CGCouY3Jvd2Rzb3VyY2UuZ29vZ2xlLmNvbYIYKi5kYXRhY29tcHV0ZS5nb29n
bGUuY29tggsqLmdvb2dsZS5jYYILKi5nb29nbGUuY2yCDiouZ29vZ2xlLmNvLmlu
gg4qLmdvb2dsZS5jby5qcIIOKi5nb29nbGUuY28udWuCDyouZ29vZ2xlLmNvbS5h
coIPKi5nb29nbGUuY29tLmF1gg8qLmdvb2dsZS5jb20uYnKCDyouZ29vZ2xlLmNv
bS5jb4IPKi5nb29nbGUuY29tLm14gg8qLmdvb2dsZS5jb20udHKCDyouZ29vZ2xl
LmNvbS52boILKi5nb29nbGUuZGWCCyouZ29vZ2xlLmVzggsqLmdvb2dsZS5mcoIL
Ki5nb29nbGUuaHWCCyouZ29vZ2xlLml0ggsqLmdvb2dsZS5ubIILKi5nb29nbGUu
cGyCCyouZ29vZ2xlLnB0gg8qLmdvb2dsZWFwaXMuY26CESouZ29vZ2xldmlkZW8u
Y29tggwqLmdzdGF0aWMuY26CECouZ3N0YXRpYy1jbi5jb22CD2dvb2dsZWNuYXBw
cy5jboIRKi5nb29nbGVjbmFwcHMuY26CEWdvb2dsZWFwcHMtY24uY29tghMqLmdv
b2dsZWFwcHMtY24uY29tggxna2VjbmFwcHMuY26CDiouZ2tlY25hcHBzLmNughJn
b29nbGVkb3dubG9hZHMuY26CFCouZ29vZ2xlZG93bmxvYWRzLmNughByZWNhcHRj
aGEubmV0LmNughIqLnJlY2FwdGNoYS5uZXQuY26CEHJlY2FwdGNoYS1jbi5uZXSC
EioucmVjYXB0Y2hhLWNuLm5ldIILd2lkZXZpbmUuY26CDSoud2lkZXZpbmUuY26C
EWFtcHByb2plY3Qub3JnLmNughMqLmFtcHByb2plY3Qub3JnLmNughFhbXBwcm9q
ZWN0Lm5ldC5jboITKi5hbXBwcm9qZWN0Lm5ldC5jboIXZ29vZ2xlLWFuYWx5dGlj
cy1jbi5jb22CGSouZ29vZ2xlLWFuYWx5dGljcy1jbi5jb22CF2dvb2dsZWFkc2Vy
dmljZXMtY24uY29tghkqLmdvb2dsZWFkc2VydmljZXMtY24uY29tghFnb29nbGV2
YWRzLWNuLmNvbYITKi5nb29nbGV2YWRzLWNuLmNvbYIRZ29vZ2xlYXBpcy1jbi5j
b22CEyouZ29vZ2xlYXBpcy1jbi5jb22CFWdvb2dsZW9wdGltaXplLWNuLmNvbYIX
Ki5nb29nbGVvcHRpbWl6ZS1jbi5jb22CEmRvdWJsZWNsaWNrLWNuLm5ldIIUKi5k
b3VibGVjbGljay1jbi5uZXSCGCouZmxzLmRvdWJsZWNsaWNrLWNuLm5ldIIWKi5n
LmRvdWJsZWNsaWNrLWNuLm5ldIIOZG91YmxlY2xpY2suY26CECouZG91YmxlY2xp
Y2suY26CFCouZmxzLmRvdWJsZWNsaWNrLmNughIqLmcuZG91YmxlY2xpY2suY26C
EWRhcnRzZWFyY2gtY24ubmV0ghMqLmRhcnRzZWFyY2gtY24ubmV0gh1nb29nbGV0
cmF2ZWxhZHNlcnZpY2VzLWNuLmNvbYIfKi5nb29nbGV0cmF2ZWxhZHNlcnZpY2Vz
LWNuLmNvbYIYZ29vZ2xldGFnc2VydmljZXMtY24uY29tghoqLmdvb2dsZXRhZ3Nl
cnZpY2VzLWNuLmNvbYIXZ29vZ2xldGFnbWFuYWdlci1jbi5jb22CGSouZ29vZ2xl
dGFnbWFuYWdlci1jbi5jb22CGGdvb2dsZXN5bmRpY2F0aW9uLWNuLmNvbYIaKi5n
b29nbGVzeW5kaWNhdGlvbi1jbi5jb22CJCouc2FmZWZyYW1lLmdvb2dsZXN5bmRp
Y2F0aW9uLWNuLmNvbYIWYXBwLW1lYXN1cmVtZW50LWNuLmNvbYIYKi5hcHAtbWVh
c3VyZW1lbnQtY24uY29tggtndnQxLWNuLmNvbYINKi5ndnQxLWNuLmNvbYILZ3Z0
Mi1jbi5jb22CDSouZ3Z0Mi1jbi5jb22CCzJtZG4tY24ubmV0gg0qLjJtZG4tY24u
bmV0ghRnb29nbGVmbGlnaHRzLWNuLm5ldIIWKi5nb29nbGVmbGlnaHRzLWNuLm5l
dIIMYWRtb2ItY24uY29tgg4qLmFkbW9iLWNuLmNvbYIZKi5nZW1pbmkuY2xvdWQu
Z29vZ2xlLmNvbYIUZ29vZ2xlc2FuZGJveC1jbi5jb22CFiouZ29vZ2xlc2FuZGJv
eC1jbi5jb22CHiouc2FmZW51cC5nb29nbGVzYW5kYm94LWNuLmNvbYINKi5nc3Rh
dGljLmNvbYIUKi5tZXRyaWMuZ3N0YXRpYy5jb22CCiouZ3Z0MS5jb22CESouZ2Nw
Y2RuLmd2dDEuY29tggoqLmd2dDIuY29tgg4qLmdjcC5ndnQyLmNvbYIQKi51cmwu
Z29vZ2xlLmNvbYIWKi55b3V0dWJlLW5vY29va2llLmNvbYILKi55dGltZy5jb22C
CmFpLmFuZHJvaWSCC2FuZHJvaWQuY29tgg0qLmFuZHJvaWQuY29tghMqLmZsYXNo
LmFuZHJvaWQuY29tggRnLmNuggYqLmcuY26CBGcuY2+CBiouZy5jb4IGZ29vLmds
ggp3d3cuZ29vLmdsghRnb29nbGUtYW5hbHl0aWNzLmNvbYIWKi5nb29nbGUtYW5h
bHl0aWNzLmNvbYIKZ29vZ2xlLmNvbYISZ29vZ2xlY29tbWVyY2UuY29tghQqLmdv
b2dsZWNvbW1lcmNlLmNvbYIIZ2dwaHQuY26CCiouZ2dwaHQuY26CCnVyY2hpbi5j
b22CDCoudXJjaGluLmNvbYIIeW91dHUuYmWCC3lvdXR1YmUuY29tgg0qLnlvdXR1
YmUuY29tghFtdXNpYy55b3V0dWJlLmNvbYITKi5tdXNpYy55b3V0dWJlLmNvbYIU
eW91dHViZWVkdWNhdGlvbi5jb22CFioueW91dHViZWVkdWNhdGlvbi5jb22CD3lv
dXR1YmVraWRzLmNvbYIRKi55b3V0dWJla2lkcy5jb22CBXl0LmJlggcqLnl0LmJl
ghphbmRyb2lkLmNsaWVudHMuZ29vZ2xlLmNvbYITKi5hbmRyb2lkLmdvb2dsZS5j
boISKi5jaHJvbWUuZ29vZ2xlLmNughYqLmRldmVsb3BlcnMuZ29vZ2xlLmNughUq
LmFpc3R1ZGlvLmdvb2dsZS5jb20wEwYDVR0gBAwwCjAIBgZngQwBAgEwNgYDVR0f
BC8wLTAroCmgJ4YlaHR0cDovL2MucGtpLmdvb2cvd3IyL29CRllZYWh6Z1ZJLmNy
bDCCAQMGCisGAQQB1nkCBAIEgfQEgfEA7wB2AJaXZL9VWJet90OHaDcIQnfp8DrV
9qTzNm5GpD8PyqnGAAABmdzuwjoAAAQDAEcwRQIhAIy8EyIagFKx1QZNf+adVnqU
y6C4DYDEiMBz2b3CK9+aAiBgOuqZRK3VQo/csWsCtvToWpC6NX66sRuDGwskHhsW
MwB1ABaDLavwqSUPD/A6pUX/yL/II9CHS/YEKSf45x8zE/X6AAABmdzuwiQAAAQD
AEYwRAIgGyuSC8qGviN0b1/0piYTs///a5QDbjyWi2tMxcjqVoMCIDGzlyHlWJEp
lCYges7uN3kP2okwvrpwRRQydhH8BLUBMA0GCSqGSIb3DQEBCwUAA4IBAQBawvcq
Ae848RbT2VvDTVtG7P/TNyfOae5GfHDyXyN4JHj4uHgueblOsK2ZfCsmhHqAZFSk
bvLCkEUvTVFq+Esjz0MROEYT4ZTy4ql4nsi31c/VX64WKLdwIOG7/vzMRSZFgvBS
Hv+9xm7YGPP8CVzIgmi9j2+GES2G4DoD7BSi5dml7dAzOxj6xOhaVk53l3SgTS7X
C3ZUZSwYS3YxGq8bwDtWirZUZy7GBIvuIte3NPitzHxN6KQUp1Gztrtxgc1McdVo
oCdvl3zmL148mGsZOniU96AiRgwHdGq2Lf9UYh8mXqPMHIzYb7yP6Ar6GmLpp0gY
vUh1cHoMYHjxtl1e
-----END CERTIFICATE-----
 1 s:C = US, O = Google Trust Services, CN = WR2
   i:C = US, O = Google Trust Services LLC, CN = GTS Root R1
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Dec 13 09:00:00 2023 GMT; NotAfter: Feb 20 14:00:00 2029 GMT
-----BEGIN CERTIFICATE-----
MIIFCzCCAvOgAwIBAgIQf/AFoHxM3tEArZ1mpRB7mDANBgkqhkiG9w0BAQsFADBH
MQswCQYDVQQGEwJVUzEiMCAGA1UEChMZR29vZ2xlIFRydXN0IFNlcnZpY2VzIExM
QzEUMBIGA1UEAxMLR1RTIFJvb3QgUjEwHhcNMjMxMjEzMDkwMDAwWhcNMjkwMjIw
MTQwMDAwWjA7MQswCQYDVQQGEwJVUzEeMBwGA1UEChMVR29vZ2xlIFRydXN0IFNl
cnZpY2VzMQwwCgYDVQQDEwNXUjIwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQCp/5x/RR5wqFOfytnlDd5GV1d9vI+aWqxG8YSau5HbyfsvAfuSCQAWXqAc
+MGr+XgvSszYhaLYWTwO0xj7sfUkDSbutltkdnwUxy96zqhMt/TZCPzfhyM1IKji
aeKMTj+xWfpgoh6zySBTGYLKNlNtYE3pAJH8do1cCA8Kwtzxc2vFE24KT3rC8gIc
LrRjg9ox9i11MLL7q8Ju26nADrn5Z9TDJVd06wW06Y613ijNzHoU5HEDy01hLmFX
xRmpC5iEGuh5KdmyjS//V2pm4M6rlagplmNwEmceOuHbsCFx13ye/aoXbv4r+zgX
FNFmp6+atXDMyGOBOozAKql2N87jAgMBAAGjgf4wgfswDgYDVR0PAQH/BAQDAgGG
MB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjASBgNVHRMBAf8ECDAGAQH/
AgEAMB0GA1UdDgQWBBTeGx7teRXUPjckwyG77DQ5bUKyMDAfBgNVHSMEGDAWgBTk
rysmcRorSCeFL1JmLO/wiRNxPjA0BggrBgEFBQcBAQQoMCYwJAYIKwYBBQUHMAKG
GGh0dHA6Ly9pLnBraS5nb29nL3IxLmNydDArBgNVHR8EJDAiMCCgHqAchhpodHRw
Oi8vYy5wa2kuZ29vZy9yL3IxLmNybDATBgNVHSAEDDAKMAgGBmeBDAECATANBgkq
hkiG9w0BAQsFAAOCAgEARXWL5R87RBOWGqtY8TXJbz3S0DNKhjO6V1FP7sQ02hYS
TL8Tnw3UVOlIecAwPJQl8hr0ujKUtjNyC4XuCRElNJThb0Lbgpt7fyqaqf9/qdLe
SiDLs/sDA7j4BwXaWZIvGEaYzq9yviQmsR4ATb0IrZNBRAq7x9UBhb+TV+PfdBJT
DhEl05vc3ssnbrPCuTNiOcLgNeFbpwkuGcuRKnZc8d/KI4RApW//mkHgte8y0YWu
ryUJ8GLFbsLIbjL9uNrizkqRSvOFVU6xddZIMy9vhNkSXJ/UcZhjJY1pXAprffJB
vei7j+Qi151lRehMCofa6WBmiA4fx+FOVsV2/7R6V2nyAiIJJkEd2nSi5SnzxJrl
Xdaqev3htytmOPvoKWa676ATL/hzfvDaQBEcXd2Ppvy+275W+DKcH0FBbX62xevG
iza3F4ydzxl6NJ8hk8R+dDXSqv1MbRT1ybB5W0k8878XSOjvmiYTDIfyc9acxVJr
Y/cykHipa+te1pOhv7wYPYtZ9orGBV5SGOJm4NrB3K1aJar0RfzxC3ikr7Dyc6Qw
qDTBU39CluVIQeuQRgwG3MuSxl7zRERDRilGoKb8uY45JzmxWuKxrfwT/478JuHU
/oTxUFqOl2stKnn7QGTq8z29W+GgBLCXSBxC9epaHM0myFH/FJlniXJfHeytWt0=
-----END CERTIFICATE-----
 2 s:C = US, O = Google Trust Services LLC, CN = GTS Root R1
   i:C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
   a:PKEY: rsaEncryption, 4096 (bit); sigalg: RSA-SHA256
   v:NotBefore: Jun 19 00:00:42 2020 GMT; NotAfter: Jan 28 00:00:42 2028 GMT
-----BEGIN CERTIFICATE-----
MIIFYjCCBEqgAwIBAgIQd70NbNs2+RrqIQ/E8FjTDTANBgkqhkiG9w0BAQsFADBX
MQswCQYDVQQGEwJCRTEZMBcGA1UEChMQR2xvYmFsU2lnbiBudi1zYTEQMA4GA1UE
CxMHUm9vdCBDQTEbMBkGA1UEAxMSR2xvYmFsU2lnbiBSb290IENBMB4XDTIwMDYx
OTAwMDA0MloXDTI4MDEyODAwMDA0MlowRzELMAkGA1UEBhMCVVMxIjAgBgNVBAoT
GUdvb2dsZSBUcnVzdCBTZXJ2aWNlcyBMTEMxFDASBgNVBAMTC0dUUyBSb290IFIx
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAthECix7joXebO9y/lD63
ladAPKH9gvl9MgaCcfb2jH/76Nu8ai6Xl6OMS/kr9rH5zoQdsfnFl97vufKj6bwS
iV6nqlKr+CMny6SxnGPb15l+8Ape62im9MZaRw1NEDPjTrETo8gYbEvs/AmQ351k
KSUjB6G00j0uYODP0gmHu81I8E3CwnqIiru6z1kZ1q+PsAewnjHxgsHA3y6mbWwZ
DrXYfiYaRQM9sHmklCitD38m5agI/pboPGiUU+6DOogrFZYJsuB6jC511pzrp1Zk
j5ZPaK49l8KEj8C8QMALXL32h7M1bKwYUH+E4EzNktMg6TO8UpmvMrUpsyUqtEj5
cuHKZPfmghCN6J3Cioj6OGaK/GP5Afl4/Xtcd/p2h/rs37EOeZVXtL0m79YB0esW
CruOC7XFxYpVq9Os6pFLKcwZpDIlTirxZUTQAs6qzkm06p98g7BAe+dDq6dso499
iYH6TKX/1Y7DzkvgtdizjkXPdsDtQCv9Uw+wp9U7DbGKogPeMa3Md+pvez7W35Ei
Eua++tgy/BBjFFFy3l3WFpO9KWgz7zpm7AeKJt8T11dleCfeXkkUAKIAf5qoIbap
sZWwpbkNFhHax2xIPEDgfg1azVY80ZcFuctL7TlLnMQ/0lUTbiSw1nH69MG6zO0b
9f6BQdgAmD06yK56mDcYBZUCAwEAAaOCATgwggE0MA4GA1UdDwEB/wQEAwIBhjAP
BgNVHRMBAf8EBTADAQH/MB0GA1UdDgQWBBTkrysmcRorSCeFL1JmLO/wiRNxPjAf
BgNVHSMEGDAWgBRge2YaRQ2XyolQL30EzTSo//z9SzBgBggrBgEFBQcBAQRUMFIw
JQYIKwYBBQUHMAGGGWh0dHA6Ly9vY3NwLnBraS5nb29nL2dzcjEwKQYIKwYBBQUH
MAKGHWh0dHA6Ly9wa2kuZ29vZy9nc3IxL2dzcjEuY3J0MDIGA1UdHwQrMCkwJ6Al
oCOGIWh0dHA6Ly9jcmwucGtpLmdvb2cvZ3NyMS9nc3IxLmNybDA7BgNVHSAENDAy
MAgGBmeBDAECATAIBgZngQwBAgIwDQYLKwYBBAHWeQIFAwIwDQYLKwYBBAHWeQIF
AwMwDQYJKoZIhvcNAQELBQADggEBADSkHrEoo9C0dhemMXoh6dFSPsjbdBZBiLg9
NR3t5P+T4Vxfq7vqfM/b5A3Ri1fyJm9bvhdGaJQ3b2t6yMAYN/olUazsaL+yyEn9
WprKASOshIArAoyZl+tJaox118fessmXn1hIVw41oeQa1v1vg4Fv74zPl6/AhSrw
9U5pCZEt4Wi4wStz6dTZ/CLANx8LZh1J7QJVj2fhMtfTJr9w4z30Z209fOU0iOMy
+qduBmpvvYuR7hZL6Dupszfnw0Skfths18dG9ZKb59UhvmaSGZRVbNQpsg3BZlvi
d0lIKO2d1xozclOzgjXPYovJJIultzkMu34qQb9Sz/yilrbCgj8=
-----END CERTIFICATE-----
---
Server certificate
subject=CN = *.google.com
issuer=C = US, O = Google Trust Services, CN = WR2
---
No client certificate CA names sent
Peer signing digest: SHA256
Peer signature type: ECDSA
Server Temp Key: X25519, 253 bits
---
SSL handshake has read 6652 bytes and written 392 bytes
Verification: OK
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
Early data was not sent
Verify return code: 0 (ok)
---
DONE
```

```txt
When you connect to a website (liek google.com), the server usually sends a chain of vertificates:
1. Leaf certificate - identifies the actual website
2. intermediate certificate(s) - issued by a CA to sign many websites
3. Root certificate - issued by a trusted root CA and stored in your OS/browser trust store.
```

## Checking certificate extensions

X509 extensions allow for additional fields to be added to a certificate. One of the most common is the subject alternative name (SAN). The SAN of a certificate allows multiple values (e.g., multiple FQDNs) to be associated with a single certificate. The SAN is even used when there aren’t multiple values because the use of a certificate’s common name for verification is deprecated. You can show an extension of a certificate using `openssl x509 -noout -ext <extensionName>`

* `-noout` instructs the command to not output the entire certificate after the output. Without this flag, the command will show the extensions and then output the entire certificate.

Piping output between multiple OpenSSL commands makes it easy to inspect specific certificate extensions. However, don't forget to redirect stderr away if you pipe from `openssl s_client`.

> Download the certificate of `redhat.com` and show the contents of the `subjectAltName` extension using a one-liner.

```shell
echo | openssl s_client -connect redhat.com:443 -servername redhat.com 2>/dev/null | openssl x509 -noout -ext subjectAltName

student@security-client:~$ echo | openssl s_client -connect redhat.com:443 -servername redhat.com 2>/dev/null | openssl x509 -noout -ext subjectAltName
X509v3 Subject Alternative Name: 
    DNS:redhat.com
```


Another common set of extensions include the basic constraints and key usage of a certificate. Specifically, you might want to check if a certificate is allowed to be used as a certificate authority. Again, this can be done in the same way that you can check for a SAN: `openssl x509 -ext basicConstraints,keyUsage`

> Show the contents of `basicConstraints` and `keyUsage` extensions from the Google certificate and the GlobalSign Root CA, which you can find in `/usr/share/ca-certificates/mozilla/GlobalSign_Root_CA.crt`.

```shell
student@security-client:~$ cat /tmp/google.com.cert | openssl x509 -noout -ext basicConstraints -ext keyUsage                                                                             
X509v3 Key Usage: critical
    Digital Signature
```

> What does the `critical` stand for? You can use the [Certificate Extension Reference](https://access.redhat.com/documentation/en-us/red_hat_certificate_system/9/html/administration_guide/standard_x.509_v3_certificate_extensions) to get a lot of information about X.509 extensions.

```txt
The 'critical' flag means that the extension is essential for correctly interpreting the certficate.
If a client or application does not understand the critical extension, it must reject the certificate.
Non-critical extensions can safely be ignored if it is unsupported.
```

Finally, if you want to see the entire contents of a certificate, you can use `openssl x509 -text`

> Show the entire contents of the Google certificate.

```shell
student@security-client:~$ openssl x509 -in /tmp/google.com.cert -text -noout




Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            f8:b4:aa:3a:b0:57:2e:7f:10:1a:9c:37:1a:fe:83:6b
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = Google Trust Services, CN = WR2
        Validity
            Not Before: Oct 13 08:37:33 2025 GMT
            Not After : Jan  5 08:37:32 2026 GMT
        Subject: CN = *.google.com
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:b9:4f:f7:ab:0b:08:b7:49:96:ee:83:6a:50:87:
                    d8:07:0f:c0:00:dc:40:14:d1:b9:9a:c9:8c:43:da:
                    64:80:19:fd:57:46:e7:4e:08:b1:48:aa:04:e4:31:
                    be:fa:34:93:26:38:11:42:e9:3e:9d:85:6b:6d:88:
                    5f:4a:25:1d:f6
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier: 
                90:A5:30:70:96:BA:92:E5:D9:D3:D5:96:92:C8:DF:3D:AD:11:E8:0B
            X509v3 Authority Key Identifier: 
                DE:1B:1E:ED:79:15:D4:3E:37:24:C3:21:BB:EC:34:39:6D:42:B2:30
            Authority Information Access: 
                OCSP - URI:http://o.pki.goog/wr2
                CA Issuers - URI:http://i.pki.goog/wr2.crt
            X509v3 Subject Alternative Name: 
                DNS:*.google.com, DNS:*.appengine.google.com, DNS:*.bdn.dev, DNS:*.origin-test.bdn.dev, DNS:*.cloud.google.com, DNS:*.crowdsource.google.com, DNS:*.datacompute.google.com, DNS:*.google.ca, DNS:*.google.cl, DNS:*.google.co.in, DNS:*.google.co.jp, DNS:*.google.co.uk, DNS:*.google.com.ar, DNS:*.google.com.au, DNS:*.google.com.br, DNS:*.google.com.co, DNS:*.google.com.mx, DNS:*.google.com.tr, DNS:*.google.com.vn, DNS:*.google.de, DNS:*.google.es, DNS:*.google.fr, DNS:*.google.hu, DNS:*.google.it, DNS:*.google.nl, DNS:*.google.pl, DNS:*.google.pt, DNS:*.googleapis.cn, DNS:*.googlevideo.com, DNS:*.gstatic.cn, DNS:*.gstatic-cn.com, DNS:googlecnapps.cn, DNS:*.googlecnapps.cn, DNS:googleapps-cn.com, DNS:*.googleapps-cn.com, DNS:gkecnapps.cn, DNS:*.gkecnapps.cn, DNS:googledownloads.cn, DNS:*.googledownloads.cn, DNS:recaptcha.net.cn, DNS:*.recaptcha.net.cn, DNS:recaptcha-cn.net, DNS:*.recaptcha-cn.net, DNS:widevine.cn, DNS:*.widevine.cn, DNS:ampproject.org.cn, DNS:*.ampproject.org.cn, DNS:ampproject.net.cn, DNS:*.ampproject.net.cn, DNS:google-analytics-cn.com, DNS:*.google-analytics-cn.com, DNS:googleadservices-cn.com, DNS:*.googleadservices-cn.com, DNS:googlevads-cn.com, DNS:*.googlevads-cn.com, DNS:googleapis-cn.com, DNS:*.googleapis-cn.com, DNS:googleoptimize-cn.com, DNS:*.googleoptimize-cn.com, DNS:doubleclick-cn.net, DNS:*.doubleclick-cn.net, DNS:*.fls.doubleclick-cn.net, DNS:*.g.doubleclick-cn.net, DNS:doubleclick.cn, DNS:*.doubleclick.cn, DNS:*.fls.doubleclick.cn, DNS:*.g.doubleclick.cn, DNS:dartsearch-cn.net, DNS:*.dartsearch-cn.net, DNS:googletraveladservices-cn.com, DNS:*.googletraveladservices-cn.com, DNS:googletagservices-cn.com, DNS:*.googletagservices-cn.com, DNS:googletagmanager-cn.com, DNS:*.googletagmanager-cn.com, DNS:googlesyndication-cn.com, DNS:*.googlesyndication-cn.com, DNS:*.safeframe.googlesyndication-cn.com, DNS:app-measurement-cn.com, DNS:*.app-measurement-cn.com, DNS:gvt1-cn.com, DNS:*.gvt1-cn.com, DNS:gvt2-cn.com, DNS:*.gvt2-cn.com, DNS:2mdn-cn.net, DNS:*.2mdn-cn.net, DNS:googleflights-cn.net, DNS:*.googleflights-cn.net, DNS:admob-cn.com, DNS:*.admob-cn.com, DNS:*.gemini.cloud.google.com, DNS:googlesandbox-cn.com, DNS:*.googlesandbox-cn.com, DNS:*.safenup.googlesandbox-cn.com, DNS:*.gstatic.com, DNS:*.metric.gstatic.com, DNS:*.gvt1.com, DNS:*.gcpcdn.gvt1.com, DNS:*.gvt2.com, DNS:*.gcp.gvt2.com, DNS:*.url.google.com, DNS:*.youtube-nocookie.com, DNS:*.ytimg.com, DNS:ai.android, DNS:android.com, DNS:*.android.com, DNS:*.flash.android.com, DNS:g.cn, DNS:*.g.cn, DNS:g.co, DNS:*.g.co, DNS:goo.gl, DNS:www.goo.gl, DNS:google-analytics.com, DNS:*.google-analytics.com, DNS:google.com, DNS:googlecommerce.com, DNS:*.googlecommerce.com, DNS:ggpht.cn, DNS:*.ggpht.cn, DNS:urchin.com, DNS:*.urchin.com, DNS:youtu.be, DNS:youtube.com, DNS:*.youtube.com, DNS:music.youtube.com, DNS:*.music.youtube.com, DNS:youtubeeducation.com, DNS:*.youtubeeducation.com, DNS:youtubekids.com, DNS:*.youtubekids.com, DNS:yt.be, DNS:*.yt.be, DNS:android.clients.google.com, DNS:*.android.google.cn, DNS:*.chrome.google.cn, DNS:*.developers.google.cn, DNS:*.aistudio.google.com
            X509v3 Certificate Policies: 
                Policy: 2.23.140.1.2.1
            X509v3 CRL Distribution Points: 
                Full Name:
                  URI:http://c.pki.goog/wr2/oBFYYahzgVI.crl
            CT Precertificate SCTs: 
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 96:97:64:BF:55:58:97:AD:F7:43:87:68:37:08:42:77:
                                E9:F0:3A:D5:F6:A4:F3:36:6E:46:A4:3F:0F:CA:A9:C6
                    Timestamp : Oct 13 09:37:38.874 2025 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:45:02:21:00:8C:BC:13:22:1A:80:52:B1:D5:06:4D:
                                7F:E6:9D:56:7A:94:CB:A0:B8:0D:80:C4:88:C0:73:D9:
                                BD:C2:2B:DF:9A:02:20:60:3A:EA:99:44:AD:D5:42:8F:
                                DC:B1:6B:02:B6:F4:E8:5A:90:BA:35:7E:BA:B1:1B:83:
                                1B:0B:24:1E:1B:16:33
                Signed Certificate Timestamp:
                    Version   : v1 (0x0)
                    Log ID    : 16:83:2D:AB:F0:A9:25:0F:0F:F0:3A:A5:45:FF:C8:BF:
                                C8:23:D0:87:4B:F6:04:29:27:F8:E7:1F:33:13:F5:FA
                    Timestamp : Oct 13 09:37:38.852 2025 GMT
                    Extensions: none
                    Signature : ecdsa-with-SHA256
                                30:44:02:20:1B:2B:92:0B:CA:86:BE:23:74:6F:5F:F4:
                                A6:26:13:B3:FF:FF:6B:94:03:6E:3C:96:8B:6B:4C:C5:
                                C8:EA:56:83:02:20:31:B3:97:21:E5:58:91:29:94:26:
                                20:7A:CE:EE:37:79:0F:DA:89:30:BE:BA:70:45:14:32:
                                76:11:FC:04:B5:01
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        5a:c2:f7:2a:01:ef:38:f1:16:d3:d9:5b:c3:4d:5b:46:ec:ff:
        d3:37:27:ce:69:ee:46:7c:70:f2:5f:23:78:24:78:f8:b8:78:
        2e:79:b9:4e:b0:ad:99:7c:2b:26:84:7a:80:64:54:a4:6e:f2:
        c2:90:45:2f:4d:51:6a:f8:4b:23:cf:43:11:38:46:13:e1:94:
        f2:e2:a9:78:9e:c8:b7:d5:cf:d5:5f:ae:16:28:b7:70:20:e1:
        bb:fe:fc:cc:45:26:45:82:f0:52:1e:ff:bd:c6:6e:d8:18:f3:
        fc:09:5c:c8:82:68:bd:8f:6f:86:11:2d:86:e0:3a:03:ec:14:
        a2:e5:d9:a5:ed:d0:33:3b:18:fa:c4:e8:5a:56:4e:77:97:74:
        a0:4d:2e:d7:0b:76:54:65:2c:18:4b:76:31:1a:af:1b:c0:3b:
        56:8a:b6:54:67:2e:c6:04:8b:ee:22:d7:b7:34:f8:ad:cc:7c:
        4d:e8:a4:14:a7:51:b3:b6:bb:71:81:cd:4c:71:d5:68:a0:27:
        6f:97:7c:e6:2f:5e:3c:98:6b:19:3a:78:94:f7:a0:22:46:0c:
        07:74:6a:b6:2d:ff:54:62:1f:26:5e:a3:cc:1c:8c:d8:6f:bc:
        8f:e8:0a:fa:1a:62:e9:a7:48:18:bd:48:75:70:7a:0c:60:78:
        f1:b6:5d:5e
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
student@security-client:~$ echo | openssl s_client -connect redhat.com:443 -cipher ECDHE-RSA-AES128-SHA -tls1_2 -brief
40F716BAC27F0000:error:0A000410:SSL routines:ssl3_read_bytes:sslv3 alert handshake failure:../ssl/record/rec_layer_s3.c:1593:SSL alert number 40
```

> Now try this with the server at `tls13.1d.pw:443`

```shell
student@security-client:~$ echo | openssl s_client -connect tls13.1d.pw:443 -cipher ECDHE-RSA-AES128-SHA -tls1_2 -brief
40370515257F0000:error:8000006F:system library:BIO_connect:Connection refused:../crypto/bio/bio_sock2.c:125:calling connect()
40370515257F0000:error:10000067:BIO routines:BIO_connect:connect error:../crypto/bio/bio_sock2.c:127:
connect:errno=111
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
student@security-client:~$ nmap --script ssl-enum-ciphers -p 443 rc4-md5.badssl.com
Starting Nmap 7.80 ( https://nmap.org ) at 2025-11-13 09:10 CET
Nmap scan report for rc4-md5.badssl.com (104.154.89.105)
Host is up (0.18s latency).
rDNS record for 104.154.89.105: 105.89.154.104.bc.googleusercontent.com

PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers: 
|   TLSv1.0: 
|     ciphers: 
|       TLS_RSA_WITH_RC4_128_MD5 (rsa 2048) - C
|     compressors: 
|       NULL
|     cipher preference: indeterminate
|     cipher preference error: Too few ciphers supported
|     warnings: 
|       Broken cipher RC4 is deprecated by RFC 7465
|       Ciphersuite uses MD5 for message integrity
|       Forward Secrecy not supported by any cipher
|   TLSv1.1: 
|     ciphers: 
|       TLS_RSA_WITH_RC4_128_MD5 (rsa 2048) - C
|     compressors: 
|       NULL
|     cipher preference: indeterminate
|     cipher preference error: Too few ciphers supported
|     warnings: 
|       Broken cipher RC4 is deprecated by RFC 7465
|       Ciphersuite uses MD5 for message integrity
|       Forward Secrecy not supported by any cipher
|   TLSv1.2: 
|     ciphers: 
|       TLS_RSA_WITH_RC4_128_MD5 (rsa 2048) - C
|     compressors: 
|       NULL
|     cipher preference: indeterminate
|     cipher preference error: Too few ciphers supported
|     warnings: 
|       Broken cipher RC4 is deprecated by RFC 7465
|       Ciphersuite uses MD5 for message integrity
|       Forward Secrecy not supported by any cipher
|_  least strength: C

Nmap done: 1 IP address (1 host up) scanned in 4.35 seconds
```

```txt
The server rc4-md5.badssl.com only supports one cipher:
  TLS_RSA_WITH_RC4_128_MD5
This cipher is insecure because RC4 is a broken stream cipher (deprecated by RFC 7465), it uses MD5 for integrity (cryptographically unsafe), and it provides no forward secrecy. Therefore, none of the supported ciphers are safe.
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
openssl x509 -in cert.pem -outform DER -out cert.der
# Convert DER to PEM
openssl x509 -in cert.der -inform DER -out cert.pem
    #website
openssl x509 -in certificatename.cer -outform PEM -out certificatename.pem
# Convert PEM to P7B
openssl crl2pkcs7 -nocrl -certfile cert.pem -out cert.p7b
    #website
openssl crl2pkcs7 -nocrl -certfile certificatename.pem -out certificatename.p7b -certfile CACert.cer
# Convert P7B to PEM
openssl pkcs7 -print_certs -in cert.p7b -out cert.pem
# Convert pfx to PEM
openssl pkcs12 -in cert.pfx -out cert.pem -nodes
    #website
openssl pkcs12 -in certificatename.pfx -out certificatename.pem
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
