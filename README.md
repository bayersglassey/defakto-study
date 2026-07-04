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


## First step: Defakto's Docs

Went to: https://d.spirl.com/

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

Let's put some basic networking stuff in the jail, and try again:
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
