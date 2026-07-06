## How does TLS work and how is SPIFFE related to it?

How does TLS encryption/decryption work?.. presumably with public key
crypto, but then where do the private keys live?..
https://istio.io/latest/docs/ops/configuration/traffic-management/manage-mesh-certificates/
> ### Managing In-Mesh Certificates
> #### Sidecars
> Since sidecars manage their own certificates for in-mesh communication, the
> sidecars are responsible for managing their private keys and generated
> Certificate Signing Request (CSRs).

By the way, SPIFFE's "X.509-SVID"s do include a private key:
https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#spiffe-workload-api
> For identity documents in X.509 format (an X.509-SVID):
> * Its identity, described as a SPIFFE ID.
> * A private key tied to that ID that can be used to sign data on behalf
>   of the workload.
>   A corresponding short-lived X.509 certificate is also created, the X509-SVID.
>   This can be used to establish TLS or otherwise authenticate to other workloads.
> * A set of certificates – known as a trust bundle – that a workload can use
>   to verify an X.509-SVID presented by another workload

Ah ha, and we can see the private key listed in the X509SVID structure's
`x509_svid_key` field:
https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Workload_API.md#5-x509-svid-profile
```
message X509SVID {
    string spiffe_id = 1;

    // Required. ASN.1 DER encoded (not PEM) certificate chain. MAY include
    // intermediates, the leaf certificate (or SVID itself) MUST come first.
    bytes x509_svid = 2;

    // Required. ASN.1 DER encoded (not PEM) PKCS#8 private key. MUST be unencrypted.
    bytes x509_svid_key = 3;

    bytes bundle = 4;
    string hint = 5;
}
```

So then, who uses that `x509_svid_key`?.. not the application, I don't think.
I'm suspecting that perhaps the idea is:
* You're running your services in Kubernetes
* There's a proxy sidecar, maybe Istio with SPIFFE configuration, or maybe a
  custom SPIRE/SPIRL-implemented sidecar
* The sidecar sits between your service's container and the network
* The sidecar is the thing which hits the "Workload API", gets the SVIDs,
  and uses their private keys when doing automatic mTLS stuff, so like
  your application doesn't do TLS itself, it just does raw TCP/HTTP/gRPC/whatever,
  and the sidecar hooks that stuff and wraps it in TLS


## Brushing up on how TLS works

In particular, I'm trying to figure out whether the client ever uses a private
key.
See also [OPENSSL.md](/OPENSSL.md), where I was looking for private keys in
/etc/ssl/private and not finding them, and wondering whether a fresh private
key is created for each TLS connection.

I tried looking at the output of `curl -vvv https://www.example.com` and
`openssl s_client www.example.com:443`, but didn't see a private key being
clearly used on my end.

Here's how Cloudfare describes the TLS handshake:
https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/
* The 'client hello' message
* The 'server hello' message
* Authentication
* The premaster secret
* Private key used
* Session keys created
* Client is ready
* Server is ready
* Secure symmetric encryption achieved

...so, what are those "session keys"?..
https://www.cloudflare.com/learning/ssl/what-is-a-session-key/
> A session key is any symmetric cryptographic key used to encrypt one
> communication session only.

Oh interesting, it's symmetric, not a private+public pair.
So then... this doesn't seem like it would be related to the private keys
in SPIFFE's SVIDs.


## Where to go from here

I git cloned SPIRE locally, to see if I could see how the keys were being
used; I searched in it for `svid_key`, and then for `SvidKey` since it's
mostly Golang.
I found some stuff for `SvidKey`, but it seemed to mostly just be the
SPIRE agent storing it, not using it.

TODO: read up on "signing certificates". I see them mentioned in the SVID
docs when I search for "key":

https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md
> An X.509 SVID signing certificate is one which has set keyCertSign
> in the key usage extension.
> It additionally has the cA flag set to true in the basic constraints
> extension (see section 4.1). That is to say, it is a CA certificate.

I just can't seem to find a clear description of what the key in the SVID
is for, and that makes me suspect it should be "obvious" if you're more
familiar with certificates...
