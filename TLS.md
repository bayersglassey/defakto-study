## What is mTLS and how is SPIFFE related to it?

"mTLS" is "mutual TLS" and basically means microservices using TLS when they talk
to each other.
Why would you want this?.. because if a bad actor sneaks into your system, and
your services are speaking e.g. plain HTTP to each other, then the bad actor can
sniff your secrets out of that HTTP traffic.
Better if the bad actor can only see encrypted stuff being sent back and forth.

Now, how does TLS encryption/decryption work?.. presumably with public key
crypto, but then where do the private keys live?..
https://istio.io/latest/docs/ops/configuration/traffic-management/manage-mesh-certificates/
> ### Managing In-Mesh Certificates
> #### Sidecars
> Since sidecars manage their own certificates for in-mesh communication, the
> sidecars are responsible for managing their private keys and generated
> Certificate Signing Request (CSRs).


