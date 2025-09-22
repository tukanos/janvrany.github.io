---
title: "How to generate TSL/SSL certificate and get it signed"
created_at: 2017-08-17 14:39:46 +0100
kind: article
published: true
---

## Introduction

I recently had a need to provide an access to my private [OpenVPN][2] server
to couple of friends / colleagues. I ran my very own [CA][3] for the purpose
of issuing certificates for [OpenVPN][2] clients. Usually I generate a key and
(signed) certificate and give it to the client in a secure-enough way, e.g on
my own USB stick. 

In this case these guys live far, far away, so far that it's impractical to meet
them face-to-face. Indeed I can generate key and certificate as usual, send them
these over an email (or a Jabber, Slack channel, you name it). I'd rather not to
do so, these channels are unsafe so someone may intercept and steal both the
key and the certificate. Let's do it properly, after all the whole point of VPN
is to have a secure access and if the process of exchanging credentials is 
insecure, it somewhat spoils the idea. 

How to issue them a working signed certificate in a secure way? 
That's the question...I hope this would shed some light on this.

<!-- more -->

## Background

[OpenVPN][2] uses TSL/SSL certificates to authenticate clients, this means that
server accepts connection from any client that presents a valid certificate.
"Valid" means certificate that is signed by a [CA][3] the server is configured
to trust. So each client needs it's private key (`.key`) and certificate (`.crt`).

As in any [public key cryprography][4], the private key must be kept secret and
never be passed to an untrusted entity (whatever it means) while the certificate 
(essentially a public key plus some identification information) can be passed 
freely. So the overall process is the following: 

  1. The *client* (in my case the friend who lives far, far away) generates 
     a key and certificate signing request (`.csr`). 

  2. The *client* sends the request to the CA, possibly over insecure channel
     (such as email or Jabber). 

  3. The *CA* verifies the identity of the subject that sent the (`csr). This 
     is the tricky bit, see comment below. 

  4. The *CA* signs the request and issues a signed certificate (`.crt`).

  5. The *CA* sends the certificate back to the client over possibly insecure 
     channel. 

Now the client has all it needs: the private key and the signed certificate. What's
essential here is that the key need not to leave clients' computer. 

The step 3. it both important and tricky. The CA must verify the one who sent the
request it really the request is claiming to be. Otherwise, anyone (read: bad guy)
can send a request on behalf of someone else and the CA would happily sign the 
request. The means of verifying the identity differ among CAs, some do rigorous 
checks, some do no checks at all. 

For my very case, I'd ask the client (my friend) to tell me a fingerprint of the
`.csr` she just sent me *over different channel* than she sent me the `.csr` itself.
So she can send me the `.csr` over email and then - ideally - tell me the 
fingerprint in voice over Skype. Practically, confirming it over Slack would do
too. 

Depending on your paranoia, the above may or may not be "safe-enough".

## Let's make it practical

A step-by-step guide using [OpenSSL][5]:

### Step 1. Generate key and signing request (client)

  1. As we're using `openssl`, first we'd need a configuration file for it. 
     Following would do (save it to some file, in the following we'd 
     use `ssl.conf`):

         [ req ]
         default_bits	= 2048
         distinguished_name	= req_distinguished_name
         prompt		= yes
         default_md		= sha256

         [ req_distinguished_name ]
        
         [ client ]
         basicConstraints	= critical, CA:FALSE
         subjectKeyIdentifier	= hash
         authorityKeyIdentifier	= keyid:always,issuer:always
         keyUsage		= digitalSignature, keyEncipherment
         nsCertType		= client
         extendedKeyUsage	= critical, clientAuth

  2. Generate a new key:

         openssl genrsa -out client.key -des3 2048

     This will generate a new 2048 bit RSA (private) key and save it to `client.key`. 
     This is the secret file no-one else should have access too. The `-des3` 
     encrypts the key using a passphrase - you'll be asked for it. Depending on 
     your paranoia you may increase key bitsize to say 4096. 

  3. Generate certificate signing request

     Certificate signing request (`.csr`) is a file that it later processed by 
     the CA to actually issue a signed certificate. Essentially, `.csr` contains 
     a public key plus some other metadata such as who's certificate it's going 
     be. This is a lengthy command:

         openssl req -config 'ssl.conf' -new -key 'client.key' -out 'client.csr' \
                     -subj '/C=UK/ST=United Kingdom/L=Dundee/O=Some Company/emailAddress=some.guy@somecompany.co.uk/CN=Guy Some/'

     *Important:* replace values in `-subj` accordingly - this identifies who's 
     certificate it is. 

     This produces a file `client.csr` which we will need shortly. To double
     check the contents `.csr`, most importantly the subject (person) asking
     for the certificate, do:

         openssl req  -subject -in client.csr


### Step 2. Send the signing request to the CA (client)

This is easy, just take a `.csr` you get in previous step and send it to the
CA, anyhow. In my case, my friends had to send the `.csr` to me by email or 
Slack. 

### Step. 3. Verify the `.csr` is genuine. 

The depends on the CA. In my case, I ask my friends to tell me the fingerprint
over another channel. This is how one can get the fingerprint of a `.csr`:

    openssl dgst -sha256 client.csr

If this matches the fingerprint of the `.csr` I received (presumably, but not 
for sure) from my friend, I consider the identity verified and can proceed to the
next step - signing the request and issuing the certificate.

### Step 4. Sign the request and issue the certificate (CA)

For the sake of completeness, this is how the CA signs the request and issues
the certificate. It assumes the CA is properly set up:

    openssl ca --config ~/SomeOtherCA/ssl.conf -batch \
              -infiles ~/SomeOtherCA/archive/Guy Some/client.csr \
              -out ~/SomeOtherCA/archive/Guy Some/client.crt \
              -key '/SomeOtherCA/ca.key' \
              -startdate $(date +%Y%m%d%H%SZ -u) -extensions client -days '365' 

Actually, different CAs and CA management softwares might do it differently. the
above is how I do it. 

### Step 5. Send the certificate back to the client (CA)

This is easy, just take a `.crt` and send it back using whatever suits you best.


# Job done! 

Hope that helps, have fun. 


# Acknowledgment 

I'd like to thank ID NEPHIRUS and other people who prefer to stay
anonymous for their valuable comments. 

[1]: https://en.wikipedia.org/wiki/Public_key_infrastructure
[2]: https://openvpn.net/
[3]: https://en.wikipedia.org/wiki/Certificate_authority
[4]: https://en.wikipedia.org/wiki/Public-key_cryptography
[5]: https://www.openssl.org/
