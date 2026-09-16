---
layout: post
title: "Java vulnerabilities of the week: 11 to 16 September 2026"
---

Most security news is useless to me as a developer: a CVE number, a score, one line saying an attacker may do something bad, and nothing about why. The part worth knowing is the mechanism, the exact queue with no limit, the exact name that turns out to be guessable. See the shape once and you spot it in your own code, which is worth far more then the score. I wrote about that in [the CVE stigma post](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html){:target="_blank"}.

This week is coordinated batches, not one-off bugs: Netty fifteen advisories, Storm fifteen CVEs, Syncope twenty-two, ZooKeeper five. A pile of fixes on one day usually means someone swept the whole surface, and this week that someone was often an automated agent. The biggest item, the Netty batch, landed on 10 September, the day my last post went out, and missed it. It reaches more of you than anything else here, so it leads.

## The week in five

- **The Netty batch, fifteen advisories, disclosed 10 September 2026, fixed in 4.1.138.Final and 4.2.18.Final.** Mostly unbounded resource allocation (CWE-770). `HttpServerCodec` grows an unbounded `methodOverflowQueue` when a client pipelines HTTP/1.1 requests and never reads the responses; the same shape hits `HttpContentEncoder` (`CVE-2026-59899`), `SmtpResponseDecoder` (`CVE-2026-44891`), a `Bzip2Decoder` infinite loop (`CVE-2026-59901`), and an HTTP/3 QPACK integer. The sharp non-DoS one is `CVE-2026-50010`: TLS hostname verification silently disabled on the OpenSSL client path on Java 25 and later. Affected 4.1.x up to 4.1.137.Final and 4.2.x up to 4.2.17.Final. Deep dive below.

- **The Apache Storm batch, `CVE-2026-82426` through `CVE-2026-82441` plus `CVE-2026-84179`, important, disclosed 13 September 2026, fixed in 3.1.0.** Multi-tenant isolation failures: `CVE-2026-82428` substitutes another tenant's dependency jar through predictable blob keys, `CVE-2026-82426` reads Nimbus's keytab or TLS keys as a "topology jar", and `CVE-2026-82429` escalates to root through a TOCTOU race in the setuid-root `worker-launcher`. The rest are authorization bypasses, disclosure, and DoS. Many credit "the ASF using Claude Agents" as the finder. Deep dive below.

- **The Apache ZooKeeper batch, five CVEs, disclosed 15 September 2026, fixed in 3.9.6 and 3.8.7.** `CVE-2026-59739` (Critical) is an incomplete fix of 2024's `CVE-2024-23944` that leaks ACL-restricted path names on reconnect replay; `CVE-2026-79993` (Critical) lets any client, even unauthenticated on the default port, delete empty znodes; plus a FIPS-mode hostname failure (`CVE-2026-59969`) and two log-forgery bugs via tab and newline injection (`CVE-2026-84439`, `CVE-2026-84501`). Affected 3.9.0 to 3.9.5 and 3.8.0 to 3.8.6.

- **The Apache Syncope batch, twenty-two CVEs, disclosed 14 September 2026, fixed in 4.0.8 and 4.1.3.** A full-surface sweep of an identity system: JWT subject spoofing (`CVE-2026-87785`), admin escalation via JWT (`CVE-2026-78330`), a Groovy sandbox escape (`CVE-2026-77147`), SQL injection in Task search (`CVE-2026-82232`), Cypher injection on Neo4j (`CVE-2026-86460`), unauthenticated reflected XSS (`CVE-2026-78318`), cross-realm authorization bypasses, and secret disclosure via logs (`CVE-2026-87779`). There is no fixed 3.0.x release on Maven Central (3.0.16 is the last), so 3.0 users have to move to 4.0.8 or 4.1.3.

- **Apache OpenNLP, `CVE-2026-82617` and `CVE-2026-67211`, disclosed 11 September 2026, fixed in opennlp-tools 2.5.12 and opennlp-core 3.0.0-M6.** A ReDoS in OpenNLP's own built-in EMAIL and URL patterns in `RegexNameFinderFactory` (ambiguous nested quantifiers, so a long local-part with no `@` drives quadratic rescanning and the URL pattern can stack-overflow on a few KB), and an out-of-memory DoS in `SymSpellModelSerializer`. Affected opennlp-tools 2.0.0 to 2.5.11 and opennlp-core 3.0.0-M1 to 3.0.0-M5.

An honourable mention for **Apache Doris**, `CVE-2026-68570` and `CVE-2026-72524`, important, disclosed 14 September 2026, fixed in 4.0.8 and 4.1.4: authorization bypasses that let a low-privilege user read, write or drop tables they were never granted.

## Netty: when the client stops reading, your server keeps allocating

Netty is the piece almost nobody depends on directly and almost everybody ships: it sits under Spring WebFlux, gRPC-Java, the Elasticsearch and Cassandra drivers, Vert.x, and a long tail of clients you never named in a `pom.xml`. So a fifteen-advisory batch reaches almost everyone.

HTTP/1.1 pipelining lets a client send request two before reading the response to request one, and the server must answer in order. `HttpServerCodec` remembers, per connection, which method each still-unanswered request used; the first thirty-two fit in a bit-packed `long`, the rest spill into an unbounded `ArrayDeque`. Pipeline thousands of requests and never read the responses, and that queue grows until the process falls over, because the responses that would drain it are stuck behind a client that will not read. The same shape appears in `HttpContentEncoder` and in the SMTP, QPACK and Bzip2 paths. Upgrade to 4.1.138.Final or 4.2.18.Final, both on Maven Central.

### Why I find it interesting

This is CWE-770, no bound on state that grows with attacker-controlled work, and the lever is free: the attacker sends nothing malformed, they just decline to read. Pipelining decouples request arrival from response consumption, so any structure keyed to "requests currently in flight" is one the peer can size at will. Every per-connection buffer or queue whose size the other side influences needs an explicit cap and a backpressure story. "The client will read eventually" is an assumption, not a bound, and a hostile client's whole job is to violate your assumptions.

## Apache Storm: a predictable name is a capability you did not mean to hand out

A Storm cluster is multi-tenant: many users submit topologies that run as separate workers, often as different OS users, all sharing Nimbus and a blob store. This batch is a tour of where that isolation leaked.

The one to remember is `CVE-2026-82428`. When you run `storm jar --artifacts`, Storm stored each Maven dependency in the blob store under a key derived from its coordinates, `dep-<group>-<artifact>-<version>.jar`, identical and predictable for every user. When a blob already existed under that key, the uploader silently reused it with no check of owner or contents. So whoever uploaded a coordinate first controlled the bytes every later submitter loaded onto their worker classpath. Poison a `commons-lang3` once and the next tenant who declares it runs your code. Thats cross-tenant remote code execution out of a naming decision. The fix in 3.1.0 uses UUID keys and lives in the submitting client, so every `storm` client has to upgrade, not just the cluster. Alongside it sit `CVE-2026-82429`, a TOCTOU race where the setuid-root `worker-launcher` re-resolves a pathname after FTS has classified it, so a tenant swaps a directory for a symlink and redirects a root-privileged `chown`, and `CVE-2026-82426`, where Nimbus accepts an `uploadedJarLocation` it never confirmed was uploaded, handing a tenant its own keytab.

### Why I find it interesting

A predictable, shared identifier is a capability. A blob key computed from public Maven coordinates is a name anyone can compute and claim, the insecure-direct-object-reference family, and "reuse whatever is already here" turns it into a way to plant bytes for someone else to execute. The fix is two parts together: unguessable identifiers, and an ownership check on reuse. More broadly, in a multi-tenant system every point where one tenant's input becomes another's execution context is a boundary to enforce, not to assume was handled upstream.

## What I take away from this week

Two things to act on. First, these are batches: when a project ships fifteen or twenty-two fixes at once it swept the whole surface and designed the fixes as a set, so take the batch release (Netty 4.1.138.Final or 4.2.18.Final, Storm 3.1.0, ZooKeeper 3.9.6 or 3.8.7, Syncope 4.0.8 or 4.1.3, OpenNLP 2.5.12) rather then cherry-picking the top score. Second, where the fix reaches you differs: Netty arrives transitively, so a Spring Boot, Camel or Quarkus BOM bump carries it, though check your resolved `io.netty` because pins lag; the rest (Storm, ZooKeeper, Syncope, Doris) are services you operate and patch by hand, and Syncope 3.0.x means a move to 4.x because there is no 3.0 fix.

One thing is worth stating plainly, because it is the factual news: many of the Storm advisories credit "the ASF using Claude Agents" as the finder. An agent walked every opcode and code path and produced a coherent batch that maintainers then triaged, fixed and shipped under their own review. The shift is coverage, and these are real, project-validated CVEs. The interesting question is still the mechanism, not who filed the report.

## References

- [oss-security daily index for 13 September 2026, Apache Storm batch](https://www.openwall.com/lists/oss-security/2026/09/13/){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-82428, Apache Storm cross-tenant dependency jar substitution](https://www.openwall.com/lists/oss-security/2026/09/13/8){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-82429, Apache Storm worker-launcher TOCTOU privilege escalation](https://www.openwall.com/lists/oss-security/2026/09/13/9){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-82426, Apache Storm Nimbus arbitrary file read](https://www.openwall.com/lists/oss-security/2026/09/13/6){:target="_blank"} - Openwall oss-security
- [oss-security daily index for 15 September 2026, Apache ZooKeeper batch](https://www.openwall.com/lists/oss-security/2026/09/15/){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-59739, Apache ZooKeeper SetWatches reconnect replay disclosure](https://www.openwall.com/lists/oss-security/2026/09/15/3){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-79993, Apache ZooKeeper missing ACL check on deleteContainer](https://www.openwall.com/lists/oss-security/2026/09/15/5){:target="_blank"} - Openwall oss-security
- [Apache ZooKeeper security advisories page](https://zookeeper.apache.org/security.html){:target="_blank"} - Apache ZooKeeper
- [oss-security daily index for 14 September 2026, Apache Syncope and Doris batches](https://www.openwall.com/lists/oss-security/2026/09/14/){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-73178, Apache Syncope JWT access-token takeover](https://www.openwall.com/lists/oss-security/2026/09/14/4){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-72524, Apache Doris authorization bypass](https://www.openwall.com/lists/oss-security/2026/09/14/3){:target="_blank"} - Openwall oss-security
- [oss-security daily index for 11 September 2026, Apache OpenNLP items](https://www.openwall.com/lists/oss-security/2026/09/11/){:target="_blank"} - Openwall oss-security
- [oss-security: CVE-2026-82617, Apache OpenNLP ReDoS in built-in EMAIL and URL patterns](https://www.openwall.com/lists/oss-security/2026/09/11/10){:target="_blank"} - Openwall oss-security
- [Netty security advisories listing](https://github.com/netty/netty/security/advisories){:target="_blank"} - Netty on GitHub
- [Netty GHSA-pvjx-v7vp-62vq, HttpServerCodec unbounded methodOverflowQueue](https://github.com/netty/netty/security/advisories/GHSA-pvjx-v7vp-62vq){:target="_blank"} - Netty on GitHub
- [Netty GHSA-q4f6-jm68-57ww, CVE-2026-59899 HttpContentEncoder pipelining queue growth](https://github.com/netty/netty/security/advisories/GHSA-q4f6-jm68-57ww){:target="_blank"} - Netty on GitHub
- [Netty GHSA-p85m-gvr3-788c, CVE-2026-50010 hostname verification disabled on OpenSSL client path](https://github.com/netty/netty/security/advisories/GHSA-p85m-gvr3-788c){:target="_blank"} - Netty on GitHub
- [Netty GHSA-q9pg-8h3j-8hvm, HTTP/3 authority and host mismatch](https://github.com/netty/netty/security/advisories/GHSA-q9pg-8h3j-8hvm){:target="_blank"} - Netty on GitHub
- [Netty GHSA-558v-64gr-wgg4, CVE-2026-59901 Bzip2Decoder infinite loop](https://github.com/netty/netty/security/advisories/GHSA-558v-64gr-wgg4){:target="_blank"} - Netty on GitHub
- [Netty GHSA-pq4x-537v-r54q, CVE-2026-44891 SmtpResponseDecoder memory exhaustion](https://github.com/netty/netty/security/advisories/GHSA-pq4x-537v-r54q){:target="_blank"} - Netty on GitHub
- [Netty netty-codec-http on Maven Central, 4.1.138.Final and 4.2.18.Final confirmed](https://repo1.maven.org/maven2/io/netty/netty-codec-http/){:target="_blank"} - Maven Central
- [Apache Storm storm-client on Maven Central, 3.1.0 confirmed](https://repo1.maven.org/maven2/org/apache/storm/storm-client/){:target="_blank"} - Maven Central
- [Apache ZooKeeper on Maven Central, 3.9.6 and 3.8.7 confirmed](https://repo1.maven.org/maven2/org/apache/zookeeper/zookeeper/){:target="_blank"} - Maven Central
- [Apache Syncope core on Maven Central, 4.0.8 and 4.1.3 confirmed](https://repo1.maven.org/maven2/org/apache/syncope/syncope-core/){:target="_blank"} - Maven Central
- [Apache OpenNLP tools on Maven Central, 2.5.12 confirmed](https://repo1.maven.org/maven2/org/apache/opennlp/opennlp-tools/){:target="_blank"} - Maven Central
- [Java vulnerabilities of the week: 4 to 10 September 2026](/2026/09/10/java-vulnerabilities-of-the-week-4-10-september-2026.html){:target="_blank"} - oscerd.github.io
- [The CVE stigma, we need a new security culture](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html){:target="_blank"} - oscerd.github.io
