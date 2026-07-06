## Questions I have about Defakto

* What is the relationship between SPIRE and SPIRL?.. is SPIRL a closed-source
  fork of SPIRE, or a totally new SPIFFE implementation, or something else?..
  Also, how does Defakto compete with SPIRE, which is open-source?..

* Does Defakto see Aembit.io as a competitor?

...ah, this article doesn't quite answer my questions, but sort of alludes
to them:
https://securityboulevard.com/2026/05/everyone-wants-spiffe-almost-no-one-can-afford-to-build-it-right/

E.g.
> Defakto (formerly SPIRL), a company founded by the engineer who ran one of
> the world’s largest SPIFFE deployments, is unusually candid in their
> documentation: small-scale SPIRE deployments typically take 6 to 12 months,
> and more complex deployments can easily require 12 to 24 months to complete,
> figures that assume a core team of experts to build the solution.
> This isn’t a critique from a competitor; it’s the measured admission of
> people who built it and went commercial because of it.
> The fact that they raised $30.75M in Series B funding to wrap SPIFFE/SPIRE
> in a commercial product is itself the strongest argument against the “just
> use the open source” framing.

Ah, and at the bottom, that article links to others, which I believe actually
do answer my questions.

In this one (published by Defakto), the argument is that SPIRE is great, very
flexible, *but* you need to set it up yourself, whereas Defakto is opinionated
and works out of the box:
https://www.defakto.security/blog/simplifying-spiffe-accessible-workload-identity-with-spirl/

And here's one, published by Aembit, comparing itself to SPIFFE/SPIRE:
https://aembit.io/blog/aembit-vs-spiffe-spire/
> SPIRE concentrates on the Identity layer.
> It requires integration with other tools to solve problems higher up the stack.
> For example, you may integrate SPIRE with Envoy for authentication, which you
> then integrate with something like Kuma to build a service mesh.
> Aembit takes care of the customer needs through the whole pyramid: Identity,
> Authentication, Access Management, and Access Analytics.
> Aembit is a complete, vendor-supported, end-to-end solution to Workload
> Identity and Access Management problems.

It doesn't mention Defakto there, so I would be curious to hear how Defakto
positions itself... as simply a SPIFFE implementation, perhaps faster to
deploy than SPIRE, but ultimately still only providing identity?..

See also:
https://www.linkedin.com/pulse/oauth2-vs-spiffe-jialin-li-o2bpc/
> The Delegation Problem: SPIFFE does not have native delegation semantics.
> There's no "act" claim. If you need to track "who acted on behalf of whom," you must:
> * Manually propagate context in HTTP headers (not cryptographically protected)
> * Use a separate context token alongside SPIFFE
> * Implement custom solutions

And:
https://www.uber.com/us/en/blog/solving-the-agent-identity-crisis/?uclick_id=a4bc8b8d-c91e-498c-b187-a4ed4c15d0a5
> ### Problem 1: Current Identity Model Doesn’t Describe Agency
> ...etc...
> ### Problem 2: Original Provenance Isn’t Effectively Carried Forward Across Agents to Systems
> Execution context (originating user, intermediate agents) is dropped across
> agent hops.
> This leads to incomplete audits across the system and limits our ability
> to consistently leverage the fine-grained access policies already configured
> by downstream systems.
> In the absence of complete audit trails, incident response would require
> stitching partial audit logs across systems together.
