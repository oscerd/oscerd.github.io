---
layout: post
title: "Java vulnerabilities of the week: 12 to 18 August 2026"
---

Second half of the catch-up. [The previous post](/2026/08/12/java-vulnerabilities-of-the-week-5-11-august-2026.html) covered 5 to 11 August, this one covers 12 to 18. Same format: the five things worth knowing, then two taken apart, because the mechanism is the part you can generalise from.

This week is lighter in volume and unusually consistent in shape. Almost every item is a security control that exists in the code, is documented, and in some cases is explicitly configured, and that for one reason or another is not in effect on the path you are actually using. A hostname verification policy that does not reach the layer that would enforce it. A trust manager, offered as a convenience, that accepts anything. A `Vary` header replaced rather then extended. A Digest `uri` parameter never compared to the request. An `HttpOnly` flag left off a cookie that is also doing duty as a CSRF token.

That gap, between a control being present and a control being in effect, is hard to see. It does not show up in code review, because the calling code is correct. It does not show up in testing, because a check that never runs and a check that passes look identical on every request that is not an attack.

## The week in five

- **Apache HttpComponents Client: 2 CVEs, both Important, 13 August.** `CVE-2026-71290`, reported by n0mi1k: `HostnameVerificationPolicy#BUILTIN` has no effect on the async version of HttpClient, so someone able to intercept traffic can present a valid certificate for a different domain. Affects `httpclient5` 5.4-alpha through 5.6.3, fixed in 5.6.4. The classic transport is unaffected. `CVE-2026-64607`, reported by Yu Bao of the PayPal Cyber Security Team, is the mirror image: the classic transport does not return a connection to the pool when it meets an invalid or unsupported `Content-Encoding` response header, so a server answering with an unknown encoding drains the pool a request at a time. Affects 5.0-alpha1 through 5.6.2; the advisory does not name a fixed version, and the 5.6.3 release notes carry the matching change. Deep dive below.

- **RabbitMQ Java client: 6 CVEs, 18 August.** `CVE-2026-63336`: `ConnectionFactory.useSslProtocol()` with no arguments installs `TrustEverythingTrustManager`, which accepts any certificate including a null chain, and hostname verification is off until you call `enableHostnameVerification()`. `CVE-2026-63337`, High: the JSON-RPC tools call `Class.forName(javaReturnType)` with `initialize=true` on class names taken from an untrusted `system.describe` response, so static initializers of attacker-named classes run in the client JVM. Then three parser issues that echo [the Qpid batch from a fortnight ago](/2026/08/05/java-vulnerabilities-of-the-week-29-july-4-august-2026.html): `CVE-2026-69219` (oversized `LongString` length, unchecked allocation), `CVE-2026-69220` (unbounded recursive table and array nesting, `StackOverflowError`) and `CVE-2026-61634` (frames larger than the negotiated `frame_max` accepted). `CVE-2026-63335` covers a malformed body frame. Fixes are spread across 5.31.0, 5.33.0 and 5.33.1, and every advisory ships with a reproducer.

- **`CVE-2026-66256`, Apache Shindig, Important, 13 August.** Remote code execution via XStream deserialization in the OpenSocial REST API, reported by Daryle Bourque and Noah King of Horizon3.ai. It affects `shindig-common` and `shindig-social-api` at all versions, and will continue to, because Shindig is retired and the advisory says clearly that no fix is planned. Deep dive below.

- **Netty: 2 CVEs, 17 August.** `CVE-2026-59903` comes down to one method call: `CorsHandler#setVaryHeader` does `response.headers().set(VARY, ORIGIN)`, and `set` replaces. An application that put `Vary: Authorization` on a response so a CDN caches it per user loses that, and the cache can then serve one user's authenticated response to another. `CVE-2026-59902` is memory exhaustion in `SctpMessageCompletionHandler`, which caps the number of buffered messages and fragments but not their total size, so the defaults work out at roughly a gigabyte per connection. It is a follow-up to `CVE-2026-46340`, which added the count limits. Both fixed in 4.1.137.Final and 4.2.17.Final, the second credited to @violetagg.

- **`CVE-2026-53660`, OpenAM Community Edition, High, 14 August.** The default configuration ships the `iPlanetDirectoryPro` SSO cookie with `HttpOnly=false` and no `SameSite`. On its own that is a weak default. What makes it this week's item is the second half: the same cookie is also used as the CSRF token in the OAuth and OIDC consent flows. A CSRF defence whose premise is that an attacker cannot read the token is built on a value that JavaScript can read, so any same-origin XSS both takes the session and completes an attacker-driven consent grant. Affects through 16.0.6, fixed in 16.1.1.

Honourable mention to **http4k**, 17 August, with three. `CVE-2026-54148`, CVSS 8.1: `DigestAuthProvider.verify` did not check the `uri` parameter in the client's `Authorization: Digest` response against the actual request URL, so a captured response replays against any other URL in the same realm. Per-request-URL binding is what Digest has instead of a session, and the advisory notes it had been missing since the provider was added in 2021, which is the kind of detail projects do not have to disclose and should. `CVE-2026-54147` is the same plus an ignored algorithm setting, and `CVE-2026-53659` is unbounded gzip decompression in `ServerFilters.GZip`. Fixed in 6.50.0.0 on the community line. And to **`CVE-2026-53752` in docx4j**, same day: `PropertyResolver` walks the OpenXML `w:basedOn` style inheritance chain without cycle detection, so a document where style A is based on B and B on A produces a `StackOverflowError`. Valid OOXML, nothing exotic, fixed in 11.5.14.

## Apache HttpComponents Client: a policy that did not reach the layer enforcing it

HttpClient 5.4 introduced `HostnameVerificationPolicy`, an enum whose javadoc is short enough to quote whole. `CLIENT`: "Hostname verification is executed by HttpClient post TLS handshake." `BUILTIN`: "Hostname verification is delegated to the JSSE provider, usually executed during the TLS handshake." `BOTH`: both of those.

`BUILTIN` exists because doing it in JSSE is the better option. Java 7 added `SSLParameters.setEndpointIdentificationAlgorithm(String)`, and setting it to `"HTTPS"` makes the provider run RFC 2818 endpoint identification during the handshake itself, so the connection fails before any application data moves rather then after a handshake has completed with an unidentified server. Earlier is better, and it is the JDK's implementation rather than a second one living in an HTTP library. That is a good design decision, and it is worth saying so before describing what went wrong with it.

`CVE-2026-71290` reports that `BUILTIN` has no effect on the async version of HttpClient. The 5.6.4 release notes describe the fix as correcting "application of SSL parameters in the async TLS upgrade method", which fills in the rest. `BUILTIN` does one thing: it sets an endpoint identification algorithm on the `SSLParameters` and lets JSSE do the work. On the async transport those parameters were not applied during the TLS upgrade, so the instruction never arrived.

The consequence follows from the delegation. With `CLIENT`, HttpClient runs the check itself, so a failure elsewhere still leaves a verification in place. With `BUILTIN`, HttpClient has deliberately stepped back on the understanding that JSSE has it covered. When the parameters do not reach JSSE, nothing verifies, and the handshake succeeds cleanly against whoever is holding a valid certificate for another domain. The advisory's own subject line notes this is reachable in a default configuration.

The paired CVE is worth a sentence for the symmetry: `CVE-2026-64607` affects only classic, `CVE-2026-71290` only async. Two transports behind one API, and the unusual path is where they diverge.

### Why I find it interesting

Because the same shape turned up three times in two weeks, across unrelated projects, and its not the shape most reviews look for.

RabbitMQ's `CVE-2026-63336` is the same failure with the polarity reversed. There the control is not skipped, it is replaced: `useSslProtocol()` with no arguments installs `TrustEverythingTrustManager`, whose `checkServerTrusted` accepts any chain including null. That class is a development convenience of the kind most networking libraries grew at some point, and the finding here is really about it being what the no-argument overload gives you. And last week Apache Ranger fixed `CVE-2026-65942`, clients accepting certificates issued for other hostnames, which makes three in fifteen days.

The bug class does not have a settled name, so I will describe it: **a control that fails open and reports nothing.** The distinguishing property is not that the check is wrong, its that the absence of the check is indistinguishable from the check succeeding. A hostname verifier that runs and passes and one that never runs produce identical behaviour on every non-attack request, so tests pass, staging is fine and monitoring is green until somebody is genuinely in the middle. That is what makes it different from most of what I write about here: a deserialization gadget needs an attacker with a payload, an unbounded allocation needs an attacker with a large number, and this needs nothing at all.

The practical response, and this is a gap in my own testing as much as anyone's, is that transport security settings deserve a negative test. Not "does my client reach the real server", which everyone has, but: point the client at a server whose certificate is for a name you did not dial, and assert the connection fails. It runs in CI and it is the only thing that distinguishes the two states above. Worth pointing it at each transport separately, since the HttpClient issue affected async and not classic, and a test exercising one would have said nothing about the other.

## Apache Shindig: a report against a project that will not release again

Shindig was the Apache implementation of OpenSocial, the gadget and social API container behind a good number of enterprise portals. It was retired to the Attic in October 2015, with the move completed in January 2016.

On 13 August, Horizon3.ai reported remote code execution through XStream deserialization in the OpenSocial REST API: anyone who can reach the REST API can execute code on the server. The advisory carries the `** UNSUPPORTED WHEN ASSIGNED **` marker, lists affected versions as "all versions", and says:

> As this project is retired, we do not plan to release a version that fixes this issue. Users are recommended to find an alternative or restrict access to the instance to trusted users.

That is the right thing to publish, and it costs something to publish it. Nobody is being paid to shepherd a CVE through for software that was archived a decade ago, and the easy path is silence. Doing it anyway means anyone still running Shindig finds out.

XStream is worth a note, because this is downstream of an old decision rather than a recent one. XStream maps XML onto object graphs, and for most of its life it guarded that with a denylist of known-dangerous runtime classes. XStream 1.4.18 changed the default to an allowlist in 2021, and the project's own security page is direct about why, saying the denylist approach "has failed". Shindig was already archived by then. The security posture is frozen at whatever the state of the art was when the project stopped, which is true of every archived project and not a criticism of any of them.

### Why I find it interesting

Because of what an unfixable finding does to the machinery we have built around CVEs.

Almost everything downstream of a CVE assumes a fix exists. A scanner's output is a list of things to upgrade. A release policy says no High or Critical findings. An update bot opens a pull request that changes a version. None of that has a state for "there is no version to change to, and there will not be one". The finding is real, correctly published and machine readable, and the only valid remediation is architectural: remove the component, or put a boundary in front of it.

In practice these become permanent red entries, and permanent red entries get suppressed. Someone adds an exception, writes "archived project, accepted risk" on a ticket, and the suppression outlives the person who wrote it. This is the failure mode I was getting at writing about [the CVE stigma](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html): when a CVE count is treated as a score to drive to zero rather then information to act on, the items you cannot fix are the first to fall out of the process.

The lesson is not about XStream or deserialization. It is about the unmaintained dependency, which is not a code property at all but a property of a project's calendar, and it is the thing a version number cannot tell you. An SBOM lists `shindig-common 2.5.2`; it does not list "archived in 2015". So the audit worth doing is not "which of my dependencies have known CVEs" but "which of them would not get a fix if one were needed on Monday". Last release date, commit activity, how many people hold commit rights, whether the project is in an Attic. Very little in the standard toolchain reports on that, and it is not hard to check by hand for the twenty things that actually matter in a build.

## What I take away from this week

The first thread is the one both dives share. Nearly every item here is a control present in the code and absent in effect: configured but not applied (HttpClient), replaced by a convenience default (RabbitMQ), overwritten downstream (Netty's `Vary`), never bound to the thing it was meant to bind to (http4k's Digest `uri`), or switched off in the shipped configuration and then depended on elsewhere (OpenAM's cookie). In none of those does reading the calling code tell you anything is wrong, because the calling code is fine.

The second follows from it: the negative test is cheap and under-used, mine included. We test that authentication succeeds with the right credentials far more than we test that it fails with the wrong ones, and that TLS connects far more than that it refuses. Pick the security properties you would least like to be wrong about, write the test that proves each can say no, and run it against every transport that claims to implement it.

On arrival: Netty and `httpclient5` are transitive under most of the Java world, so they come with a Camel, Spring Boot or Quarkus BOM bump. The RabbitMQ client is usually direct and a one line change, and since `useSslProtocol()` has behaved this way for a long time it is worth grepping for that call whichever version you are on. OpenAM and http4k are direct. Shindig does not arrive at all, which is the point of that section.

If I were picking one thing to do this week it would not be a version bump. It would be the negative TLS test, on whichever client talks to the system you would least like to lose. Two of the items above would have shown up the first time it ran.

Back on schedule from here. See you next week.

## References

- [CVE-2026-71290: Apache HttpComponents Client, TLS hostname verification silently disabled on the async transport](https://www.openwall.com/lists/oss-security/2026/08/13/6){:target="_blank"} - oss-security
- [CVE-2026-64607: Apache HttpComponents Client, connection leak on Content-Encoding decode error leads to pool exhaustion DoS](https://www.openwall.com/lists/oss-security/2026/08/13/5){:target="_blank"} - oss-security
- [HttpClient 5.6.x release notes](https://downloads.apache.org/httpcomponents/httpclient/RELEASE_NOTES-5.6.x.txt){:target="_blank"} - Apache HttpComponents
- [HostnameVerificationPolicy javadoc](https://hc.apache.org/httpcomponents-client-5.6.x/5.6/httpclient5/apidocs/org/apache/hc/client5/http/ssl/HostnameVerificationPolicy.html){:target="_blank"} - Apache HttpComponents
- [HTTPCLIENT-2151: optionally use JSSE inbuilt endpoint identification](https://issues.apache.org/jira/browse/HTTPCLIENT-2151){:target="_blank"} - ASF Jira
- [CVE-2026-66256: Apache Shindig, remote code execution via XStream deserialization](https://www.openwall.com/lists/oss-security/2026/08/13/7){:target="_blank"} - oss-security
- [Apache Shindig in the Attic](https://attic.apache.org/projects/shindig.html){:target="_blank"} - Apache Attic
- [XStream Security Framework](https://x-stream.github.io/security.html){:target="_blank"} - XStream
- [GHSA-5m9f-rphj-c435: RabbitMQ Java client, TrustEverythingTrustManager used by default in useSslProtocol()](https://github.com/advisories/GHSA-5m9f-rphj-c435){:target="_blank"} - GitHub Advisory Database
- [GHSA-6g32-pxv4-2wfj: RabbitMQ Java client, unvalidated Class.forName in JSON-RPC ProcedureDescription](https://github.com/advisories/GHSA-6g32-pxv4-2wfj){:target="_blank"} - GitHub Advisory Database
- [GHSA-68mj-5wr7-6fgg: RabbitMQ Java client ValueReader, oversized LongString/bytes length triggers OOM](https://github.com/advisories/GHSA-68mj-5wr7-6fgg){:target="_blank"} - GitHub Advisory Database
- [GHSA-93j5-89vc-pph4: RabbitMQ Java client ValueReader, unbounded recursive table/array nesting](https://github.com/advisories/GHSA-93j5-89vc-pph4){:target="_blank"} - GitHub Advisory Database
- [GHSA-5xwg-cfvj-gff5: RabbitMQ Java client accepts broker frames larger than the negotiated AMQP frame_max](https://github.com/advisories/GHSA-5xwg-cfvj-gff5){:target="_blank"} - GitHub Advisory Database
- [GHSA-qx7j-jv8m-fppr: RabbitMQ Java client, malformed body frame triggers raw command assembler exception](https://github.com/advisories/GHSA-qx7j-jv8m-fppr){:target="_blank"} - GitHub Advisory Database
- [GHSA-8c42-7qj2-3j46: Netty, cache poisoning and information disclosure via CORS Vary header overwrite](https://github.com/advisories/GHSA-8c42-7qj2-3j46){:target="_blank"} - GitHub Advisory Database
- [GHSA-2qj4-mmr9-4v2f: Netty, memory exhaustion in SctpMessageCompletionHandler](https://github.com/advisories/GHSA-2qj4-mmr9-4v2f){:target="_blank"} - GitHub Advisory Database
- [GHSA-fpmh-vx4h-xc33: OpenAM insecure SSO cookie initialization](https://github.com/advisories/GHSA-fpmh-vx4h-xc33){:target="_blank"} - GitHub Advisory Database
- [GHSA-p28p-j94q-pg32: http4k, DigestAuthProvider.verify did not bind to request URI](https://github.com/advisories/GHSA-p28p-j94q-pg32){:target="_blank"} - GitHub Advisory Database
- [GHSA-gc95-3vw8-vg43: docx4j, stack overflow via cyclic w:basedOn style chain](https://github.com/advisories/GHSA-gc95-3vw8-vg43){:target="_blank"} - GitHub Advisory Database
- [oss-security daily archive](https://www.openwall.com/lists/oss-security/2026/08/13/){:target="_blank"} - Openwall
