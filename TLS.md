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

IS THAT IT?.. DID WE GET IT?..

