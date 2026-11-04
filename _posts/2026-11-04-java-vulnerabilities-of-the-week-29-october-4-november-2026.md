---
layout: post
title: "Java vulnerabilities of the week: 29 October to 4 November 2026"
---

One of this week's ZooKeeper advisories comes down to a question every SASL server has to answer: this client proved it is one principal, may it act as the one it asked for? The 8.6 score gives you nothing to look for in your own code; the question does. That is the point of this series, the mechanism you can carry into a code review, and the longer argument is in [the CVE stigma post](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html){:target="_blank"}.

This is 29 October to 4 November 2026, and it is mostly Apache: ZooKeeper, Wicket, ActiveMQ and Syncope published more than thirty CVEs between them. MariaDB Connector/J starts three days before the window, and its dated where it comes up.

The theme is revocation. In ZooKeeper a request authorized just before an ACL change could commit just after it, in ActiveMQ a connection kept its write access through a live ACL update, and in Wicket a WebSocket connection outlived the session it was opened with. The deep dives are the ZooKeeper SASL bug, a code one, and the Wicket batch, a design story about one enforcement point and the routes around it.

## The week in five

- **Apache ZooKeeper, five CVEs disclosed 3 November, fixed in 3.8.8 and 3.9.7.** In `CVE-2026-100252` (Important, 8.6) a SASL client whose principal falls outside the `auth_to_local` rules could take any identity it named, the superuser included (first deep dive). `CVE-2026-100235` (Moderate) is a `setACL` race: a request prepared during the change was checked against the old ACL and applied after the new one, so a client could keep writing, or grant itself permissions again, through its own revocation. An unauthenticated client on a default deployment can leave the server unable to restart, with malformed data under the world-writable `/zookeeper/quota` (`CVE-2026-100300`) or a znode chain deep enough to overflow the recursive snapshot serializer (`CVE-2026-115448`), and a small `multiRead` can build a response far bigger then the request (`CVE-2026-100302`). The snapshot advisory's header lists 3.8.8 and 3.9.7 as affected while its text names them as the fix; both release tags contain it.

- **Apache Wicket, 19 CVEs disclosed 1 November, fixed in 8.20.0, 9.25.0 and 10.12.0.** Most fall into two families: authorization that did not run on every route to a protected object (second deep dive), and escaping for the wrong context. `JavaScriptUtils#escapeQuotes()` did not escape the backslash, so `\'` became `\\'` and the quote closed the string literal (`CVE-2026-83559`). `ExternalLink` on a non-anchor tag put its URL into an `onclick` string with HTML escaping only, which the browser decodes before it runs the handler (`CVE-2026-84231`, 8.x only). And `AuthenticatedWebSession#signIn()` renewed the session id only on the first sign-in, a session fixation on re-authentication (`CVE-2026-82840`). 8.20.0 and 9.25.0 are the last releases of their lines.

- **Apache ActiveMQ, five broker CVEs disclosed 3 November, fixed in 5.19.12, 6.3.3 and 6.4.0.** The broker took connection ids from the client without always checking they belong to the sending connection (`CVE-2026-108946`, Important): with a victim's connection id in its own `ConsumerId`, a client could read the victim's temporary queue, drop its subscription or acknowledge away a message in flight. A connection kept writing after the runtime configuration plugin removed its grant, because the security context caches writable destinations (`CVE-2026-111822`, Important). And a Java detail: KahaDB indexes durable subscriptions by `clientId + ":" + subscriptionName` and writes keys with `DataOutput.writeUTF`, whose length prefix is 16 bits, so two strings each under the limit make a key the store cannot write, and per the advisory the broker shuts down and does not recover (`CVE-2026-111821`). The other two are LDAP DN injection and a heap leak.

- **Apache Syncope, five CVEs disclosed 2 November, fixed in 4.0.9 and 4.1.4.** Administrator-supplied Groovy runs behind a sandbox whose interceptor lived in a `ThreadLocal`, so code that started a new `Thread` ran unchecked (`CVE-2026-113687`, Important); the test that ships with the fix calls `"whoami".execute()` from inside one. The fix moves to `groovy-security-sandbox` 1.1.0, which registers the interceptor in any `Runnable` handed to `new Thread()` or an `Executor`. In `CVE-2026-109083` (Moderate) Flowable evaluated JUEL from a BPMN definition an administrator imported, with no sandbox, and the fix rejects definitions with expression attributes. The rest are two privilege escalations and stacked SQL through the audit search, a follow-up to `CVE-2026-77051`. The 3.0 line, affected through 3.0.16, has no fixed release.

- **MariaDB Connector/J, seven advisories published on GitHub on 26 October (CVE records 29 October), fixed in 3.3.7, 3.4.5, 3.5.11 and 4.0.0.** All assume a malicious or impersonated server. In `CVE-2026-119210` (Medium) the driver answers a server's LOCAL INFILE request by sending the file, after checking that its own query was a `LOAD DATA LOCAL INFILE` of that file. The check was a regular expression whose prefix accepts any character, so it asked whether the text appeared anywhere in the SQL. And the driver reuses names the server returns: if the server names its database `LOAD DATA LOCAL INFILE '/etc/passwd'`, `DatabaseMetaData.getImportedKeys()` sends that name back, quoted, in a `SHOW CREATE TABLE`, the server asks for `/etc/passwd`, and the check passes. `allowLocalInfile` defaults to true. The fix parses the statement instead. Most of the other six are server-declared lengths, such as an 8 MiB field in an 18-byte OK packet.

A few smaller ones. Apache Thrift `CVE-2026-120135` (Important, 31 October): before 0.24.0, `TProtocolUtil.skip()` recursed with a default depth of `Integer.MAX_VALUE`; 0.24.0, out since July, takes the limit from `TConfiguration` (64 by default) and turns `setMaxSkipDepth()` into a no-op. Apache MINA `CVE-2026-118636` (9.8, 31 October): `JavaNativeMessageDecoder` in the 3.0 milestones deserializes with no class filter and will not be fixed, so use 2.2.x. Apache Juneau 9.2.1 (3 November) fixes nine issues. On 30 October the reporter of the PLC4X OPC UA CVEs from 30 September explained them: `verify()` threw on a valid signature (a missing negation) and was handed the pre-decryption length, so together a forged signature passed, and fixing the negation alone would reject legitimate responses. Only 1.0.0 is fixed, and Camel moves to it in 4.24.0 ([CAMEL-25617](https://issues.apache.org/jira/browse/CAMEL-25617){:target="_blank"}). And on 30 October Oracle added products that embed Java, Oracle Database Server among them, to the eMudhra alert from [last week's post](/2026/10/29/java-vulnerabilities-of-the-week-22-28-october-2026.html){:target="_blank"}.

## ZooKeeper: the identity a client asks for is not the one it proved

SASL keeps two identities apart on purpose: the authentication ID the client proved, a Kerberos principal with GSSAPI, and the authorization ID (authzid) it wants to act as, which exists so a gateway can act for a user. A Java server decides in its `CallbackHandler`, which gets an `AuthorizeCallback` holding both strings and answers with `setAuthorized(boolean)`, optionally naming a canonical identity with `setAuthorizedID()`. In the JDK's GSSAPI server the authzid is whatever the client put in its last token, and the callback resolves the result like this:

```java
public String getAuthorizedID() {
    if (!authorized) {
        return null;
    }
    return (authorizedID == null) ? authorizationID : authorizedID;
}
```

So if the handler says yes and names no identity, the session gets the identity the client asked for. ZooKeeper adds it as `new Id("sasl", authorizationID)`, plus `super` when it matches `zookeeper.superUser`.

The client port handler called `ac.setAuthorized(true)` first, then canonicalized the authentication ID with `KerberosName.getShortName()` and passed the result to `setAuthorizedID()`. For a principal in the default realm that overwrites the requested authzid, and the session gets the proved identity. For any other realm, under the default `auth_to_local` rule, `getShortName()` throws `NoMatchingRule`; the handler logged the `IOException` and returned, still authorized and with no authorized ID. A client authenticated as `bob@CROSSREALM` (the case in the fix's own test) that asks for the superuser name gets the whole tree.

The fix, commit `0afc927`, compares the two IDs first and denies on a mismatch, the same `authenticationID.equals(authorizationID)` check the quorum port's `SaslQuorumServerCallbackHandler` makes.

### Why I find it interesting

It is improper authorization, CWE-285, in the form where the identity used for access control is not the one that was authenticated. Each line reads correctly on its own, authorize, canonicalize, log the failure; the bug is the order plus a JDK default, and in the common same-realm setup it never shows.

Two habits follow. Write a security decision last, after everything that can throw, with deny as the default. And treat a SASL principal as two strings: if you maintain a SASL server in Java, grep for `AuthorizeCallback` and check what happens when they differ. Without proxy users the rule is equality; with them it is an explicit list of who may act as whom.

## Wicket: an enforcement point covers only the routes that pass through it

Wicket's `IAuthorizationStrategy` holds an application's access rules and answers three questions: may this class be instantiated, may this component `RENDER` or `ENABLE`, and may this resource be served. The rules live in one place and the framework asks at the right moment. Most of this batch is about the moments.

The resource question is asked in one place, the constructor of `ResourceRequestHandler`, built for requests to a mounted or shared `ResourceReference`. But `ResourceLink`, `Image`, `ImageButton` and `AjaxDownloadBehavior` can also be handed an `IResource`, and then they served it themselves:

```java
Attributes a = new Attributes(RequestCycle.get().getRequest(), RequestCycle.get()
    .getResponse(), null);
resource.respond(a);
```

So the constructor decided whether the check ran: `new ResourceLink(id, reference)` was authorized and `new ResourceLink(id, resource)` was not. `ExportToolbar` uses the second form (`CVE-2026-82705`). The WebSocket transport called `onPayload()` on a shared `WebSocketResource` directly, so `@AuthorizeResource` had no effect there (`CVE-2026-82115`).

`RENDER` had its own way around. The outcome of the `RENDER` check is stored in a visibility flag that only `Component#configure()` writes, during rendering. A bookmarkable listener URL with an empty page id makes Wicket build a fresh page and invoke the listener before any render pass, so the flag still says visible and a denied component's listener runs (`CVE-2026-82704`). For stateless pages Wicket writes such URLs itself.

The fixes make each route ask the strategy. `ResourceLink.onRequest()` now responds through `new ResourceRequestHandler(resource, resourceParameters)`, which in the commit's words "keeps one enforcement point rather than adding a second one that can drift"; the WebSocket transport asks about the resource a client connects to; and a fresh page gets its `RENDER` check before the listener runs.

### Why I find it interesting

This is missing authorization, CWE-862, measured against complete mediation, every access to every object checked, as Saltzer and Schroeder put it in 1975. A single enforcement point is the right way to get there, and the fix kept it rather then adding checks to each component. The review question is which routes reach the object without passing through the check. Thats a question about entry points, here a second constructor, a second transport and a request that skipped a lifecycle phase, and it fits any Java stack: a filter on `/admin/*` does not see the same handler reached through a WebSocket, a message listener or a scheduled job.

The `RENDER` case adds that a decision stored as state is only as current as the code that last wrote it, which is the revocation theme one level down. Fifteen of the nineteen advisories credit Emond Papegaaij, who sent the announcement for the PMC.

## What I take away from this week

First, revocation. ZooKeeper's prepared request, ActiveMQ's per-connection write cache and Wicket's session copy from the handshake each held a permission longer than the policy did. When code keeps an authorization decision on a long-lived object, find the event that should invalidate it and test it: revoke, then try the old connection. Its the cheapest test I know for all three.

Second, identity taken from a field the client fills in: the ZooKeeper authzid, the connection id in an ActiveMQ `ConsumerId`, the upload limits Wicket read from request parameters (`CVE-2026-84170`). In each case the server already knew the answer from its own state, and the fix uses that.

Third, where the fixes land. ZooKeeper, ActiveMQ, Wicket, Syncope and Juneau are direct upgrades, and all five ActiveMQ items are in the broker. Camel main has moved to ZooKeeper 3.9.7, PLC4X 1.0.0 and MariaDB Connector/J 4.0.0 for 4.24.0. MariaDB is the one to act on now: Spring Boot 4.0.9 and 4.1.2 and Quarkus 3.40.1 manage 3.5.10, so override it to 3.5.11 (`mariadb.version` in Spring Boot) until the next patch releases. If you don't use `LOAD DATA LOCAL INFILE`, `allowLocalInfile=false` also works, since 3.5.9 enforces it for server-initiated requests (`CVE-2026-61700`, July). Thrift users on 0.24.0 or later, Camel included, already have that fix. Every fixed version named here is on Maven Central; I checked.

## References

- [oss-security: Apache ZooKeeper SASL authorization bypass CVE-2026-100252](https://www.openwall.com/lists/oss-security/2026/11/03/10){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ZooKeeper setACL race CVE-2026-100235](https://www.openwall.com/lists/oss-security/2026/11/03/9){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ZooKeeper quota denial of service CVE-2026-100300](https://www.openwall.com/lists/oss-security/2026/11/03/11){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ZooKeeper multiRead allocation CVE-2026-100302](https://www.openwall.com/lists/oss-security/2026/11/03/12){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ZooKeeper snapshot recursion CVE-2026-115448](https://www.openwall.com/lists/oss-security/2026/11/03/13){:target="_blank"} - Openwall oss-security
- [ZooKeeper fix: SASL authzid validation on the client port](https://github.com/apache/zookeeper/commit/0afc92745ce5249e8aff511bab6d2208a72e45fb){:target="_blank"} - apache/zookeeper on GitHub
- [AuthorizeCallback.java](https://github.com/openjdk/jdk/blob/master/src/java.security.sasl/share/classes/javax/security/sasl/AuthorizeCallback.java){:target="_blank"} - openjdk/jdk on GitHub
- [GssKrb5Server.java](https://github.com/openjdk/jdk/blob/master/src/jdk.security.jgss/share/classes/com/sun/security/sasl/gsskerb/GssKrb5Server.java){:target="_blank"} - openjdk/jdk on GitHub
- [oss-security: Apache Wicket: 19 CVEs fixed in 8.20.0, 9.25.0 and 10.12.0](https://www.openwall.com/lists/oss-security/2026/11/01/4){:target="_blank"} - Openwall oss-security
- [Apache Wicket CVE-2026-82705: authorization strategy not applied to an IResource embedded in a component](https://lists.apache.org/thread/t9y3sho3wkqkr1vsf74xn8tzmgdr1tgo){:target="_blank"} - Apache mailing lists
- [Apache Wicket CVE-2026-82115: authorization strategy not applied to a shared WebSocketResource](https://lists.apache.org/thread/0lj9og7vp22ps77nrwdj42f64cfb9g4c){:target="_blank"} - Apache mailing lists
- [Apache Wicket CVE-2026-82704: listener invoked on a page instance never configured](https://lists.apache.org/thread/yrtov7vfmj7c7wl7j1tt0dsv2t0zwy9b){:target="_blank"} - Apache mailing lists
- [Apache Wicket CVE-2026-82559: WebSocket messages processed against an HTTP session snapshot](https://lists.apache.org/thread/5vkw9wz6gf135ccosj928zjs4hqy7bcv){:target="_blank"} - Apache mailing lists
- [Apache Wicket CVE-2026-83559: escapeQuotes does not escape the backslash](https://lists.apache.org/thread/77kc85ztkxdpp7wg7cv61c4539j9s1g3){:target="_blank"} - Apache mailing lists
- [Apache Wicket CVE-2026-84231: XSS in ExternalLink on a non-anchor tag](https://lists.apache.org/thread/39kz9838w25797bpkxn1hvfsm8jtc4zv){:target="_blank"} - Apache mailing lists
- [Wicket fix: ask IAuthorizationStrategy about a resource served by a component](https://github.com/apache/wicket/commit/bc80f0b8d6bffc57f1ec5d098f15dd4956615bd4){:target="_blank"} - apache/wicket on GitHub
- [Wicket fix: re-evaluate the RENDER authorization on a freshly created page](https://github.com/apache/wicket/commit/3fb12ceef0179b7fa9b1b77ffb2ea70cb80b96b8){:target="_blank"} - apache/wicket on GitHub
- [Wicket fix: ask IAuthorizationStrategy about the WebSocketResource a client connects to](https://github.com/apache/wicket/commit/30dfd703b0dac2225a382cfc394889f859d73a67){:target="_blank"} - apache/wicket on GitHub
- [oss-security: Apache ActiveMQ connection id validation CVE-2026-108946](https://www.openwall.com/lists/oss-security/2026/11/03/17){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ActiveMQ write access after live ACL revocation CVE-2026-111822](https://www.openwall.com/lists/oss-security/2026/11/03/20){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ActiveMQ KahaDB client identifier length CVE-2026-111821](https://www.openwall.com/lists/oss-security/2026/11/03/19){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ActiveMQ LDAPAuthorizationMap DN injection CVE-2026-108500](https://www.openwall.com/lists/oss-security/2026/11/03/16){:target="_blank"} - Openwall oss-security
- [oss-security: Apache ActiveMQ rejected temporary destination subscriptions CVE-2026-108949](https://www.openwall.com/lists/oss-security/2026/11/03/18){:target="_blank"} - Openwall oss-security
- [ActiveMQ fix: validate subscription key length in KahaDB](https://github.com/apache/activemq/commit/06d4e17786540a5a7df4bd8d8e0b4e0ec04ecaa3){:target="_blank"} - apache/activemq on GitHub
- [oss-security: Apache Syncope Groovy sandbox bypass CVE-2026-113687](https://www.openwall.com/lists/oss-security/2026/11/02/10){:target="_blank"} - Openwall oss-security
- [oss-security: Apache Syncope Flowable JUEL expression CVE-2026-109083](https://www.openwall.com/lists/oss-security/2026/11/02/7){:target="_blank"} - Openwall oss-security
- [oss-security: Apache Syncope audit events SQL injection CVE-2026-114364](https://www.openwall.com/lists/oss-security/2026/11/02/11){:target="_blank"} - Openwall oss-security
- [Syncope fix: strengthen Groovy sandbox](https://github.com/apache/syncope/commit/8597bfa5813597d26b54cd7a2f3319ccc5fc52cb){:target="_blank"} - apache/syncope on GitHub
- [groovy-security-sandbox issue 2: propagate SandboxInterceptor to child threads](https://github.com/Tirasa/groovy-security-sandbox/issues/2){:target="_blank"} - Tirasa on GitHub
- [MariaDB Connector/J LOAD DATA LOCAL INFILE advisory GHSA-fhx5-2xqw-2cgm](https://github.com/mariadb-corporation/mariadb-connector-j/security/advisories/GHSA-fhx5-2xqw-2cgm){:target="_blank"} - mariadb-corporation/mariadb-connector-j on GitHub
- [MariaDB Connector/J OK packet allocation advisory GHSA-h599-m3w5-9m7x](https://github.com/mariadb-corporation/mariadb-connector-j/security/advisories/GHSA-h599-m3w5-9m7x){:target="_blank"} - mariadb-corporation/mariadb-connector-j on GitHub
- [oss-security: Apache Thrift TProtocolUtil.skip() recursion depth CVE-2026-120135](https://www.openwall.com/lists/oss-security/2026/10/31/2){:target="_blank"} - Openwall oss-security
- [oss-security: Apache MINA JavaNativeMessageDecoder CVE-2026-118636](https://www.openwall.com/lists/oss-security/2026/10/31/3){:target="_blank"} - Openwall oss-security
- [oss-security: Apache Juneau 9.2.1 fixes multiple vulnerabilities](https://www.openwall.com/lists/oss-security/2026/11/03/7){:target="_blank"} - Openwall oss-security
- [oss-security: Apache PLC4X OPC UA mechanisms for CVE-2026-102508 and CVE-2026-102509](https://www.openwall.com/lists/oss-security/2026/10/30/18){:target="_blank"} - Openwall oss-security
- [CAMEL-25617: camel-plc4x, upgrade to PLC4X 1.0](https://issues.apache.org/jira/browse/CAMEL-25617){:target="_blank"} - Apache JIRA
- [Oracle Security Alert for CVE-2026-114820](https://www.oracle.com/security-alerts/alert-cve-2026-114820.html){:target="_blank"} - Oracle
- [Java vulnerabilities of the week: 22 to 28 October 2026](/2026/10/29/java-vulnerabilities-of-the-week-22-28-october-2026.html){:target="_blank"} - oscerd.github.io
- [The CVE stigma, we need a new security culture](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html){:target="_blank"} - oscerd.github.io
