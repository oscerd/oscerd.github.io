---
layout: post
title: "Java vulnerabilities of the week: 29 July to 4 August 2026"
---

Second week of this. The idea is the same as last time: list what caught my attention in the Java ecosystem, then pick two and look at the mechanism, because the CVE number and the score never tell you anything you can use.

Between 29 July and 4 August two unrelated projects, a cryptography library and a messaging stack, disclosed around sixty CVEs between them. Line up the titles and a lot of them are the same sentence. Allocation from a declared length. An iteration count read out of a file. Recursion with no depth limit. A count field with no ceiling.

So the theme this week is what happens when the input gets to decide how much work you do.

## The week in five

- **Bouncy Castle 1.85, 32 CVEs, released 28 July, reached oss-security 3 August.** The JCE provider a large fraction of Java crypto quietly runs on. Thirty two CVE identifiers in one release, across ASN.1 parsing, OpenPGP, DTLS, PKCS#12 and BCFKS keystores, CMS, S/MIME, JSSE hostname verification and MLS. The release is one day before my window opens, so I am cheating slightly, but the disclosure and the discussion both happened inside it. Deep dive below.

- **Apache Qpid: 24 CVEs across four AMQP 1.0 implementations, 4 August.** Six in Proton-J (through 0.34.1, fixed in 0.35.0), seven in Broker-J (through 10.0.1, fixed in 10.1.0), five in ProtonJ2 (through 1.1.0, fixed in 1.2.0) and six in Proton Dotnet. Most rated Important, most reachable pre-authentication, and really about six distinct issues repeated across the four codebases. Deep dive below.

- **CVE-2026-66755, Apache Tika, 30 July.** Relative path traversal in the ISA-Tab parser. The `Study Assay File Name` values inside an investigation file are not validated, so a crafted dataset reads files outside its own directory with the privileges of the Tika process. Affects `tika-parser-scientific-module` from 1.8 through 3.3.1 and 4.0.0-alpha-1, fixed in 3.3.2 and 4.0.0-beta-1. Credited to BugQore, who also supplied the patch, with an independent discovery by Rui Heng Koh. A sibling issue, CVE-2026-66756, covers the `unpack` endpoint in `tika-server` being reachable with `unsecureFeatures=false`.

- **Four CVEs in Apache NiFi, 3 August**, all fixed in 2.11.0. Three are authorization issues around Parameter Contexts: CVE-2026-62354 (High) let a read-only user submit proposed parameter values and trigger component validation with them, CVE-2026-68979 (Medium) skipped authorization on the components referencing a parameter when the context was updated, and CVE-2026-68980 (Low) authorized asset deletion against the Parameter Context identifier supplied in the request rather then the stored one. The fourth, CVE-2026-68981 (High), is this week's theme in miniature: the Jersey encoding filter applied size limits to the compressed request body instead of the decompressed output.

- **Four CVEs in Apache Zeppelin, 30 July.** The interesting one is CVE-2026-44617, an LDAP filter injection in `LdapRealm`. The input is escaped, just against the wrong specification: RFC 4514 distinguished-name escaping applied to a value going into a search filter, which needs RFC 4515 filter escaping. Two grammars, two different sets of metacharacters. Moderate, affects 0.11.1, 0.11.2 and 0.12.0, fixed in 0.12.1, and explicitly an incomplete fix of CVE-2024-31867. The others are a CSRF in REST and WebSocket handling, a path traversal in `NotebookRepo`, and a separate LDAP injection in `ActiveDirectoryGroupRealm`.

Honourable mention to **CVE-2026-62391 in Apache Kyuubi**, 31 July. Kyuubi has an allowlist, `kyuubi.session.local.dir.allow.list`, restricting which local directories a session can touch, and you get past it with unprefixed Spark config aliases instead of the properly prefixed names: the check reads the name you used, not the setting you end up modifying. Affects 1.6.0 up to 1.12.0. Same shape as the ActiveMQ composite destination bug I wrote about last week, where the authorization decision was made about a label and the effect happened on what the label expanded to.

## Bouncy Castle: thirty two CVEs in one release

Bouncy Castle sits under a lot of Java that nobody thinks of as crypto code: PKI, signing, S/MIME, OpenPGP, TLS. If you ever needed an algorithm the JDK did not ship, you added `bcprov` and moved on. It is also where post-quantum cryptography currently lives for Java, which is why I read its source properly a few months ago while writing about [how Camel is preparing for post-quantum cryptography](/2026/05/06/apache-camel-is-preparing-for-post-quantum-cryptography.html).

Version 1.85 shipped on 28 July with expanded NIST and Korean post-quantum algorithms, hybrid certificates for quantum-safe PKI migration, a high-level CAdES API, KEM-based key management in CMS, and BLS12-381 and Taproot Schnorr signatures. It also fixes thirty two CVEs, and read as a set they fall into two groups.

The first group is this week's theme. `BKS/UBER keystore allocates from untrusted lengths before integrity check`. `Possible OOM from unbounded up-front allocation on definite-length read`. `PKCS#8 / PBES2 decryptors honour unbounded KDF cost from input`. `BCFKS keystore load honours unbounded KDF cost from untrusted file`. `OpenPGP Argon2 S2K honours attacker-chosen memory and passes`. `HSS public-key level count unbounded, enabling huge allocation on verify`. `Lazy ASN.1 sequence forcing resets nesting-depth guard`. `DTLS handshake reassembler allocates buffer from unchecked 24-bit length`.

The keystore one is worth pausing on. The point of a keystore MAC is to establish that the file has not been tampered with, so allocating on lengths read out of the file before that MAC is verified puts the integrity check after the expensive part. It is the encrypt-then-MAC principle in allocation terms: "allocate a buffer this big" is already acting on the data.

The second group is different in kind. `CMS verifySignatures returns true for SignedData with zero signers`. `RSA PKCS#1 verification skips last two hash bytes in NULL-omitted path`. `CCM-family modes write plaintext to caller buffer before tag check`. `Stapled OCSP response accepted without binding to the checked certificate`. `S/MIME validator trusts signer-asserted signingTime for path validation`. `JSSE hostname verifier CN-fallback enabled by default despite documented opt-in`. Those are not resource issues, those are the primitive returning the wrong answer.

The zero signers one takes ten seconds to explain. `verifySignatures` walks the signers, checks each, and reports success if none failed. Hand it a `SignedData` with no signers and nothing fails, so it returns true. Vacuous truth: a universal statement over an empty set holds, and validation phrased as "no element failed" rather then "at least one passed and none failed" lets empty input through. Same shape as the Cedar-Java `equals()` issue from last week.

### Why I find it interesting

Two things, and neither is the individual bugs.

The first is the split between the two groups, because they need different responses from you. The allocation ones are availability: bad, but they fail loudly and you can often bound them at a layer above. The second group is silent. A signature check returning true, a hostname verifier falling back to CN, plaintext handed to your buffer before the tag is checked. Nothing throws, nothing logs, and the security property you designed around is simply not there. If you are triaging thirty two CVEs with limited time, that distinction is the one that matters, not the severity ordering.

The second is the attribution, and specifically how hard it is to find.

Alan Coopersmith posted the release to oss-security on 3 August with the CVE list. Peter Gutmann replied the next day asking whether some new code analysis tool was behind the volume, guessing AI and noting there would be an interesting backstory. Coopersmith answered that he had not seen anything in the announcements from the Bouncy Castle folks about that.

The answer is in the announcement. Version 1.85 "represents a significant hardening of the APIs as a result of the help the team has received from people using advanced AI-based coding analysis tools." That is the tail of a sentence whose subject is standards coverage and preparing for the next generation of cryptographic security, in a paragraph selling the release. Two people read that page carefully enough to post it to a security list and ask a question about it, and the sentence answering the question did not register with either of them.

I do not think that is anybody being careless. It is a question of where the information lives. Thirty two CVEs, and the only statement of how they were found is prose in a feature announcement, so it is not in the advisories, not in the per-CVE credits, not in anything a tool or a reader scanning the CVE list would pick up. Compare last week's ActiveMQ advisory, which carried "Claude and Ada Logics" in the credit field, where credit normally goes: structured, per-CVE, and impossible to miss. Same category of fact, and only one of the two formats survives contact with a reader.

That matters more as the volume grows. If AI-assisted analysis is going to be a significant share of how bugs get found, the provenance is worth recording where provenance is recorded, and it is the sort of thing I had in mind writing about [the CVE stigma](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html) back in February. A count of thirty two mostly tells you how closely a library has been looked at. The how is the part that lets you interpret the count, and this week it was there and still effectively unreadable.

## Apache Qpid: when the wire decides the allocation

On 4 August Qpid published twenty four advisories at once. Unbounded symbol value caching appears as CVE-2026-66257 in Proton-J, CVE-2026-68074 in Broker-J, CVE-2026-67588 in ProtonJ2 and CVE-2026-67465 in Proton Dotnet. Type size and count handling leading to excessive allocation appears as CVE-2026-66273, CVE-2026-68060, CVE-2026-67589 and CVE-2026-67551. Unbounded type nesting leading to a `StackOverflowError` appears as CVE-2026-66274, CVE-2026-68073, CVE-2026-67590 and CVE-2026-67552. Flow control windows, disposition ranges and transfer frames per delivery repeat across the same set. Robbie Gemmell reported on Proton-J, Daniil Kirilyuk on Broker-J, Timothy A. Bish on ProtonJ2.

One fact about AMQP 1.0 explains all of it. It is a self-describing binary type system: every value is a constructor byte saying what type follows, then a size or count field for anything variable-length, then the bytes. A list is a constructor, a size, a count of elements, then the elements, each of which is itself a constructor with possibly another size and count. Symbols, the interned ASCII strings AMQP uses for names, get cached by the decoder so repeated names cost nothing.

Each of those is a reasonable design, and each hands the sender a lever.

The size field is the peer telling you how big to make the buffer, so allocate first and read second and a four-byte field asks for four gigabytes. The count field is the peer telling you how many elements to expect, so a preallocated array is sized by the sender even when the payload is a hundred bytes. A list can contain a list, and a recursive descent decoder turns nesting depth into stack depth. Symbol caching inserts a sender-chosen key into a map that never evicts.

The timing matters as much as the mechanism. AMQP negotiates its protocol header, opens a connection and exchanges performatives before SASL authentication completes, so the decoder is parsing peer-controlled structures before anyone has proven who they are. That is why most of these are reachable pre-authentication.

### Why I find it interesting

Because of where the constraint actually lives.

Four implementations, two of them different generations of the same idea, one a broker, one on a different runtime with a different memory model, written at different times. The same six issues turn up in all of them. That is a good signal that the thing to fix is not any one decoder but the assumption the format encourages: AMQP 1.0 makes the wire authoritative on how much memory to allocate and how deep to recurse, and the spec does not require an implementation to impose a ceiling, so the natural reading of the document produces a decoder without one.

That puts it in a bug class family Java developers already know. XML entity expansion, where the document says how many times to expand. Zip bombs, and NiFi's gzip filter from the same week, where the compressed size is a lie about the decompressed size. ASN.1 length prefixes, which is where a third of the Bouncy Castle CVEs live. `ObjectInputStream` reading a declared array length and allocating it before reading an element. The rule is short: **a length field is a request, not a fact.** Anything from an untrusted source that controls allocation size, loop iterations, recursion depth or cache growth needs a ceiling that comes from your configuration.

The one I would go and look at in my own code is the symbol cache, because it is the least obvious. A memoisation cache does not look like an attack surface, its an optimisation. But an unbounded cache keyed on peer-supplied input grows on demand, and Java is full of them: interned strings, class-name to metadata maps, regex pattern caches, JSON field-name caches. If the key comes from the network and the map has no eviction policy, you have a slow OOM with extra steps.

## What I take away from this week

The first is a checklist rather then a moral. Find the places where a number from outside decides how much work happens. Buffer allocation from a declared length. Array preallocation from a declared count. Loop bounds from an iteration count in a file, which is the PKCS#12, PBES2 and BCFKS case. Recursion driven by input structure. Cache insertion keyed on input. Decompression limited on the compressed side. Every serious item in this post is one of those six.

The second is that two items this week, Zeppelin and Kyuubi, were labelled as incomplete fixes of earlier CVEs, and both are cases where the check and the effect are separated by a translation step. Zeppelin escapes, but the value crosses from one LDAP grammar into another between the escaping and the query. Kyuubi checks a config name, but alias resolution happens after the check. When something gets renamed, rewritten or expanded between validation and use, that gap is where the next report comes from, and it is worth looking for deliberately.

The third is about arrival times. Bouncy Castle arrives through whatever pulls in `bcprov`, which for many people is Keycloak or a PDF signing library. Qpid Proton-J arrives through the ActiveMQ and Artemis AMQP clients, through Camel's AMQP component, through anything speaking AMQP 1.0 from the JVM. Tika arrives through Solr, through Camel, through most content extraction pipelines. You get these fixes when a BOM moves, and on an unmaintained line you do not get them at all.

Bouncy Castle is the one I would not wait on. The resource exhaustion half can ride a BOM bump, but the correctness half changes what your verification calls actually mean. If you pin `bcprov` directly, 1.85 is worth doing on your own schedule.

See you next week.

## References

- [Bouncy Castle 1.85 release fixes 32 CVEs](https://www.openwall.com/lists/oss-security/2026/08/04/2){:target="_blank"} - Alan Coopersmith, oss-security
- [Re: Bouncy Castle 1.85 release fixes 32 CVEs](https://seclists.org/oss-sec/2026/q3/403){:target="_blank"} - Peter Gutmann, oss-security
- [Re: Bouncy Castle 1.85 release fixes 32 CVEs](https://www.openwall.com/lists/oss-security/2026/08/04/7){:target="_blank"} - Alan Coopersmith, oss-security
- [New Release: Bouncy Castle Java 1.85](https://www.bouncycastle.org/resources/new-release-bouncy-castle-java-1-85/){:target="_blank"} - Bouncy Castle
- [CVE-2026-66257: Apache Qpid Proton-J, unbounded symbol value caching](https://www.openwall.com/lists/oss-security/2026/08/04/8){:target="_blank"} - oss-security
- [CVE-2026-66273: Apache Qpid Proton-J, type size/count handling can lead to excessive allocation](https://www.openwall.com/lists/oss-security/2026/08/04/9){:target="_blank"} - oss-security
- [CVE-2026-66274: Apache Qpid Proton-J, unbounded type nesting can lead to pre-authentication stackoverflow](https://www.openwall.com/lists/oss-security/2026/08/04/10){:target="_blank"} - oss-security
- [CVE-2026-66275: Apache Qpid Proton-J, incoming session flow control window can be exceeded](https://www.openwall.com/lists/oss-security/2026/08/04/11){:target="_blank"} - oss-security
- [CVE-2026-68060: Apache Qpid Broker-J, type size/count handling can lead to excessive allocation](https://www.openwall.com/lists/oss-security/2026/08/04/14){:target="_blank"} - oss-security
- [CVE-2026-68080: Apache Qpid Broker-J, unbounded echo flow responses can lead to denial of service](https://www.openwall.com/lists/oss-security/2026/08/04/20){:target="_blank"} - oss-security
- [CVE-2026-67588: Apache Qpid ProtonJ2, unbounded symbol value caching](https://www.openwall.com/lists/oss-security/2026/08/04/27){:target="_blank"} - oss-security
- [CVE-2026-66755: Apache Tika, arbitrary local file read in ISArchiveParser](https://www.openwall.com/lists/oss-security/2026/07/30/23){:target="_blank"} - oss-security
- [CVE-2026-66756: Apache Tika, unpack endpoint in tika-server allows configuration with unsecureFeatures=false](https://www.openwall.com/lists/oss-security/2026/07/30/24){:target="_blank"} - oss-security
- [CVE-2026-44617: Apache Zeppelin, LDAP filter injection in LdapRealm, incomplete fix of CVE-2024-31867](https://www.openwall.com/lists/oss-security/2026/07/30/4){:target="_blank"} - oss-security
- [CVE-2026-62391: Apache Kyuubi, allow.list bypass via unprefixed Spark file-conf aliases](https://www.openwall.com/lists/oss-security/2026/07/31/2){:target="_blank"} - oss-security
- [CVE-2026-68979: Apache NiFi, missing authorization for components referenced by Parameter Context updates](https://www.openwall.com/lists/oss-security/2026/08/03/10){:target="_blank"} - oss-security
- [CVE-2026-62354: Apache NiFi, incorrect authorization for Parameter Context validation requests](https://www.openwall.com/lists/oss-security/2026/08/03/11){:target="_blank"} - oss-security
- [CVE-2026-68980: Apache NiFi, authorization bypass for Parameter Context asset deletion](https://www.openwall.com/lists/oss-security/2026/08/03/12){:target="_blank"} - oss-security
- [CVE-2026-68981: Apache NiFi, uncontrolled resource consumption through decompression of HTTP requests](https://www.openwall.com/lists/oss-security/2026/08/03/13){:target="_blank"} - oss-security
- [oss-security daily archive](https://www.openwall.com/lists/oss-security/2026/08/04/){:target="_blank"} - Openwall
