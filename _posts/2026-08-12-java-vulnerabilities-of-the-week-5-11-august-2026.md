---
layout: post
title: "Java vulnerabilities of the week: 5 to 11 August 2026"
---

This one covers 5 to 11 August, the next one covers 12 to 18. Same idea as always: what caught my attention in the Java ecosystem, then two items looked at properly, because the CVE number and the score are the least useful part of any of this. The mechanism is what you can carry to your own code.

The thread running through this week is what happens when two pieces of code look at the same input and reach different conclusions about it. A sanitizer and a browser. A hardened XML parser and the separate library that handles its imports. A class filter and the resolution path that runs alongside it. A username canonicaliser and the comparison applied to its output. In each case both halves are reasonable on their own, and the vulnerability lives in the space between them.

## The week in five

- **Apache CXF: 12 CVEs, 6 August.** Fixed in 4.2.3, 4.1.8 and 3.6.12, which means all three maintained lines including the older one. Two Important issues from n0mi1k: `CVE-2026-65432`, XXE through WSDL and XSD imports, and `CVE-2026-66909`, native Java deserialization of inbound JMS `ObjectMessage`. Seven OAuth2 and OpenID Connect findings from Guanping Zhang, clearly the result of one systematic pass over the specs. `CVE-2026-64958` from Markus Vogl at RISE GmbH, a follow-up on the earlier attachment header fix. And two denial of service issues that were really missing defaults, now set: a 50mb attachment cap (`CVE-2026-54225`) and a 500 form parameter cap (`CVE-2026-57819`). Deep dive below.

- **Apache Ranger: 10 CVEs, 9 August, fixed in 2.9.0.** Four are code execution: `CVE-2026-42537` (JDBC URL injection), `CVE-2026-44416` (arbitrary class instantiation in `plugin-schema-registry`), `CVE-2026-55799` (`GraalScriptEngineCreator`) and `CVE-2026-28672` (command injection through a username in `UnixUserGroupBuilder`). The rest cover SQL injection in lookup, privilege escalation through a URL parameter, unauthenticated download APIs, TLS certificates accepted for the wrong hostname, replayable JWTs in logs and no brute force protection on UnixAuth. Reporters include Andrew Rukin of Arenadata on six of them. Deep dive below.

- **Jenkins security advisory, 5 August.** Five in core, eighteen across plugins. The Critical one is `CVE-2026-70426`: the JEP-200 class filter, which is what stands between an agent and arbitrary deserialization on the controller, is not applied to classes resolved through a fallback path in Remoting. The one I will remember longest is `CVE-2026-70429`. Jenkins canonicalises usernames by lowercasing them, then compares them with `String#equalsIgnoreCase`, and those are not the same relation. The dotless i (`ı`) compares equal to `i` under `equalsIgnoreCase` but does not lowercase to it, so on a realm that permits non-ASCII you can register a name that collides with an existing user. Core is fixed in weekly 2.576 and LTS 2.568.2, and the comparison now runs on the canonical form. Eleven plugin issues have no fix yet, and Jenkins publishes those anyway, which is the behaviour you want even though it makes the advisory longer.

- **`CVE-2026-71497`, jsoup, Medium, 6 August.** The advisory says a tag name ending in a control character "may acquire the parsing behavior of a different element", so content that should stay text is emitted as active markup after serialization. The issue thread gives the cause: the tokenizer normalised tag names with `trim()` before resolving them, and the fix stops trimming source names while keeping the trim for programmatic `Tag.valueOf()` input. Reading those together, the trimmed name resolves to a raw-text element, the tokenizer swallows everything up to the close tag as data, and the untrimmed name goes back out on serialization where no browser treats it as raw text. That last step is my reconstruction rather than something the advisory spells out. Only custom `Safelist` configurations that permit raw-text elements are affected. Fixed in 1.23.1.

- **`CVE-2026-61899`, Apache Tapestry, Critical, 8 August.** Classpath assets downloadable through crafted URLs in `tapestry-core`. The advisory is short and does not describe the URL manipulation, so I cannot tell you what it looks like, but reading arbitrary files from a web application's classpath reaches configuration, credentials and class files, and Critical is the right label. Affects 5.5.0 through 5.9.0, fixed in 5.9.1, reported by Ilyass El Hadi.

Honourable mention to **`CVE-2026-56818` in `netty-codec-redis`**, 7 August, where the diff is the whole explanation. `RedisArrayAggregator.decodeRedisArrayHeader` has two adjacent guard clauses. The nested-depth one calls `releaseAndClearDepths()` before throwing. The max-elements one, right above it, throws directly, so the retained partial aggregate survives the exception. Fixed in 4.1.136.Final and 4.2.16.Final. Also **`CVE-2026-44630` in Apache IoTDB**, 10 August, an unchecked Thrift string length and a pre-authentication OOM, fixed in 2.0.10, and **`CVE-2026-64640` in Apache Polaris**, 6 August, a registration endpoint that reads a caller-supplied Iceberg metadata file before checking it is inside the catalog's allowed locations, which I reported, so I will leave that one alone.

## Apache CXF: hardening that stops at the import

CXF is the SOAP and JAX-RS stack that a large amount of Java integration work still runs on, often through something else rather than directly. If there is a WSDL contract anywhere in your architecture, CXF is probably parsing it.

`CVE-2026-65432` is the clearest of the twelve. CXF reads a top-level WSDL through its own `StaxUtils` path, which disables DTDs and external entities. That hardening has been there for years and it works. But a WSDL can contain `<wsdl:import>` and `<xsd:import>`, and CXF hands those to WSDL4J, a separate and much older library that does not disable DOCTYPE declarations or external entities and that CXF does not control.

So the document you fetched is protected and the document it references is not. An attacker does not touch the top-level WSDL at all: they add one import, and everything interesting happens in the imported file, parsed by a different library with different defaults. This is a genuinely awkward class of problem, because the fix is not "turn on the flag", its "find every component downstream of your entry point and establish what its defaults are".

The other half of the batch that stayed with me is the pair of encrypting data providers. `CVE-2026-68079`: `DefaultEncryptingCodeDataProvider` allows an authorization code to be redeemed more than once, which RFC 6749 says it must not be. `CVE-2026-68481`: `DefaultEncryptingOAuthDataProvider` cannot invalidate a revoked token, so a revoked token still decrypts and introspection reports `active: true`.

### Why I find it interesting

Because the second pair is not really a coding mistake, it is a consequence of a design choice that nobody wrote down.

The appeal of a self-contained encrypted token is that the authorization server holds no state. The token carries its claims, you decrypt it, you trust it, you scale out without a shared store. That is the entire point of the design. But "already used" and "revoked" are both statements about the past, and a token that carries only its own claims carries no history. There is nowhere for `removeCodeGrant` to write, and nothing for revocation to change. The interface is satisfied by the type system and not by the implementation.

That generalises well beyond OAuth, and its worth having in mind whenever you trade a lookup for a self-describing artifact. Signed JWTs instead of session ids: you give up logout. Presigned URLs instead of an authorization call: you give up revocation. The performance argument is usually correct, and the property you dropped tends to go unnoticed until someone writes it up.

The XXE one generalises differently, and it is the week's theme at its clearest. CXF hardened a code path; the input took a data path. The useful question is not "is this parser hardened" but "does every parser that will see bytes derived from this input have the same settings". Imports, includes, redirects, nested archives, external DTD subsets, JSON `$ref`: anything that says "now go and read that other thing" is a handoff, and the receiving code rarely inherits the caller's configuration.

I want to note what CXF did here as well as what went wrong. Twelve issues from three separate reporters, fixed and released across three branches on the same day, with per-CVE credit and mechanism described in each advisory. Zhang's seven findings came from working through the OIDC and OAuth2 requirements against the implementation one at a time, which is unglamorous work that anyone with the RFC open in another tab can do, and it found real problems in a mature codebase.

## Apache Ranger: ten advisories and a template that did not render

Ranger is the authorization layer for a Hadoop-shaped data platform. It holds the policies that decide who reads which Hive table, which HDFS path, which Kafka topic, so that the answer to "can this user do that" lives in one auditable place.

Ten advisories went out on 9 August, four of them code execution. `CVE-2026-42537` is JDBC URL injection, which in practice means someone who controls a connection string gets to pick driver properties, and several JDBC drivers will turn a connection property into class loading or a file read. `CVE-2026-44416` is arbitrary class instantiation in `plugin-schema-registry`. `CVE-2026-55799` is a script engine reachable from input. `CVE-2026-28672` is command injection where a username reaches a shell.

All of that is fixed in Ranger 2.9.0. But nine of the ten advisories, as posted to oss-security, contain this line:

> Users are recommended to upgrade to version [FIXED_VERSION], which fixes this issue.

The literal placeholder, brackets and all. The information itself is not missing: the GitHub Advisory Database record for `CVE-2026-42537`, `GHSA-4rwg-mr3v-x763`, carries the same sentence with the version filled in. So one rendering of the advisory came out correct and another did not, from the same source, the same day.

### Why I find it interesting

Not as a criticism of the people who did the work. Publishing ten advisories with per-CVE credit, mechanism and affected ranges, rather then quietly folding the fixes into a release, is exactly the behaviour the ecosystem should be asking for, and it is more effort than staying silent.

It is interesting because of what an advisory is for, and there are two answers. The CVE record is for tooling: a scanner reads it, matches it against your SBOM, opens a ticket. The mailing list post is for a person reading the security list in the morning and deciding whether today is a patching day. Both need the same fact, they travel through different pipelines, and a project has to get both right without much help from the tooling. That is a real gap, and it is the sort of thing that would be worth solving once for everyone rather than per project.

The second part matters more in practice. The GitHub records for these advisories have an empty affected-packages array: no Maven coordinates, no version range. Which means a scanner that would otherwise tell you your `ranger-plugins-common` is affected has nothing to match on. Ten CVEs, four of them code execution, and the machine-readable half that would reach most people through their existing tooling is blank. Severity is inconsistent too: the ASF advisory calls `CVE-2026-42537` important, GitHub's database rates it Critical at 9.8. When the project and the database disagree by two bands, whichever your dashboard reads is quietly making the decision for you.

The practical version is short. If you run Ranger 2.8.0 or earlier, the version you want is 2.9.0. And do not assume a published CVE is a matchable CVE, because empty version ranges are silent. Which is another angle on something I wrote about in February, that we tend to read [a CVE as a verdict on a project](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html) when it is really a message, and a message is only as good as the channel carrying it.

## What I take away from this week

The first is a habit worth building. When you harden something, ask what else parses the same input. CXF hardened its WSDL reader and WSDL4J read the imports. Jenkins wrote a class filter and Remoting had a second resolution path. jsoup normalised a tag name for lookup and serialized the original. Jenkins canonicalised usernames one way and compared them another. In each case both halves are individually correct, and the question that finds the gap is not "is this function safe", its "who else touches this, and do we agree".

The second is that some defects live in the shape of a system rather than in a method. CXF's encrypting providers cannot revoke a token however carefully they are written, because there is no state to revoke against. Code review looks at the method; that class of problem is one level up. When you trade a lookup for a self-contained artifact, write down what you gave up, then check whether some interface in your codebase still promises it.

On arrival: CXF reaches most people through a BOM, via Camel, Karaf or anything doing SOAP on the JVM. jsoup is usually a direct dependency and a one line bump. Ranger is deployed rather then depended on, so it moves when someone upgrades a cluster. Jenkins core is on you. If I had to order them, Jenkins first, because `CVE-2026-70426` turns agent access into controller code execution and plenty of setups treat agents as semi-trusted. Then CXF, because the JMS `ObjectMessage` issue hands `readObject` to anyone who can put a message on the destination, and the fix disables the feature by default rather then patching around it.

Next post covers 12 to 18 August.

## References

- [CVE-2026-65432: Apache CXF, XXE via WSDL/XSD import parsing](https://www.openwall.com/lists/oss-security/2026/08/06/17){:target="_blank"} - oss-security
- [CVE-2026-66909: Apache CXF, unsafe deserialization of inbound JMS ObjectMessage](https://www.openwall.com/lists/oss-security/2026/08/06/18){:target="_blank"} - oss-security
- [CVE-2026-54225: Apache CXF, denial of service via large attachments](https://www.openwall.com/lists/oss-security/2026/08/06/14){:target="_blank"} - oss-security
- [CVE-2026-57819: Apache CXF, no default restriction on the amount of form parameters per message](https://www.openwall.com/lists/oss-security/2026/08/06/15){:target="_blank"} - oss-security
- [CVE-2026-64958: Apache CXF, denial of service via message header attachments](https://www.openwall.com/lists/oss-security/2026/08/06/16){:target="_blank"} - oss-security
- [CVE-2026-57817: Apache CXF, the authorization code hash (c_hash) is not enforced for the hybrid OIDC flow](https://www.openwall.com/lists/oss-security/2026/08/06/19){:target="_blank"} - oss-security
- [CVE-2026-57818: Apache CXF, OAuth2 authorization code replay via TOCTOU in JCacheCodeDataProvider](https://www.openwall.com/lists/oss-security/2026/08/06/20){:target="_blank"} - oss-security
- [CVE-2026-61466: Apache CXF, OAuth2 dynamic client registration scope self-escalation](https://www.openwall.com/lists/oss-security/2026/08/06/21){:target="_blank"} - oss-security
- [CVE-2026-63687: Apache CXF, JwtRequestCodeFilter silently overrides outer PKCE and nonce parameters](https://www.openwall.com/lists/oss-security/2026/08/06/22){:target="_blank"} - oss-security
- [CVE-2026-65583: Apache CXF, self-issued ID token claims validation skipped](https://www.openwall.com/lists/oss-security/2026/08/06/23){:target="_blank"} - oss-security
- [CVE-2026-68079: Apache CXF, DefaultEncryptingCodeDataProvider allows unlimited authorization code replay](https://www.openwall.com/lists/oss-security/2026/08/06/24){:target="_blank"} - oss-security
- [CVE-2026-68481: Apache CXF, revocation bypass in DefaultEncryptingOAuthDataProvider](https://www.openwall.com/lists/oss-security/2026/08/06/25){:target="_blank"} - oss-security
- [CVE-2026-28672: Apache Ranger, OS command injection via username in UnixUserGroupBuilder](https://www.openwall.com/lists/oss-security/2026/08/09/2){:target="_blank"} - oss-security
- [CVE-2026-42537: Apache Ranger, remote code execution via JDBC URL injection](https://www.openwall.com/lists/oss-security/2026/08/09/5){:target="_blank"} - oss-security
- [CVE-2026-44416: Apache Ranger, remote code execution via arbitrary class instantiation](https://www.openwall.com/lists/oss-security/2026/08/09/6){:target="_blank"} - oss-security
- [CVE-2026-55799: Apache Ranger, remote code execution in GraalScriptEngineCreator](https://www.openwall.com/lists/oss-security/2026/08/09/7){:target="_blank"} - oss-security
- [CVE-2026-65942: Apache Ranger, clients accept TLS certificates issued for other hostnames](https://www.openwall.com/lists/oss-security/2026/08/09/9){:target="_blank"} - oss-security
- [GHSA-4rwg-mr3v-x763: remote code execution via JDBC URL injection in Apache Ranger](https://github.com/advisories/GHSA-4rwg-mr3v-x763){:target="_blank"} - GitHub Advisory Database
- [Download Apache Ranger](https://ranger.apache.org/download.html){:target="_blank"} - Apache Ranger
- [Jenkins Security Advisory 2026-08-05](https://www.jenkins.io/security/advisory/2026-08-05/){:target="_blank"} - Jenkins
- [GHSA-pmhh-3w7g-xqp8: jsoup Cleaner may expose markup with custom raw-text elements](https://github.com/advisories/GHSA-pmhh-3w7g-xqp8){:target="_blank"} - GitHub Advisory Database
- [jsoup issue 2538: tokenizer tag name normalization](https://github.com/jhy/jsoup/issues/2538){:target="_blank"} - jsoup
- [CVE-2026-61899: Apache Tapestry, possible classpath file download through URL manipulation](https://www.openwall.com/lists/oss-security/2026/08/08/2){:target="_blank"} - oss-security
- [GHSA-p9jm-q85p-7mcp: Netty RedisArrayAggregator max-elements failure leaves retained partial aggregate state](https://github.com/advisories/GHSA-p9jm-q85p-7mcp){:target="_blank"} - GitHub Advisory Database
- [CVE-2026-44630: Apache IoTDB, RPC service denial of service via unchecked Thrift string length](https://www.openwall.com/lists/oss-security/2026/08/10/1){:target="_blank"} - oss-security
- [CVE-2026-64640: Apache Polaris, register endpoint reads attacker-controlled storage location before allowed-locations validation](https://www.openwall.com/lists/oss-security/2026/08/06/7){:target="_blank"} - oss-security
- [oss-security daily archive](https://www.openwall.com/lists/oss-security/2026/08/09/){:target="_blank"} - Openwall
