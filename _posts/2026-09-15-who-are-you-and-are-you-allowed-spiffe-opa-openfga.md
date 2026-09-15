---
layout: post
title: "Who are you, and are you allowed? SPIFFE, OPA and OpenFGA"
---

Two questions sit underneath almost every distributed system, and most of the security work anyone does is really about answering them well. Who is calling me. And are they allowed to do what they are asking for. The first is authentication, the second is authorization, and the reason they are worth keeping apart is that the good answers to them look nothing alike.

I have spent a good part of the last months with three projects that answer these questions for cloud-native systems: SPIFFE for identity, OPA for policy, and OpenFGA for relationship-based authorization. They tend to be introduced one at a time, each written up as if it were the answer to everything, and that framing hides the most useful thing about them, which is that they answer different questions and slot together instead of competing.

This post is my attempt to put them in one place and say plainly what each is for.

## SPIFFE: identity you cannot forge and did not have to store

Start with the oldest bad habit in service-to-service communication: the shared secret. Service A needs to prove it is Service A when it calls Service B, so someone generates an API key or a certificate, copies it into a config file or a vault, and now the security of the whole thing rests on that string never leaking. It leaks. It gets committed to a repository, printed into a log, copied onto a laptop, or simply never rotated because rotating it means a coordinated redeploy nobody wants to schedule.

There is an even more basic problem hiding underneath, the one people call secret zero, or the bottom turtle: to read its secret out of the vault, the workload needs a credential to authenticate to the vault, and where does that one come from. You can push the problem around but you cannot make it disappear by adding one more secret.

[SPIFFE](https://spiffe.io/){:target="_blank"} (Secure Production Identity Framework For Everyone), a CNCF graduated standard, answers this differently. It gives a workload an identity based on what the workload provably is at runtime, not on a secret it managed to hold onto. Three concepts carry it:

* A **SPIFFE ID** is a URI, something like `spiffe://example.org/payments/api`. It names a workload inside a trust domain (`example.org` here). It is only a name.
* An **SVID** (SPIFFE Verifiable Identity Document) is the proof of that name. It comes as an **X.509-SVID**, a certificate carrying the SPIFFE ID in its URI SAN, which is what you use for mutual TLS, or as a **JWT-SVID**, a signed short-lived token with the SPIFFE ID as its subject, which you use as a bearer token where mTLS is not practical.
* The **Workload API** is how a workload obtains its SVID. It is a local endpoint, typically a Unix domain socket, and it is unauthenticated on purpose: there is no secret to present to it.

The trick is in that last point, and it is worth slowing down on. Something has to decide that the process calling the Workload API really is `payments/api` and deserves that SVID. In [SPIRE](https://spiffe.io/docs/latest/spire-about/){:target="_blank"}, the reference implementation, an agent runs on each node and **attests** the workload: it inspects properties the workload cannot fake (its Unix UID, the Kubernetes service account of its pod, the cloud instance identity document of the machine it runs on) and only hands over the SVID that matches. The identity is derived from the workload's position in the infrastructure, checked locally, and then the certificate is rotated automatically on a short lifetime, so a leaked one is worthless in minutes instead of months.

Once every workload has an SVID, mutual TLS between services stops being a certificate-management chore and becomes the default. Each side presents its X.509-SVID, each side validates the other against the trust bundle for the trust domain, and the SPIFFE ID in the peer's certificate tells you exactly who you are talking to. This is not hypothetical: it is the identity model underneath service meshes like Istio, where the workload identities you write into an authorization policy are SPIFFE IDs.

SPIFFE answers "who are you" and then stops. It does not decide what you are allowed to do, and that is a separate question with a separate tool.

## OPA: decisions that live outside the code

[Open Policy Agent](https://www.openpolicyagent.org/){:target="_blank"}, OPA, is also a CNCF graduated project, and it exists to get authorization out of your application. The idea it insists on is the split between the Policy Enforcement Point, the code that asks "can this happen" and then obeys the answer, and the Policy Decision Point, the thing that actually decides. Your service is the enforcement point. OPA is the decision point. The rules never live inside the service.

You write those rules in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/){:target="_blank"}, a declarative query language. OPA takes a JSON input document, evaluates the policy against it, and returns a decision. The decision is usually allow or deny, but it does not have to be: a policy can return a structured object with reasons, obligations, or a filtered view of some data. A minimal policy reads like this:

```rego
package authz

default allow := false

allow if {
    input.method == "GET"
    input.user.team == "orders"
}
```

You hand it an input like `{"method": "GET", "user": {"team": "orders"}}` and you get back a decision.

What makes this more than a fancy if-statement is where the policy lives and how it moves. Rego is kept in git, tested with its own test runner, reviewed like any other code, and distributed as bundles that OPA pulls and hot-reloads without redeploying the services that enforce it. OPA runs as a sidecar or a host-level daemon exposing a small REST API, it can be embedded as a Go library, or a policy can be compiled to WebAssembly and evaluated in-process where a network hop is not welcome.

The reach of this is wider than API calls. The same engine backs Kubernetes admission control through OPA Gatekeeper (deciding whether a manifest is even allowed into the cluster), authorizes requests at the edge through Envoy's external authorization, gates CI/CD pipelines, and checks infrastructure-as-code before it is applied. Wherever there is a yes or no that ought to be governed by a written, reviewable rule rather then by conditionals scattered through a codebase, OPA fits.

One design point I care about: a decision engine should fail closed. If OPA cannot be reached or the policy errors out, the safe outcome is to deny, not to wave the request through, and that is OPA's normal behaviour. There is a related subtlety worth knowing, which is that an undefined result (no rule matched, and no default was declared) is not the same thing as a deny, so you give every decision a `default` and never leave "no answer" to be interpreted by accident.

The way to think about OPA is that it decides from **attributes**: the properties of the request, the caller and the context, plus whatever external data you load into it. That covers an enormous amount of ground. It also has an edge, and finding that edge is what brings in the third project.

## Where SPIFFE and OPA meet

Before that edge, the obvious pairing. SPIFFE tells you who the caller is, in a way the caller cannot fake. OPA decides what that identity may do, from rules you keep outside the code. Put them in a line and you have the backbone of a zero-trust posture: no service is trusted because of the network it sits on, every call carries a verified identity, and every decision is made against an explicit policy.

The discipline that matters here is keeping the wire between the two clean. The identity SPIFFE established is trustworthy precisely because it was verified, so it should reach the policy as verified input, not as some value a caller could have set on the side and labelled "user". Authenticate first, then authorize the result of that authentication. It sounds obvious written down, and it is exactly the step people skip, feeding the policy engine a field the request itself supplied and treating it as identity.

## OpenFGA: when authorization is really about relationships

Here is the edge. OPA is at its best when a decision follows from attributes you can put in front of it. Some decisions are not really about attributes at all, they are about **relationships**, and the classic example is document sharing. Can Alice open this one document? She can if she is its owner, or an editor someone added, or a member of a group the document was shared with, or a member of a team that owns the parent folder, which happens to have been shared with another team she belongs to. None of that is a property of the request. It is a graph, and the answer is a question about reachability in that graph.

You can encode this in Rego, but you end up either shipping the entire relationship graph into every input document or teaching the policy to go and fetch the pieces it needs, and neither plays to Rego's strengths. What you actually want is a system that stores the relationships and is built to walk them quickly.

That system is [OpenFGA](https://openfga.dev/){:target="_blank"}, and the idea comes straight from Google's [Zanzibar](https://research.google/pubs/pub48190/){:target="_blank"} paper, the design behind permissions in Google Docs, Drive, Calendar and the rest. OpenFGA is a relationship-based access control (ReBAC) engine. You define an authorization model, the types in your domain (`user`, `team`, `folder`, `document`) and the relations between them (`owner`, `editor`, `viewer`, `parent`, `member`), including how relations imply one another (an editor is also a viewer, a viewer of a folder is a viewer of the documents under it). Then you write **relationship tuples** as plain facts: `user:alice is editor of document:q3-plan`, `folder:budgets is parent of document:q3-plan`. And you ask **check** queries: is `user:alice` a `viewer` of `document:q3-plan`? OpenFGA walks the model and the tuples to a yes or a no, and it can also list every object a user can reach or every user who can reach an object.

The relationships are the data OpenFGA owns and indexes, which is the part a policy engine deliberately does not want to own. It runs as a service with an API and SDKs, it centralizes permission logic the way OPA centralizes policy logic, and it is engineered for the scale and latency that per-object checks demand when a single page load can fire a handful of them. OpenFGA came out of Auth0/Okta, was donated to the CNCF, and [moved from Sandbox to Incubating in October 2025](https://www.cncf.io/blog/2025/11/11/openfga-becomes-a-cncf-incubating-project/){:target="_blank"}, with Grafana, Docker and Canonical among its production users. It is not the only implementation of the Zanzibar idea (SpiceDB from Authzed is another well-known one), but it is the one in the CNCF.

It is tempting to file this as OPA versus OpenFGA, and that is the wrong frame. They answer different halves of authorization. OPA is policy over attributes and context. OpenFGA is fine-grained permission over relationships. Real systems use both, often with the policy layer calling the relationship layer for the "does this user actually have access to this specific object" part of a larger decision.

## One stack, two questions

Step back and the shape is clean. There are two questions, who are you and are you allowed, and there are purpose-built, cloud-native, CNCF-hosted answers to each. SPIFFE gives every workload an identity it did not have to store and cannot forge. OPA moves the yes or no of policy out of your services and into rules you can version and test. OpenFGA handles the slice of authorization that is really a question about relationships between users and the things they act on.

The thread running through all three is the same instinct: take a cross-cutting security concern that used to be smeared across every service as static secrets and hand-written conditionals, and give it to a system built to do that one job well, so it can be reasoned about, changed and audited in one place. You authenticate with the first, and you authorize with the other two. It is not the whole of security, but its a large and too often badly handled part of it, and these three are the best answers I know of at the moment.

One last note, for anyone who works with Apache Camel. This is not theory for me: I recently brought the first two of these into Camel itself, as the `camel-spiffe` and `camel-opa` components (both Preview in Camel 4.23), so a route can fetch a workload identity or enforce an OPA decision without hand-rolling either one. How that fits into a route is a subject for its own post, but the Jira issues below are the place to start if you want the detail.

## References

- [SPIFFE](https://spiffe.io/){:target="_blank"} - SPIFFE
- [SPIRE, the SPIFFE Runtime Environment](https://spiffe.io/docs/latest/spire-about/){:target="_blank"} - SPIFFE
- [SPIFFE concepts: SPIFFE ID, SVID, Workload API](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/){:target="_blank"} - SPIFFE
- [SPIFFE and SPIRE graduate from the CNCF Incubator](https://www.cncf.io/announcements/2022/09/20/spiffe-and-spire-projects-graduate-from-cloud-native-computing-foundation-incubator/){:target="_blank"} - CNCF
- [Open Policy Agent](https://www.openpolicyagent.org/){:target="_blank"} - OPA
- [The Rego policy language](https://www.openpolicyagent.org/docs/latest/policy-language/){:target="_blank"} - OPA
- [CNCF announces Open Policy Agent graduation](https://www.cncf.io/announcements/2021/02/04/cloud-native-computing-foundation-announces-open-policy-agent-graduation/){:target="_blank"} - CNCF
- [OpenFGA](https://openfga.dev/){:target="_blank"} - OpenFGA
- [OpenFGA becomes a CNCF incubating project](https://www.cncf.io/blog/2025/11/11/openfga-becomes-a-cncf-incubating-project/){:target="_blank"} - CNCF
- [Zanzibar: Google's Consistent, Global Authorization System](https://research.google/pubs/pub48190/){:target="_blank"} - Google Research
- [SpiceDB](https://authzed.com/spicedb){:target="_blank"} - Authzed
- [CAMEL-23305: the camel-spiffe component](https://issues.apache.org/jira/browse/CAMEL-23305){:target="_blank"} - Apache Camel
- [CAMEL-24634: the camel-opa component](https://issues.apache.org/jira/browse/CAMEL-24634){:target="_blank"} - Apache Camel
- [camel-spiffe component source](https://github.com/apache/camel/tree/main/components/camel-spiffe){:target="_blank"} - Apache Camel
- [camel-opa component source](https://github.com/apache/camel/tree/main/components/camel-opa){:target="_blank"} - Apache Camel
