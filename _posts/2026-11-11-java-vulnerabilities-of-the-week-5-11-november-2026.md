---
layout: post
title: "Java vulnerabilities of the week: 5 to 11 November 2026"
---

Security news gives you a number and a score, and a number teaches you nothing you can act on. The score tells you how loud to be in the planning meeting, not what to look for in your own code. The thing that travels from one project to yours is the mechanism, the shape of the mistake, and that is what this series is for. The longer version of the argument is in [the CVE stigma post](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html){:target="_blank"}.

This is 5 to 11 November 2026, and it is an Apache week: James, Axis2/Java, Axis 1.x, Sling and OFBiz all published, with a non-Apache visitor in the HashiCorp Vault Java client. The two most serious items share a single mechanism. In Apache Axis 1.x and in Apache Juneau, a class named in an untrusted message gets loaded, and in Java loading a class is not a passive act. That is the first deep dive. The second is a quieter bug in James, where a mailbox is reachable through IMAP and through JMAP, and a permission you set in one of them was invisible to the other.

## The week in five

- **Apache Axis 1.x (EOL), `CVE-2026-90436`, Critical, disclosed 6 November, no fixed release.** An inbound SOAP `<Fault>` can carry a `<detail><exceptionName>` naming any class on the server's classpath, and Axis loads it and, if a String `<data>` element is present, builds it through its String constructor, all before it checks that the class is actually an `AxisFault`. Loading alone runs the class's static initializer, and the fault is parsed while the engine is still resolving the target service, so it happens before any authentication handler on that service runs. With the right class present this is unauthenticated remote code execution (first deep dive). Axis 1 is end of life and the project does not plan a release, so the fix is commit `41f3b4e8` applied to your own build, and the practical move is to migrate to a maintained SOAP stack such as Axis2.

- **Apache Juneau, `CVE-2026-73098`, Critical (CVSS 4.0 10.0), CVE record 5 November, fixed in 9.2.1.** Last week's post noted the 9.2.1 release and its nine fixes; this is the issue behind the headline, now that the record is public. When Juneau parses a value into a property of type `java.lang.Class`, including parameterised forms, the default class swap loaded the named class with initialisation enabled, so a payload putting a classpath class name where a `Class` was expected ran that class's static initializer. Its the same family as the Axis bug above, CWE-470, and dangerous for the same reason. `org.apache.juneau:juneau-marshall` through 9.2.0, fixed in 9.2.1, with 9.2.2 the current release.

- **Apache James, seven CVEs disclosed 6 November, fixed in 3.9.1 (MIME4J 0.8.15).** The one I dig into below is `CVE-2026-49939` (Moderate): a right you removed from a user with a negative IMAP ACL was still granted when the same mailbox was reached over JMAP. Alongside it, `CVE-2025-66621` (Moderate) let a token through because the SASL `OAUTHBEARER`/`XOAUTH2` path did not validate the JWT `aud` claim when introspection was not configured; `CVE-2026-76168` (Important) is a path traversal in `SieveFileRepository` that let a logged-in user write Sieve scripts into another user's space, which means silent mail redirection or a denial of service; and `CVE-2026-74756` (Low) is the project's Docker Compose exposing the WebAdmin REST API with no authentication, which 3.9.1 closes by generating a random credential by default. A JMAP push-subscription SSRF (`CVE-2026-76171`) rounds out the server set. Two more, `CVE-2026-85108` (Important) and `CVE-2026-85111` (CVSS 4.0 8.7), are the same unbounded-allocation denial of service in the MIME4J parser; if you depend on `org.apache.james:apache-mime4j-core` directly, as a lot of mail-handling code does, upgrade it to 0.8.15 whether or not you run James.

- **Apache Axis2/Java, seven CVEs disclosed 6 November (CVE records 9 November), fixed in 2.0.2.** The one to act on is `CVE-2026-120761` (Important): when a client builds an `AxisService` from a WSDL URL, through `ServiceClient` or `AxisService.createClientSideAxisService`, `wsdl4j` resolves the documents named by `wsdl:import` and `xsd:import` with its own XML parser, which honours `DOCTYPE` and external entities, so whoever controls that WSDL or anything it imports can read local files or make the JVM fetch internal URLs. The rest are two more server-side request forgeries (via WS-Addressing `ReplyTo` and `FaultTo`, `CVE-2026-123753`, and via schema import resolution, `CVE-2026-120832`), unbounded multipart and JSON bodies (`CVE-2026-123770`, `CVE-2026-120367`), reflected XSS in the Swagger UI page (`CVE-2026-123766`), and metadata routes that ignored the `exposeServiceMetadata` setting (`CVE-2026-123779`). All fixed in 2.0.2.

- **Apache Sling Engine, three CVEs disclosed 5 November, fixed in 3.0.4.** An included component could change the response `Content-Type`, charset or redirect through `setHeader()`, repeated `setContentType()` calls, the charset APIs or the Servlet 6.1 redirect overloads, and so slip past the `protectheaders` and `checkcontenttype` enforcement that is meant to stop stored XSS (`CVE-2026-103863`). The engine's own query and form parser accepted non-standard percent-escapes and wrong charsets and continued processing on the errors, which diverges from the container's parsing and opens request smuggling (`CVE-2026-103866`). And request-derived values reached log sinks without CR/LF neutralisation, a log-injection primitive (`CVE-2026-103864`). The advisories name the Apache Software Foundation as the finder and Claude Code as the tool used. All three fixed in 3.0.4.

Two outside the web-services pile. The `vault-java-driver` client (`io.github.jopenlibs:vault-java-driver`, 5.2.0 through 6.2.3) had two issues disclosed on 5 November and fixed in 7.0.0. `CVE-2026-109408` is the one worth remembering: the driver silently turns off TLS certificate and hostname verification in three situations where the caller actually asked for it, an `SslConfig` used without calling `build()`, an `SslConfig` restored by Java deserialization, and `VAULT_SSL_VERIFY` set to a truthy value that is not exactly `true`, such as `1`, `yes` or `on`. In each case `SslConfig.isVerify()` returns false and the driver installs a trust-all `SSLContext`, so anyone on the path to Vault reads the token and secrets in transit and can forge responses. `CVE-2026-112072` is a stack-overflow denial of service from the vendored JSON parser having no nesting limit. And Apache OFBiz shipped `24.09.08` on 5 November for four issues, including a server-side request forgery whose fetch runs before the screen's permission check (`CVE-2026-82902`) and a `use-when` expression that escapes the Groovy sandbox (`CVE-2026-82960`).

## Apache Axis 1.x and Juneau: loading a class is already running code

In SOAP a server reports an error by sending a `<Fault>`, and Axis 1 lets a fault carry a custom exception type so a client can rebuild the server's own exception object. The type travels in `<detail><exceptionName>`, as a fully qualified Java class name, and a constructor argument travels in a sibling `<data>` element as a string. On the receiving side Axis turns those back into an object.

Here is the receiving code, before the fix. `SOAPFaultDetailsBuilder.setValue()` handles the `exceptionName`:

```java
} else if ("exceptionName".equals(hint)) {
    String faultClassName = (String) value;
    try {
        Class faultClass = ClassUtils.forName(faultClassName);
        builder.setFaultClass(faultClass);
    } catch (ClassNotFoundException e) {
        // Just create an AxisFault, no custom exception
    }
}
```

`ClassUtils.forName(faultClassName)` loads the class the message named. Nothing checks first that the name refers to an `AxisFault`, or to anything in particular. Then `SOAPFaultBuilder.createFault()` builds it:

```java
Class argClass = ConvertWrapper(faultData.getClass());
try {
    Constructor con = faultClass.getConstructor(new Class[] { argClass });
    f = (AxisFault) con.newInstance(new Object[] { faultData });
} catch(Exception e){
    // ... fall through and use a plain AxisFault
}
```

The cast to `AxisFault` is on the same statement as the `newInstance`, so the constructor runs first and the cast is checked after. If the class is not an `AxisFault`, the `ClassCastException` lands in the catch and is dropped, but by then the constructor has already run.

So two things happen to an attacker-named class before anyone has asked whether they should. Loading it runs its static initializer, whatever that does. And if it has a public constructor taking a single `String`, that constructor runs with the attacker's `<data>` string. The commit that fixes this names the gadget: Spring's `org.springframework.context.support.ClassPathXmlApplicationContext`, if it is on the classpath, has a constructor that takes a string location of a bean definition file and, in constructing, loads that file and instantiates the beans it declares. Point the `<data>` string at a URL you control and that is remote code execution. Readers will recognise it as the gadget behind the ActiveMQ OpenWire RCE, `CVE-2023-46604`.

The part that makes it unauthenticated is where in the pipeline this sits. The fault is parsed while the Axis engine is still resolving which service the request is for, before the per-service handler chain, which is where an authentication handler would live. The class is loaded and built before the request reaches the point that would reject an anonymous caller.

The fix, commit `41f3b4e8`, does the plain thing in two places. It loads the class without initialising it, using the non-initialising overload of `ClassUtils.forName`, so naming a class no longer runs its static initializer, and it checks `AxisFault.class.isAssignableFrom(faultClass)` before it keeps the class or calls a constructor. A real custom `AxisFault` subtype still deserialises; anything else is ignored.

Juneau the same week is the short version of the identical idea. When its marshaller parses a value into a property whose declared type is `java.lang.Class`, the default conversion loaded the named class with initialisation on, so a JSON or similar payload that put a classpath class name where a `Class` was expected ran that class's static initializer. `CVE-2026-73098`, CVSS 10.0, fixed in 9.2.1. Same cause, same fix shape.

### Why I find it interesting

The bug class is CWE-470, use of externally controlled input to select classes or code, sometimes called unsafe reflection. It is the cousin of deserialization: you do not need `readObject` to reach dangerous code if you will take a class name from the wire and load or construct it. The thing a Java developer should carry away is that `Class.forName(name)` is not a lookup. The default three-argument behaviour initialises the class, which runs static initializers, and `newInstance` or a reflective constructor runs reachable code. When the name comes from outside, those are side effects, not conveniences.

Three habits follow, and they are exactly what both fixes do. Decide what the class is allowed to be before you load it, by an allow-list or an `isAssignableFrom` against the base type you actually expect. Load without initialising when you only need to reflect on the type, with `Class.forName(name, false, loader)`. And keep the type check ahead of the constructor, never after it behind a cast, because a cast that throws has already let the constructor run. One grep covers a codebase: look for `Class.forName`, `loadClass`, `getConstructor` and `newInstance` where the string argument traces back to a request.

The Axis case adds the end-of-life wrinkle. There will be no Axis 1 release, so the choice is to carry the patch on a build you maintain or to move to Axis2 or another engine. If you do not know whether Axis 1 is in your tree, that is the first thing to find out, because it is the kind of dependency that rode in under an old integration and keeps answering requests.

## Apache James: the deny rule and the protocol that could not see it

James is a mail server, and the same mailbox is reachable through more then one protocol. IMAP clients talk to it, and JMAP clients, over the JSON protocol from RFC 8621, talk to the same mailboxes over HTTP. Sharing a mailbox with another user is an access-control decision, and James keeps it in one place, an access control list on the mailbox.

IMAP's ACL model is RFC 4314, and it has a feature that is easy to miss: negative rights. An entry whose identifier starts with a minus sign does not grant rights, it removes them. You can grant a group read and administer on a folder and then add `-alice a` to take the administer right back from Alice in particular. Alice's effective rights are the rights her positive entries give her, minus the rights her negative entries take away. The deny is a first-class entry in the list, not the absence of a grant.

JMAP exposes the same capabilities as a set of booleans on the mailbox, `mayRead`, `mayAdmin` and the rest, so James has to translate its stored ACL into those booleans for a JMAP caller. `CVE-2026-49939`, reported by mitchell at revampstudio.com.au, is that the translation added up the positive entries and did not subtract the negative ones. A user who had been explicitly denied the administer right with a negative IMAP ACL still saw `mayAdmin` as true over JMAP, and could administer the folder there, the exact thing the deny was meant to stop. Fixed in 3.9.1.

It is worth seeing next to the other James items. The WebAdmin default (`CVE-2026-74756`) is the admin surface reachable with no credential on one deployment path. The `OAUTHBEARER` `aud` gap (`CVE-2025-66621`) is a token accepted on the SASL path that another configuration would have rejected. Each is the same mailbox or the same server, reached through a door that did not enforce what another door did.

### Why I find it interesting

This is improper authorization, CWE-863, and the specific shape is a permission model that two front ends read differently. The lesson is about deny semantics. When your rules can both grant and deny, the deny is the load-bearing half, because it exists to stop something a grant would otherwise allow. A reader that only sums grants is correct for every mailbox with no negative entry, which is most of them, and wrong exactly where someone took the trouble to remove a right. Thats the worst place to be wrong, because the negative entry is there precisely because someone cared.

The review question is concrete. If one resource is reachable through two protocols or two APIs, do not ask whether each enforces access, ask whether they compute the same answer from the same rules, the deny rules included. The cheapest test is to set a deny in one interface and try the action through the other. It generalises past mail: a REST API and a GraphQL API over the same data, a read path and an admin path, an IMAP server and its JSON twin. The store can hold a rule that one of them was never taught to read.

## What I take away from this week

First, a class name on the wire is an instruction. Axis 1.x and Juneau are the same CWE-470 in two codebases, and the OFBiz `use-when` sandbox escape is a relative of it: untrusted input choosing which code runs. If your code turns a string from a request into a `Class`, the questions are whether you checked the type before loading, whether you loaded without initialising, and whether the type check comes before any construction. None of those cost anything, and all three were the fix.

Second, where the fixes reach people. MIME4J is the one that travels: `org.apache.james:apache-mime4j-core` is a transitive dependency of a lot of mail-handling code, so bump it to 0.8.15 directly even if you never deploy James. James itself, Sling Engine, Juneau and Axis2 are direct upgrades for the teams that run them, to 3.9.1, 3.0.4, 9.2.1 and 2.0.2, and all four are on Maven Central; I checked. `vault-java-driver` 7.0.0 is a major-version bump, so read the notes before you move, though the TLS default it corrects is reason enough to. Axis 1.x is the one with no release: find out whether it is in your tree, carry the patch or migrate off it. OFBiz operators upgrade the application to 24.09.08.

Third, a note for the record rather then a lesson. The Sling advisories credit the Apache Software Foundation as the finder and Claude Code as the tool, and the Axis 1.x fix commit lists an Anthropic model as a co-author. AI-assisted review turning up on both the finding side and the fixing side of the same week's advisories is new enough to write down, and I will leave it at the fact.

## References

- [oss-security: Apache Axis 1.x unauthenticated remote class instantiation CVE-2026-90436](https://www.openwall.com/lists/oss-security/2026/11/06/12){:target="_blank"} - Openwall oss-security
- [Axis 1.x fix: prevent arbitrary class instantiation from SOAP fault exceptionName](https://github.com/apache/axis-axis1-java/commit/41f3b4e8d04e310ca81f86b92630ab2c11d608bd){:target="_blank"} - apache/axis-axis1-java on GitHub
- [CVE-2026-73098: Apache Juneau unsafe reflection during parsing](https://www.cve.org/CVERecord?id=CVE-2026-73098){:target="_blank"} - CVE.org
- [oss-security: Apache James SASL OAUTHBEARER aud validation CVE-2025-66621](https://www.openwall.com/lists/oss-security/2026/11/06/13){:target="_blank"} - Openwall oss-security
- [oss-security: Apache James negative IMAP ACL bypass through JMAP CVE-2026-49939](https://www.openwall.com/lists/oss-security/2026/11/06/14){:target="_blank"} - Openwall oss-security
- [oss-security: Apache James WebAdmin exposed on Docker default CVE-2026-74756](https://www.openwall.com/lists/oss-security/2026/11/06/15){:target="_blank"} - Openwall oss-security
- [oss-security: Apache James ManageSieve path traversal CVE-2026-76168](https://www.openwall.com/lists/oss-security/2026/11/06/16){:target="_blank"} - Openwall oss-security
- [oss-security: Apache James JMAP PushSubscription SSRF CVE-2026-76171](https://www.openwall.com/lists/oss-security/2026/11/06/17){:target="_blank"} - Openwall oss-security
- [oss-security: Apache James MIME4J denial of service CVE-2026-85108](https://www.openwall.com/lists/oss-security/2026/11/06/18){:target="_blank"} - Openwall oss-security
- [oss-security: Apache James denial of service upon MIME parsing CVE-2026-85111](https://www.openwall.com/lists/oss-security/2026/11/06/19){:target="_blank"} - Openwall oss-security
- [oss-security: Apache Axis2/Java XXE via imported WSDL/XSD CVE-2026-120761](https://www.openwall.com/lists/oss-security/2026/11/06/22){:target="_blank"} - Openwall oss-security
- [oss-security: Apache Axis2/Java exposeServiceMetadata not enforced CVE-2026-123779](https://www.openwall.com/lists/oss-security/2026/11/06/28){:target="_blank"} - Openwall oss-security
- [Apache Axis2/Java 2.0.2 release notes](https://axis.apache.org/axis2/java/core/release-notes/2.0.2.html){:target="_blank"} - Apache Axis
- [Axis2 fix commit for CVE-2026-120761 (imported WSDL/XSD XXE)](https://github.com/apache/axis-axis2-java-core/commit/1d3457d12399c7721ed3f52e7031d1d24d169460){:target="_blank"} - apache/axis-axis2-java-core on GitHub
- [oss-security: Apache Sling Engine include-protection bypass CVE-2026-103863](https://www.openwall.com/lists/oss-security/2026/11/05/6){:target="_blank"} - Openwall oss-security
- [oss-security: Apache Sling Engine log injection CVE-2026-103864](https://www.openwall.com/lists/oss-security/2026/11/05/7){:target="_blank"} - Openwall oss-security
- [oss-security: Apache Sling Engine parameter parser smuggling CVE-2026-103866](https://www.openwall.com/lists/oss-security/2026/11/05/8){:target="_blank"} - Openwall oss-security
- [CVE-2026-109408: vault-java-driver disables TLS verification](https://www.cve.org/CVERecord?id=CVE-2026-109408){:target="_blank"} - CVE.org
- [CVE-2026-112072: vault-java-driver JSON nesting denial of service](https://www.cve.org/CVERecord?id=CVE-2026-112072){:target="_blank"} - CVE.org
- [CVE-2026-82902: Apache OFBiz SSRF in webtools viewdatafile](https://www.cve.org/CVERecord?id=CVE-2026-82902){:target="_blank"} - CVE.org
- [CVE-2026-82960: Apache OFBiz use-when expression Groovy sandbox bypass](https://www.cve.org/CVERecord?id=CVE-2026-82960){:target="_blank"} - CVE.org
- [Java vulnerabilities of the week: 29 October to 4 November 2026](/2026/11/04/java-vulnerabilities-of-the-week-29-october-4-november-2026.html){:target="_blank"} - oscerd.github.io
- [The CVE stigma, we need a new security culture](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html){:target="_blank"} - oscerd.github.io
