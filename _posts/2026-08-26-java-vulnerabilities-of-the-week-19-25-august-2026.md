---
layout: post
title: "Java vulnerabilities of the week: 19 to 25 August 2026"
---

[The previous post](/2026/08/19/java-vulnerabilities-of-the-week-12-18-august-2026.html) covered 12 to 18 August, this one covers 19 to 25. The format is unchanged: five things worth knowing, then two of them taken apart properly, because a CVE number and a CVSS score tell you nothing you can reuse and the mechanism usually does.

This week the interesting thing is not any single bug, its the volume. Four Java projects published multi-CVE batches inside five days, and three of them landed on the same day: Thursday 20 August brought the Spring advisories, six in Netty's OHTTP incubator codec and nine in Apache InLong, then Camel added eight on the Monday. Depending on how you count the Spring batch, that is somewhere between fifty and a hundred and twenty CVEs in one working week, in one language ecosystem.

So I want to write about two things: one bug class that shows up four times inside a single project, and what happens to all this output once it leaves the advisory and tries to reach the people running the code. The second turned out to be the harder problem. Several of this week's fixes are published, real, correct, and still will not reach you by upgrading.

## The week in five

- **Spring: a large batch, 20 August.** Sources disagree on the size, which is worth saying up front. [spring.io/security](https://spring.io/security){:target="_blank"} lists 25 advisories dated 20 August, 7 of them High. Sonatype and SecurityWeek both report 91 CVEs published by Broadcom that day, spanning Spring Security, Cloud Config, AI, Data REST, Integration, Reactor Core, Reactor Netty, AMQP and Batch. The gap is presumably public OSS advisories versus the full count including commercial lines, but no source says so explicitly, so treat the two numbers as measuring different things. The headline is `CVE-2026-59270`, Critical: `UnboundIdContainer` registers a well-known administrative bind DN and binds its listener to all interfaces instead of loopback, so anyone reaching the embedded LDAP port binds as admin and reads or modifies the directory. Fixed in 7.1.1 and 7.0.7. Deep dive below.

- **Apache Camel: 8 CVEs, 24 August.** Four of the eight are one bug class wearing different clothes, the Camel message header map: `CVE-2026-59230` (camel-mail), `CVE-2026-63621` (camel-knative), `CVE-2026-71300` (camel-atmosphere-websocket) and `CVE-2026-78329` (camel-undertow), all external input reaching Camel's internal header namespace without passing a `HeaderFilterStrategy`. Then a path traversal trio, `CVE-2026-60093` (Azure DataLake), `CVE-2026-66906` (Azure Blob) and `CVE-2026-66907` (Google Storage), same shape in all three: a remote object name joined onto a configured `fileDir` with no normalisation, so an object called `../../something` writes where it likes. The outlier is `CVE-2026-66908`, Important, in `camel-platform-http-main`: with JWT configured from a keystore but no `jwtIssuer` or `jwtAudience`, `buildJwtOptions` returned null and claim validation was skipped, so any unexpired token signed by a trusted key was accepted. All fixed in 4.14.9, 4.18.4 and 4.22.0. Deep dive below.

- **netty-incubator-codec-ohttp: 6 CVEs, 20 August.** Four are the Binary HTTP parser, and `CVE-2026-63202`, High, is the best one: the field-section decode loop relies on a Java `assert` for forward progress, and production JVMs run with assertions off. Because the loop counter can go negative and `readFieldLine(...)` can legitimately consume zero bytes on a truncated message, roughly 17 bytes of crafted BHTTP inside an ordinary OHTTP request pins an event-loop thread at 100% CPU permanently. `CVE-2026-63124`, `CVE-2026-61827` and `CVE-2026-61799` are a second infinite loop, unbounded variable-length fields, and varint lengths wrapping into negative `int` offsets. The sixth I would have led with in a quieter week: `CVE-2026-61798`, High, where `BoringSSLAsymmetricKeyParameter.toString()` returns `"bytes=" + Arrays.toString(bytes)` without checking `isPrivate`, so logging a key pair writes raw HPKE private key bytes into your logs. Fixes are split, 0.0.22.Final for the key exposure and 0.0.23 for the parser.

- **Apache InLong: 9 CVEs, 20 August.** Four SQL injections, three authorization gaps, a path traversal and an SSRF. The teaching one is `CVE-2026-63039`, SQL injection through MyBatis dollar-sign interpolation in `AuditAlertRuleService`. MyBatis gives you `#{}`, which becomes a bound parameter, and `${}`, which is substitution into the SQL text before it is prepared: near-identical in a mapper XML file, and one of them is `String` concatenation with extra steps. `CVE-2026-63037` is the same family with no authentication at all, in the ORDER BY of the Manager OpenAPI audit alert endpoint, where you cannot bind a parameter and have to allowlist instead. `CVE-2026-63043`, Important, is arbitrary file read from the Agent host. All affect 2.0.0 through 2.3.x, fixed in 2.4.0.

- **`CVE-2026-76904`, GeoTools, Critical, and older than this window on purpose.** Unauthenticated SQL injection in the `jsonArrayContains` filter function against PostGIS layers: the value goes into the generated SQL unescaped, so an OGC filter arriving over WFS becomes arbitrary SQL. CVSS 9.8, no authentication, no user interaction. It is here because the dates do not agree and the disagreement is the point. GeoTools' own advisory says 15 August, GitLab's database says 21 August, and CyCognito, writing it up as a GeoServer zero-day, says 12 August and reports hundreds of probe attempts from a small pool of addresses, reconnaissance rather then compromise. So this broke during last week's window, I missed it, and the record most dependency scanners consume did not exist until the 21st. Patched in 33.6.0, 34.5.0 and 35.1.0. GeoServer inherits it through `gt-jdbc-postgis`.

Honourable mention to **Sakai** on 24 August, `CVE-2026-54049` (stored XSS in Conversations, High) and `CVE-2026-54050` (IDOR on profile image deletion, Moderate). And a pointer rather than an item: as I write this on the 26th, **Apache Tomcat has just published ten**, including a security constraint bypass and an incomplete fix for `CVE-2026-32990`. Outside this window, so next week's post.

## Apache Camel: the header map is a control plane

I did the remediation work on all eight of these and I am the finder on `CVE-2026-63621`, so read this as an insider's account rather than an independent assessment.

Camel's central abstraction is the Exchange, carrying a message with a body and a map of headers, and those headers are how components talk to each other. The HTTP producer reads `CamelHttpUri` to decide where to send a request, the file producer reads `CamelFileName` to name the file it writes, and the SQL producer will take the statement to execute from a header when one is present. So the header map is not a bag of metadata. It is a control plane, and it shares one namespace with the data plane, because a message that arrived from outside has headers too and they land in the same map.

What keeps the two apart is `HeaderFilterStrategy`, applied by each component when it maps wire headers onto a Camel message, dropping anything in the Camel-internal namespace so an inbound header cannot become an instruction. All four header CVEs are that strategy failing to apply, in four genuinely different ways.

In `camel-mail`, it was never applied. With `headersInline` enabled, the `MimeMultipart` unmarshal path enumerated every MIME header except the three it generates itself and called `setHeader` for each, with no strategy in the path at all. A MIME part carrying a header named after a Camel control header became that control header.

In `camel-knative`, it was applied on one path and not the other. In binary content mode the CloudEvent attributes are HTTP headers and `KnativeHttpConsumer` runs them through `KnativeHttpHeaderFilterStrategy`. In structured mode the whole event is JSON in the body, and `AbstractCloudEventProcessor` mapped every remaining field, the extension attributes whose names the sender chooses, straight onto the message headers with no filtering. And because Camel's header map is case-insensitive, an extension named `camelhttpuri`, a perfectly legal CloudEvent extension name, collides with `CamelHttpUri`.

In `camel-atmosphere-websocket`, the filter ran correctly and did not cover the headers that mattered. `HttpHeaderFilterStrategy` filters the `Camel` and `camel` prefixes, but the websocket producer's dispatch headers are `websocket.connectionKey.list`, `websocket.sendToAll` and friends: dotted, lowercase, outside the namespace the filter knows about. In a bridged route an external sender could supply the list header and take over which clients receive the message.

In `camel-undertow`, the right strategy was constructed and then thrown away. `UndertowEndpoint` instantiated an `UndertowHeaderFilterStrategy` and then handed the base `HttpHeaderFilterStrategy` to `UndertowHttpBinding` instead, so the Undertow-specific logic that rejects legacy websocket prefixes never ran on endpoint-configured routes.

### Why I find it interesting

Because the bug class has an old and well-studied name that nobody thinks to apply to a message header map: **in-band signalling**.

It is the blue box. The phone network carried control tones on the same channel as your voice, so a 2600 Hz whistle down the line was indistinguishable from the switch's own signalling and you got a free call. Same as SQL injection, where query structure and data travel in one string. Same as HTTP response splitting and SMTP header injection. Every one of them is a control channel and a data channel sharing a medium, and a receiver deciding which is which by looking at the content.

Camel's header map is exactly that. `CamelHttpUri` means "send it here" if the framework put it there and "some bytes arrived from the internet" if a component copied it in, and by the time a downstream producer reads the map there is nothing left to tell them apart. Seen that way the four variants stop being four separate bugs and become one design property that has to be guarded everywhere external data enters the map, and the question stops being "did we filter here" and becomes "how many places copy into this map, and do we have a list".

For Camel that list was not written down anywhere, which is what a boundary enforced by convention tends to look like in a project with several hundred components. The structural answer is not more careful review, it is making the unfiltered path harder to write than the filtered one, and that is a better thing to take from this batch than a count of CVEs.

The near-term advice is duller and still worth doing. If a route takes input from outside and hands it to a header-driven component, put `removeHeaders("Camel*")` at the boundary rather then relying on every component upstream having got this right, add `removeHeaders("websocket.*")` if you bridge into a websocket producer, and set `bridgeEndpoint` on HTTP producers that should not be redirectable.

## Spring: ninety one CVEs, and where the fixes actually go

The Spring batch is the bigger story of the week and almost none of it is about a specific bug. Start with the volume. Sonatype's writeup reports that Spring historically averaged about 6.5 new security reports a month. In March 2026 that was 55. In April it was 482 across 65 scanned projects, 370 of them from Spring's own scanning and 112 from the community. That is not a doubling, its two orders of magnitude in about six weeks.

The cause is AI-assisted vulnerability discovery, and the reporting here needs care. Sonatype notes that Spring has pointed to Anthropic's Mythos research as an example of models finding bugs at greater scale, and is explicit that it has not confirmed Mythos found the 91 CVEs in this batch. SecurityWeek attributes the surge to "Broadcom's use of AI" without saying how individual flaws were found. So: the volume is documented, the mechanism is AI-assisted scanning, and the attribution of any specific CVE to any specific system is not established by anything I read.

It is not only Spring. This week's `CVE-2026-59230` in Camel is credited, in the advisory's own words, to "Atuin - Automated Vulnerability Discovery Engine, anciety of Tencent Xuanwu Lab". A named machine in the credits line of an Apache advisory, which is worth noticing as a plain fact about how this work will arrive from now on.

Now the part that matters to somebody running Spring. `CVE-2026-59270` affects 5.7.0 all the way up to 7.1.0. The OSS fixes are 7.1.1 and 7.0.7. The fixes for 6.5.x, 6.4.x, 5.8.x and 5.7.x are, in the advisory's own words, Enterprise Support only. The same split appears on `CVE-2026-59318` in Spring AI, where 2.0.1 is OSS and 1.1.9 and 1.0.10 are commercial.

I want to be fair about this. Somebody has to pay for maintaining a six year old branch, commercial LTS is a legitimate way to fund that, and the disclosure itself is public, detailed and free. None of that is the complaint. The complaint is narrower: for a large slice of the installed base, the vulnerability became public knowledge on 20 August and the patch did not. Publishing the flaw and withholding the fix, even for good reasons, hands attackers a diff and hands defenders an invoice.

### Why I find it interesting

Because there is a much milder version of the same shape in this week's Camel batch, and I only noticed it by looking at Spring first.

`CVE-2026-66908` is the JWT issue: no `jwtIssuer` or `jwtAudience`, `buildJwtOptions` returns null, claim validation silently skipped, server starts clean with no warning. The fix in 4.22.0 is fail-closed: `assertIssuerOrAudienceConfigured` runs at the top of both `configureAuthentication` overloads and throws `IllegalArgumentException` if a JWT keystore has neither claim configured, and if you genuinely want signature-and-expiry-only you set `jwtAllowMissingIssuerAndAudience` explicitly. That is the right shape of fix, and it is the one from [last week's post](/2026/08/19/java-vulnerabilities-of-the-week-12-18-august-2026.html): a control that fails open and reports nothing, converted into one that refuses to start.

But that enforcement only exists in 4.22.0. The backports to 4.14.9 and 4.18.4 add the options and deliberately change no existing behaviour, because a maintenance release that suddenly refuses to start on a configuration which worked yesterday is its own kind of outage. I think that is the right call for an LTS line. It does mean the upgrade on its own leaves claim checking off until you set one of the two properties yourself.

Two projects, two very different reasons, one shared consequence: **a version number can move without the security posture moving with it.** Spring's reason is commercial and Camel's is compatibility, and I do not think those deserve the same verdict at all. But a scanner reading a version string cannot tell either of them from a real fix, and that is the gap I would worry about more than any single bug this week. I have no clean answer beyond reading the remediation section for your own LTS lines rather then trusting the fixed-in field.

## What I take away from this week

Three threads.

The first is that the volume problem is now a consumption problem, not a discovery problem. Roughly 114 CVEs across four Java projects in five days, if you take Broadcom's count for Spring, and Sonatype puts over 200,000 software components in scope for that batch alone. Nobody's Tuesday has room for it, and the failure mode will not be a missed critical, it will be triage fatigue: 91 advisories arrive, 90 do not apply to your configuration, and the one that does gets the same thirty seconds as the rest.

The second is that the plumbing between disclosure and your scanner is creaking under it, and GeoTools is the evidence. A 9.8 unauthenticated SQL injection, disclosed somewhere between 12 and 15 August depending on which source you believe, already being probed in the wild, and the GitHub and GitLab records landed on the 21st. Six to nine days when the exploit was public and your dependency scanner said nothing, in the same week four projects dumped over a hundred CVEs into the same pipes. I do not think that is a coincidence, and I do not think a weekly blog post is a fix for it either.

The third is practical. Most people will pick up the header fixes through a BOM bump in Spring Boot, Quarkus or Camel itself and never think about them again, which is the system working. The ones who need to move deliberately: anyone on `camel-platform-http-main` with JWT from a keystore, who should set `jwtIssuer` or `jwtAudience` themselves since the LTS backports leave that choice to you, anyone on `netty-incubator-codec-ohttp` (0.0.23, and go grep your logs for `CVE-2026-61798`), anyone running InLong below 2.4.0, and anyone with GeoServer or GeoTools in front of a PostGIS database, who should stop reading and patch that one first.

## References

- [oss-security archive, 24 August 2026](https://www.openwall.com/lists/oss-security/2026/08/24/){:target="_blank"} - Openwall
- [oss-security archive, 20 August 2026](https://www.openwall.com/lists/oss-security/2026/08/20/){:target="_blank"} - Openwall
- [oss-security archive, 26 August 2026](https://www.openwall.com/lists/oss-security/2026/08/26/){:target="_blank"} - Openwall
- [CVE-2026-59230, camel-mail MimeMultipart header injection](https://camel.apache.org/security/CVE-2026-59230.html){:target="_blank"} - Apache Camel
- [CVE-2026-63621, camel-knative structured mode header injection](https://camel.apache.org/security/CVE-2026-63621.html){:target="_blank"} - Apache Camel
- [CVE-2026-71300, camel-atmosphere-websocket dispatch header injection](https://camel.apache.org/security/CVE-2026-71300.html){:target="_blank"} - Apache Camel
- [CVE-2026-78329, camel-undertow header filter strategy discarded](https://camel.apache.org/security/CVE-2026-78329.html){:target="_blank"} - Apache Camel
- [CVE-2026-66908, camel-platform-http-main JWT claim validation](https://camel.apache.org/security/CVE-2026-66908.html){:target="_blank"} - Apache Camel
- [CVE-2026-66906, camel-azure-storage-blob path traversal](https://camel.apache.org/security/CVE-2026-66906.html){:target="_blank"} - Apache Camel
- [CVE-2026-66907, camel-google-storage path traversal](https://camel.apache.org/security/CVE-2026-66907.html){:target="_blank"} - Apache Camel
- [Spring Security Advisories](https://spring.io/security){:target="_blank"} - Spring
- [CVE-2026-59270, embedded UnboundID LDAP administrative bind DN](https://spring.io/security/cve-2026-59270/){:target="_blank"} - Spring
- [CVE-2026-59285, Spring for GraphQL unsafe deserialization](https://spring.io/security/cve-2026-59285/){:target="_blank"} - Spring
- [CVE-2026-59318, Spring AI unadvertised tool dispatch](https://spring.io/security/cve-2026-59318/){:target="_blank"} - Spring
- [91 Spring CVEs Highlight the Growing AI Vulnerability Consumption Problem](https://www.sonatype.com/blog/91-spring-cves-highlight-the-growing-ai-vulnerability-consumption-problem){:target="_blank"} - Sonatype
- [91 Vulnerabilities Patched in Spring Application Framework](https://www.securityweek.com/91-vulnerabilities-patched-in-spring-application-framework/){:target="_blank"} - SecurityWeek
- [Anthropic Mythos exposes gaps in AI vulnerability reporting](https://cybernews.com/ai-news/anthropic-mythos-ai-vulnerability-reporting-cve-disclosure/){:target="_blank"} - Cybernews
- [CVE-2026-63202, BinaryHttpParser CPU-exhaustion DoS](https://advisories.gitlab.com/maven/io.netty.incubator/netty-incubator-codec-bhttp/CVE-2026-63202/){:target="_blank"} - GitLab Advisory Database
- [CVE-2026-61798, BoringSSL HPKE private key bytes exposed](https://advisories.gitlab.com/maven/io.netty.incubator/netty-incubator-codec-ohttp-hpke-classes-boringssl/CVE-2026-61798/){:target="_blank"} - GitLab Advisory Database
- [CVE-2026-61827, BinaryHttpParser variable length field limits](https://advisories.gitlab.com/maven/io.netty.incubator/netty-incubator-codec-bhttp/CVE-2026-61827/){:target="_blank"} - GitLab Advisory Database
- [CVE-2026-63124, Binary HTTP parser infinite loop](https://advisories.gitlab.com/maven/io.netty.incubator/netty-incubator-codec-bhttp/CVE-2026-63124/){:target="_blank"} - GitLab Advisory Database
- [CVE-2026-61799, Binary HTTP parser varint length overflow](https://advisories.gitlab.com/maven/io.netty.incubator/netty-incubator-codec-bhttp/CVE-2026-61799/){:target="_blank"} - GitLab Advisory Database
- [CVE-2026-76904, unauthenticated SQL injection in jsonArrayContains](https://github.com/geotools/geotools/security/advisories/GHSA-mqjf-5f49-2fjh){:target="_blank"} - GeoTools
- [CVE-2026-76904 advisory record](https://advisories.gitlab.com/maven/org.geotools.jdbc/gt-jdbc-postgis/CVE-2026-76904/){:target="_blank"} - GitLab Advisory Database
- [Emerging Threat: GeoServer Zero-Day SQL Injection via jsonArrayContains](https://www.cycognito.com/blog/emerging-threat-geoserver-zero-day-sql-injection-via-jsonarraycontains/){:target="_blank"} - CyCognito
- [CVE-2026-63039, InLong MyBatis dollar-sign interpolation](https://github.com/apache/inlong/pull/12080){:target="_blank"} - Apache InLong
- [GitHub Advisory Database](https://github.com/advisories){:target="_blank"} - GitHub
