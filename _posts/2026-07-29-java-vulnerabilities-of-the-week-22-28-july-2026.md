---
layout: post
title: "Java vulnerabilities of the week: 22 to 28 July 2026"
---

I read a lot of security news and most of it is useless. Not wrong, just useless: a CVE number, a CVSS score, a vendor advisory link, "patch immediately". You close the tab and you have learned nothing. The interesting part of a vulnerability is almost never the score, its the mechanism. How does a broker end up skipping an ACL check because you gave a destination the right kind of name? What happens when the library whose entire job is to answer yes or no gets `equals()` backwards?

So I am going to try something. Once a week I will list the vulnerabilities in the Java ecosystem that caught my attention, then pick a couple and dig into them properly: the advisory, the mechanism, and what I think the actual lesson is. Not a feed, there are better places for that.

This week has a theme, and I did not go looking for it. Almost everything that landed between 22 and 28 July was an authorization failure of one kind or another. A broker that forgot to check an ACL. An authorization policy engine with three separate ways to get the wrong answer. A SOAP stack accepting serialized objects from anybody who can reach a port. It was a bad week for saying no.

## The week in five

- **CVE-2026-66713, Apache Axis2/Java, CVSS 9.8, 28 July.** Unsafe Java deserialization in the Tribes-based clustering component. An unauthenticated attacker with network access to the clustering port sends a crafted serialized object, it gets deserialized in `Axis2ChannelListener#messageReceived`, and that is remote code execution. Clustering is off by default, which is the only reason this is not a much bigger story. The fix in 2.0.1 is to remove the clustering feature entirely. Not harden it, remove it.

- **CVE-2026-61487, Apache ActiveMQ, 27 July.** An authenticated low-privilege user bypasses per-destination write ACLs by sending to a temporary composite destination whose physical name is a comma-separated list of real queues. Rated Important. Affects everything before 5.19.9 and 6.0.0 through 6.2.7, fixed in 5.19.9, 6.2.8 and 6.3.0. Deep dive below, and look at the credit line while you are there.

- **CVE-2026-55771, CVE-2026-55772 and CVE-2026-55773, Cedar-Java, CVSS 8.8 each, 28 July.** Three vulnerabilities in the Java binding for the Cedar authorization policy language: a policy injection, a type confusion across the Java to Rust boundary, and an incorrect equality comparison. Affects everything before 2.3.6, 3.1.2 through 3.4.0, and 4.0.0 through 4.8.9. Fixed in 2.3.6, 3.4.1 and 4.9.0. Deep dive below.

- **CVE-2026-66390 and CVE-2026-66391, Apache Wicket, 27 July.** The first is an XSS: crafted `Link` URL strings can break out of the surrounding JavaScript sequence, because the URL is interpolated into a script context without being neutralised for that context. Rated Important, affects 9.0.0 through 9.23.0 and 10.0.0 through 10.9.0, fixed in 10.10.0. The second, filed the same day by the same reporter, is leaked and missing CSP headers. I like that pairing a lot: one bug lets you inject script, the sibling bug is the defence in depth that should have limited the damage not being properly applied. They found the hole and then found that the safety net had a hole too.

- **CVE-2026-59878, Apache ActiveMQ AMQP, 27 July.** Improper input validation in the AMQP NIO connector: an unauthenticated remote attacker sends a negative frame size value, NIO threads crash, and the thread pool gets exhausted. Moderate severity, same affected and fixed versions as the ACL bug above, reported by zx (Jace). A negative length field crashing a network protocol parser in 2026 is a comforting reminder that the classics never really go away.

## ActiveMQ: the name is not the resource

Some background, because this bug only makes sense if you know two ActiveMQ features that are individually reasonable.

The first is composite destinations. ActiveMQ lets you address more than one destination with a single name by comma separating them, so publishing to `A,B,C` fans the message out to queues A, B and C. It is a genuinely useful thing for integration work and I have used it plenty.

The second is temporary destinations. These are created on the fly, typically for request and reply patterns where a consumer needs somewhere to receive the answer. Because they are owned by the connection that created them and disappear when it goes away, they are treated differently by the authorization layer. You do not write an ACL entry for a queue that will exist for 200 milliseconds and then never again.

Put the two together and you get the bug. Create a destination that is marked temporary but whose physical name is `A,B,C`, where A, B and C are real queues you have no write permission on. The broker looks at the destination, sees the temporary flag, and takes the path that skips the per-destination write ACL. Then the composite expansion runs and your message is delivered to all three queues.

The authorization decision was made about the wrapper, and the delivery was performed on the expansion. A composite destination is not one destination, its N destinations wearing a single name, and the check happened while it still looked like one thing.

### Why I find it interesting

The generalisable lesson is that you have to authorize the effect and not the label. Any time a system has a naming mechanism that expands, aliases, globs or otherwise resolves one identifier into several resources, the authorization check has to happen after resolution, on the actual set. Checking before is checking a string. This is the same shape as path traversal (the path you validated is not the file you opened), as SSRF via redirect (the host you allowed is not the host you fetched), and as the Tomcat and Reactor Netty redirect bugs from earlier this year where the credential was authorized for one host and then sent to another. The resource you checked has to be the resource you touched, and every layer of indirection between those two moments is a place for them to diverge.

The other detail worth pausing on is the credit line. The advisory credits the discovery to **Claude and Ada Logics**.

Ada Logics is a security firm that does a lot of open source auditing and fuzzing work, much of it funded through OpenSSF and CNCF programmes. So what that line means in practice is a professional audit team using an AI model as part of their workflow, and the Apache Security Team crediting both in the advisory as a matter of routine. No announcement, no blog post, no controversy. Just a line in an advisory on oss-security.

I find that more significant than any single one of these bugs. Back in February I wrote about [the CVE stigma and how AI-assisted discovery was going to change the volume and the culture around vulnerability reporting](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html), and at the time the examples were still notable enough to be news. Five months later it is an unremarkable attribution in a routine Apache advisory. Steve Poole made a similar point recently after [seven jackson-databind CVEs landed in a single day](https://foojay.io/today/7-new-vulnerabilities-in-jackson-in-one-day-this-is-what-ai-assisted-security-research-looks-like/){:target="_blank"} from AI assisted analysis, and his line has stayed with me: the absence of CVEs was never evidence of safety, it was evidence of silence.

For a project like ActiveMQ this is fine, it has a real security team and a release process. My worry, and I have said this before, is the enormous middle of the ecosystem that has neither. The rate of finding is going up much faster then the capacity to triage and fix.

## Cedar-Java: three ways to get the wrong answer

Cedar is the authorization policy language AWS open sourced. The engine is written in Rust and there are bindings for other languages, of which CedarJava is the Java one. You describe who can do what in Cedar's policy language, you hand the engine a request, it tells you permit or forbid. It exists specifically so that you do not have to hand-roll authorization logic, which is exactly the kind of code everybody gets wrong.

Three CVEs landed on 28 July, all rated 8.8, all in the Java binding rather then the Rust core.

**CVE-2026-55773, policy injection.** Improper input handling lets attacker-controlled data end up influencing the policy itself rather then just being data the policy is evaluated against. This is the authorization equivalent of SQL injection and the mitigation is the same as it has been for twenty five years: policies are code, user input is data, and the two must never be joined with string concatenation.

**CVE-2026-55772, type confusion.** This one is the most interesting technically. Cedar's serialization format, the one used to pass values from Java across the FFI boundary into the Rust evaluator, reserves two JSON key names: `__entity` and `__extn`. They mark a value as an entity reference or an extension value rather then a plain record. CedarJava did not validate that keys in a `CedarMap` avoided those reserved names. So if your service builds a `CedarMap` from anything user-controlled, and request headers or resource metadata are the obvious candidates, an attacker who can choose a key name can inject `__entity` and get the Rust evaluator to interpret a plain record as an entity reference. The conditions are that the integrating service builds a map from user-controlled data and a policy references that value in a condition, which is not exotic.

**CVE-2026-55771, incorrect equality comparison.** The advisory itself is vague, it says only that under certain circumstances this could lead to incorrect equality comparisons. Secondary reporting attributes it to inverted logic in `EntityIdentifier.equals()`, returning `true` when the argument is null and `false` in cases where it should return `true`. I have not verified that against the source, so treat the specific mechanism as reported rather then confirmed, but the shape is clear enough.

### Why I find it interesting

Start with the obvious one and then the useful one.

The obvious one is the irony, and I do not think it is only irony. Cedar exists so that applications stop writing their own authorization logic, on the entirely correct theory that hand-rolled authorization is where bugs live. And then the binding layer ships a policy injection, a type confusion and a broken `equals()`. This is not an argument against using a policy engine, you should absolutely use one. It is a reminder that adopting a library moves the risk, it does not delete it, and the place it moves to is the glue you wrote around the library plus the glue the library wrote around its own core.

The useful lesson is about the type confusion, and it is one I think Java developers systematically underrate: **a reserved key in a user-controlled map is an injection vector.** `__entity` and `__extn` are in-band signalling. They are metadata living in the same namespace as the data, distinguished only by a naming convention, and a naming convention is not a boundary. This is precisely the shape of prototype pollution in JavaScript with `__proto__`, and of every YAML tag and JSON `$type` deserialization bug we have all been fixing for years. If your serialization format has magic keys and you build objects in that format out of anything that came from a request, you have to filter for the magic keys. The library should have done it here, but the integrating service is the one holding the untrusted input.

The second lesson is that the FFI boundary is a trust boundary and it does not look like one. Java to Rust in the same process feels like a function call, so nobody treats it like a protocol. But it is a protocol, with a wire format and a parser on the other side, and the Rust evaluator trusted the Java side to have produced well-formed input. Everything we know about validating input at the edges applies to internal boundaries too, and internal boundaries are much easier to forget about precisely because they do not have a socket in front of them.

## What I take away from this week

Two things, plus a note about timing.

The first is that all three of the serious bugs this week are the same failure at different altitudes: a decision was made about a representation rather then about the thing it represents. ActiveMQ authorized a destination name instead of the queues it expanded into. Cedar interpreted a record as an entity because it carried the right key. Axis2 turned bytes into live objects because they arrived on the clustering port and the port was the only credential. In each case the check was performed against something that stood in for the real resource, and the substitution was attacker-influenced.

The second is the pattern I noticed last week too and which is becoming a genuine trend: deletion as a fix. Axis2's answer to a deserialization RCE in clustering is to remove clustering in 2.0.1. Logback did the same thing in June, taking out Janino-based conditional configuration entirely rather then patching the denylist for the fourth time. When a capability's security history reads as a list instead of an incident, taking the feature out is usually the honest answer, even though somebody's upgrade gets painful.

The note about timing: none of this will reach most Java teams this week. It reaches you when a BOM moves. Camel, Spring Boot, Quarkus and everything else pin these versions transitively, so the practical arrival date for the ActiveMQ and Wicket fixes is whenever the next framework release picks them up, and for anything on an EOL line it is never. If you run ActiveMQ directly, though, 5.19.9 or 6.2.8 is worth doing on your own schedule rather then waiting. An authorization bypass available to any authenticated user is exactly the kind of thing that sits quietly in an environment where you assumed the ACLs were doing the work.

See you next week.

## References

- [CVE-2026-61487: Apache ActiveMQ authorization bypass via temporary composite destinations](https://seclists.org/oss-sec/2026/q3/288){:target="_blank"} - oss-security
- [CVE-2026-59878: Apache ActiveMQ AMQP NIO negative frame size validation bypass](https://www.openwall.com/lists/oss-security/2026/07/27/7){:target="_blank"} - oss-security
- [ActiveMQ Classic security advisories](https://activemq.apache.org/components/classic/security){:target="_blank"} - Apache ActiveMQ
- [CVE-2026-66390: Apache Wicket, crafted Link URL strings can break out of the JavaScript sequence](https://www.openwall.com/lists/oss-security/2026/07/27/4){:target="_blank"} - oss-security
- [CVE-2026-66391: Apache Wicket, leaked and missing CSP headers](https://www.openwall.com/lists/oss-security/2026/07/27/5){:target="_blank"} - oss-security
- [CVE-2026-66713: Apache Axis2/Java deserialization of untrusted data](https://www.cve.org/cverecord?id=CVE-2026-66713){:target="_blank"} - CVE Program
- [CVE-2026-55771: Cedar-Java policy injection, type confusion and incorrect equality comparison](https://nvd.nist.gov/vuln/detail/CVE-2026-55771){:target="_blank"} - NVD
- [CVE-2026-55772: CedarJava type confusion](https://github.com/advisories/GHSA-93g4-m6xv-cmvr){:target="_blank"} - GitHub Advisory Database
- [CVE-2026-55773: CedarJava policy injection](https://advisories.gitlab.com/maven/com.cedarpolicy/cedar-java/CVE-2026-55773/){:target="_blank"} - GitLab Advisory Database
- [7 New Vulnerabilities in Jackson in One Day: This Is What AI-Assisted Security Research Looks Like](https://foojay.io/today/7-new-vulnerabilities-in-jackson-in-one-day-this-is-what-ai-assisted-security-research-looks-like/){:target="_blank"} - Steve Poole, foojay.io
