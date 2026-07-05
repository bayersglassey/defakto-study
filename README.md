# Defakto study

This repo is a place for me to record what I did while studying Defakto
Security, a software company I applied to last week.
https://www.defakto.security/

It sounds like they're essentially implementing the
[SPIFFE protocol](https://spiffe.io/) and then selling various products on
top of that.
What is SPIFFE at a high level?.. its site has some bullet points:
* Secure microservices communication automatically with Envoy, X.509 PKI, or JWT
* Authenticate securely to common databases or platforms without passwords or API keys
* Build, bridge, and extend service mesh across organizations without sharing keys
* Cross-service authentication for zero trust security model

I have some knowledge of OAuth2, OpenID Connect, certificates, etc, but I've
never needed to have a really deep understanding of that stuff.
My goal here is to review all that, plus learn SPIFFE, plus read Defakto's
docs, and tie it all together.
For instance, I'd like to be able to confidently describe how SPIFFE is
different than OAuth2, and have a good understanding of at least one use
case, e.g. one interaction between two microservices using Defakto's
platform.

**NOTE ON LLM USAGE**: I didn't use an LLM for any of this.
Maybe it makes me a dinosaur, but the best way I know of to build a mental
model of something is to jump around between reading documentation,
recording my thoughts, doing little experiments in a shell or REPL, and
factoring reusable tools out of the experiments.


## Defakto's "Quick Start", spirlctl

Went to Defakto's docs: https://d.spirl.com/

Clicked on "Quick Start", was asked to download `spirlctl`:
https://d.spirl.com/mint/quick-start/download-spirlctl
```shell
$ curl -o spirlctl-v0.34.0.tar.gz \
    https://spirl-releases.s3.us-west-2.amazonaws.com/spirlctl/v0.34.0/spirlctl-v0.34.0-linux-amd64.tar.gz \
$ tar -zxvf spirlctl-v0.34.0.tar.gz
```

That gives me a single `spirlctl` binary, with which I'm supposed to `spirlctl login`.
It's a self-contained Golang binary, no shared library dependencies:
```shell
$ ldd spirlctl 
	not a dynamic executable
$ strings spirlctl | ack golang | head -n 4
golang.org/x/term
golang.org/x/xerrors
golang.org/x/sys/unix
google.golang.org/grpc
```

Let's try to be somewhat safe, and run it in a chroot jail:
```shell
$ mkdir jail
$ mv spirlctl jail/
$ cd jail
$ sudo chroot . ./spirlctl --version
spirlctl version 0.34.0
```

Cool. What if we login?
```shell
$ sudo chroot . ./spirlctl login
Error: login to SPIRL failed: failed to open login stream:
rpc error: code = Unavailable desc = dns: A record lookup error:
lookup api.spirl.com on [::1]:53: read udp [::1]:58018->[::1]:53:
read: connection refused
```

Let's put some basic networking stuff in the jail, and try again.
(NOTE: for some discussion of these files, see [OPENSSL.md](/OPENSSL.md).)
```shell
$ for f in /etc/resolv.conf /etc/ssl/certs/ca-certificates.crt; do cp $f ./$f; done
$ sudo chroot . ./spirlctl login
Login URL: https://auth.api.spirl.com/auth/cli/start?session_token=<REDACTED>
Pairing Code:  <REDACTED>
Failed to open the web browser to the login URL.
Please navigate to the login URL to complete the login process
```

Looks a lot like OAuth2!
I went to that URL, and tried to log in using my Google account. Denied!
> ### Authentication Failed
> #### Reason: no user account available
> You may close this page

Back in the terminal, `spirlctl` says:
```
Error: login to SPIRL failed: login failed
```

That all makes sense, I haven't paid for Defakto, no reason for them to let me in.
Is there a free trial?.. looks like no, closest thing would be to "request a demo".

I decided to try and poke at `spirlctl` using `strace` and `tcpdump`, and
learned a bit about what calls its making.
See [SPIRLCTL.md](/SPIRLCTL.md) for details.


## Digging further into the documentation, SPIFFE, SPIRE

Okay, following along with the "Quick Start", next is creating a trust domain:
```
./spirlctl trust-domain create example.com --managed
```

I recall from reading the SPIFFE docs yesterday that a trust domain shows up
in SPIFFE IDs as the top-level thing, like the "host" part of an URL.
https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#spiffe-id

> For example, `spiffe://acme.com/billing/payments` is a valid SPIFFE ID.

> SPIFFE IDs are a Uniform Resource Identifier (URI) which takes the following
> format: `spiffe://trust domain/workload identifier`

> The workload identifier uniquely identifies a specific workload within a
> trust domain.

Cool, so the Quick Start wants us to create `example.com` as a trust domain.
But... does that make sense? Are trust domains a global namespace?
Presumably not, or everybody following the Quick Start would be stepping
on each others' toes.
So, presumably trust domains exist inside a given... SPIFFE namespace.
Or whatever.
So like, in this case we're supposed to have "logged in" with `spirlctl`.
Presumably all our trust domains live within that... login.
That... organization?

Actually, let's go back to the docs:
https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/#trust-domain
> The trust domain corresponds to the trust root of a system.
> A trust domain could represent an individual, organization, environment
> or department running their own independent SPIFFE infrastructure.

No answers here. I guess `spirlctl login` maybe doesn't correspond with
a SPIFFE concept, and long story short, SPIFFE trust domains aren't a
global namespace.

Actually, the Quick Start shows this example:
```
$ ./spirlctl trust-domain list
Name         ID             State      SPIRL-hosted  Clusters  Federations  Agents
example.com  td-xxxxxxxxxx  available  true          0/0       0/0          0
```

...and I find "SPIRL-hosted" interesting.
Sounds to me like SPIRL is Defakto's custom implementation of SPIRE, the
SPIFFE Runtime?..
https://spiffe.io/docs/latest/spire-about/

It seems like Defakto is trying to get rid of the name SPIRL, though.
I see it cached in web search results, like when I search DuckDuckGo
for "defakto spirl", the first result is:
> SPIRL - Secure Access for Non-Human Identity Reimagined | Defakto

...but when I click that link, the page title is just:
> Secure Access for Non-Human Identity Reimagined | Defakto

Ooh but looking at the SPIRE docs, I think maybe we can get an understanding
of what's so special about SPIFFE from this page?
https://spiffe.io/docs/latest/spire-about/comparisons/

Hmmmm my takeaway from that page is:
* Apparently SPIRE thinks its "attestation process" is one of the coolest
  things about it?
* SPIFFE and SPIRE apparently provide authentication, but not authorization.
  So often they are paired with something else which does authorization, using
  the SVIDs produced by SPIFFE and SPIRE
* I'd like to see what the SPIFFE Workload API is

Regarding my first point there, here are some quotes from that page:
> SPIRE’s attestation policies provide a flexible and powerful solution for
> secure introduction to secrets managers.

> Most identity providers rely on a pre-existing identity document or secret
> to identify a workload, such an API key and secret or keytab.
> What distinguishes SPIRE from other identity providers is the attestation
> process by which it identifies the workload in the first place, before
> issuing it an identity document.
> This significantly improves security since no long-lived static credential
> needs to be co-deployed with the workload itself.

Regarding the Workload API,
https://spiffe.io/docs/latest/spiffe-specs/spiffe_workload_api/

I see. Sounds like if I'm a microservice, and I want to use SPIFFE, I can
look at the `SPIFFE_ENDPOINT_SOCKET` env var, which will be e.g.
`unix:///path/to/endpoint.sock` or `tcp://127.0.0.1:8000`.

I should speak gRPC to that endpoint, which will serve up the Workload API,
which itself has multiple endpoints, conceptually grouped into "profiles".
I don't think gRPC has any concept of "profiles", the idea is just that if
you don't want to support JWT, then your Workload API implementation doesn't
need to implement the endpoints in the JWT "profile", etc.

Anyway, the X.509 "profile" consists mainly of this endpoint:
```
    // Fetch X.509-SVIDs for all SPIFFE identities the workload is entitled to,
    // as well as related information like trust bundles and CRLs. As this
    // information changes, subsequent messages will be streamed from the
    // server.
    rpc FetchX509SVID(X509SVIDRequest) returns (stream X509SVIDResponse);
```

...while the JWT "profile" consists mainly of this endpoint:
```
    // Fetch JWT-SVIDs for all SPIFFE identities the workload is entitled to,
    // for the requested audience. If an optional SPIFFE ID is requested, only
    // the JWT-SVID for that SPIFFE ID is returned.
    rpc FetchJWTSVID(JWTSVIDRequest) returns (JWTSVIDResponse);

    // Fetches the JWT bundles, formatted as JWKS documents, keyed by the
    // SPIFFE ID of the trust domain. As this information changes, subsequent
    // messages will be streamed from the server.
    rpc FetchJWTBundles(JWTBundlesRequest) returns (stream JWTBundlesResponse);
```

...and there's also an experimental WIT "profile" dabbling in some newfangled
WIMSE credential system. I'll ignore that.

By the way, the structure for an X.509 SVID is:
```
    // Required. The SPIFFE ID of the SVID in this entry
    string spiffe_id = 1;

    // Required. ASN.1 DER encoded (not PEM) certificate chain. MAY include
    // intermediates, the leaf certificate (or SVID itself) MUST come first.
    bytes x509_svid = 2;

    // Required. ASN.1 DER encoded (not PEM) PKCS#8 private key. MUST be unencrypted.
    bytes x509_svid_key = 3;

    // Required. ASN.1 DER encoded (not PEM) X.509 bundle for the trust domain.
    bytes bundle = 4;
```

...while the structure for a JWT SVID is:
```
    // Required. The SPIFFE ID of the JWT-SVID.
    string spiffe_id = 1;

    // Required. Encoded JWT using JWS Compact Serialization.
    string svid = 2;
```

...and then the bundles are streamed separately:
```
    // Required. JWK encoded JWT bundles, keyed by the SPIFFE ID of the trust
    // domain.
    map<string, bytes> bundles = 1;
```

Okay. So remember, we are a microservice, and we want a SVID, so the idea
is that we find (via e.g. the env var) this gRPC thing and connect and hit
one of the endpoints and get baaaaack... a stream of SVIDs?..
Or like, a stream of SVIDs and/or their bundles?..
And "as this information changes, subsequent messages will be streamed from
the server".
So the idea there is that you don't need to poll, 'cos it's gRPC streaming
endpoints, so fancy.
You connect once, and then the SPIFFE infrastructure will decide to let you
know over your open TCP connection whenever there are updates about your
SVIDs.
And what do you do with these SVIDs?.. I guess you use their certificate
bundles wheeeeen you make TLS-secured API calls?..
If so, how -- byyyyy adding certificates in the place the TLS libraries
expect it, e.g. libssl?
Or do we generate tokens from them, and use them in API calls between
services?..

Now, remember earlier, we said SPIRE seems really proud of its "attestation"
concept:
> What distinguishes SPIRE from other identity providers is the attestation
> process by which it identifies the workload in the first place, before
> issuing it an identity document.

And I believe we can see it described here:
https://spiffe.io/docs/latest/spiffe-specs/spiffe_workload_endpoint/#5-authentication
> In place of direct client authentication, implementers SHOULD perform
> out-of-band authenticity checks.
> This may include techniques such as kernel introspection or orchestrator
> interrogation.
> As an example, it is possible to understand which process is calling the
> API by examining kernel socket state.
> Another approach is to allow an orchestrator to place a Unix Domain Socket
> into a particular container, informing the SPIFFE Workload Endpoint
> implementation of the container’s properties/identity.
> This information can then be used as an authentication mechanism.

And in particular:
> It should be noted that while the method(s) by which this is done is
> implementation-specific, the chosen method(s) MUST NOT require the workload
> to actively participate.

Hmmmm.
Oh, I googled "spiffe attestation", and found a paper!
https://arxiv.org/html/2504.14760v1
> Establishing Workload Identity for Zero Trust CI/CD:
> From Secrets to SPIFFE-Based Authentication
