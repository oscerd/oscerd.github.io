---
layout: page
title: About
---

<div class="about-header">
  <div class="about-avatar" aria-hidden="true">AC</div>
  <div>
    <h2>Andrea Cosentino</h2>
    <p class="about-title">Senior Principal Software Engineer at IBM</p>
  </div>
</div>

I'm an open source enthusiast and **Senior Principal Software Engineer** at [IBM](https://www.ibm.com){:target="_blank"}, working on integration technologies. I co-lead the [Apache Camel](https://camel.apache.org){:target="_blank"} project together with Claus Ibsen, focusing on enterprise integration.

Most of what I do sits in the open: components, releases, security fixes, the boring maintenance work that keeps a project the size of Camel usable. This blog is where I write the parts that don't fit in a commit message.

## Open Source Leadership

<div class="about-roles">
  <div class="role-card">
    <span class="role-icon">🐪</span>
    <div class="role-details">
      <strong>Apache Camel PMC Chair</strong>
      <span>Leading the Apache Camel project</span>
    </div>
  </div>
  <div class="role-card">
    <span class="role-icon">🪶</span>
    <div class="role-details">
      <strong>Apache Member</strong>
      <span>Member of the Apache Software Foundation</span>
    </div>
  </div>
  <div class="role-card">
    <span class="role-icon">📦</span>
    <div class="role-details">
      <strong>Apache Karaf Committer</strong>
      <span>Contributing to the OSGi runtime</span>
    </div>
  </div>
  <div class="role-card">
    <span class="role-icon">🔧</span>
    <div class="role-details">
      <strong>Apache ServiceMix PMC</strong>
      <span>Project Management Committee member</span>
    </div>
  </div>
</div>

## How I got here

I didn't plan any of this as a career path. I picked up a component that needed fixing, then another one, and the rest followed. Looking back at what I was writing about, the arc is reasonably clear.

<ul class="timeline">
  <li class="is-current">
    <span class="timeline-when">Since July 2025</span>
    <span class="timeline-what">Senior Principal Software Engineer, IBM</span>
    <span class="timeline-detail">Integration technologies. On the Apache side I chair the Camel PMC and co-lead the project with Claus Ibsen, which in practice means releases, reviews, security triage and a lot of mailing list.</span>
  </li>
  <li>
    <span class="timeline-when">2026</span>
    <span class="timeline-what">Security became the main thread</span>
    <span class="timeline-detail">
      Running a weekly <a href="/2026/08/05/java-vulnerabilities-of-the-week-29-july-4-august-2026.html">Java vulnerabilities digest</a>,
      getting Camel ready for <a href="/2026/05/06/apache-camel-is-preparing-for-post-quantum-cryptography.html">post-quantum cryptography</a>,
      and arguing that we need <a href="/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html">a better culture around CVEs</a>
      instead of treating every advisory as an accusation.
    </span>
  </li>
  <li>
    <span class="timeline-when">2020 &ndash; 2025</span>
    <span class="timeline-what">Camel 3, the cloud and secrets management</span>
    <span class="timeline-detail">
      The <a href="/2020/03/06/aws2-components-are-here.html">AWS2 component family</a>,
      <a href="/2020/12/24/camel-kafka-connector-0.7.0.html">Camel Kafka Connector</a>, and the
      <a href="/2022/03/24/camel-vault-cloud-provider.html">Vault and Secrets property function</a>
      with <a href="/2022/12/20/camel-vault-automatic-context-reload.html">automatic context reload</a>,
      so a running route can pick up a rotated secret without a restart.
      More recently <a href="/2025/10/15/intelligent-document-processing-camel-docling-meet-langchain4j.html">document processing with LangChain4j</a>
      and <a href="/2025/12/16/deploying-camel-on-aws-ecs-fargate.html">Camel on ECS Fargate</a>.
    </span>
  </li>
  <li>
    <span class="timeline-when">2016 &ndash; 2018</span>
    <span class="timeline-what">Components, containers and the OSGi years</span>
    <span class="timeline-detail">
      Writing and maintaining components: <a href="/2016/07/11/camel-kubernetes-introduction.html">camel-kubernetes</a>,
      <a href="/2016/04/18/introducing-camel-nats.html">camel-nats</a>,
      <a href="/2016/10/14/camel-cassandraql-on-kubernetes-cassandra-cluster.html">camel-cassandraql on Kubernetes</a>,
      the Camel 2.x releases, and <a href="/2016/07/06/contributing-camel-components.html">helping other people land their own components</a>.
      This is also the ServiceMix and Karaf period, which is where I learned how much of integration is really classloading.
    </span>
  </li>
  <li>
    <span class="timeline-when">2015 &ndash; 2025</span>
    <span class="timeline-what">Red Hat</span>
    <span class="timeline-detail">
      Ten years, and almost everything above happened there: the middleware and integration work, the components,
      the releases, and most of the patents further down this page. A decade in one place sounds static,
      but the stack underneath moved from OSGi containers to Spring Boot and Quarkus and then to whatever we are
      calling cloud native this year, so in practice very little of it stayed still.
    </span>
  </li>
</ul>

## Current Focus

I'm currently working on expanding the Camel ecosystem through subprojects like:

- **Camel Spring Boot** - Spring Boot integration for Apache Camel
- **Camel Quarkus** - Supersonic, subatomic Java integrations
- **Kamelets** - Reusable route snippets for easy integrations

Outside of Camel itself I contribute to Hawtio and to a number of smaller projects around the same ecosystem, usually because something broke and I happened to be the person looking at it.

A lot of the work is not feature work at all. Keeping a project the size of Camel healthy means release management, reviewing other people's patches, triaging security reports, chasing dependency updates that nobody enjoys, and making sure the same route behaves the same way on Spring Boot, on Quarkus and on plain Java. None of that shows up in a changelog in a way anyone notices, but its the part that decides whether the project is still usable in five years.

The rest of my time goes on the boundary between the project and the people using it: answering questions on the mailing lists, writing up what changed and why, and trying to make the upgrade path from one major version to the next less painful then it needs to be.

## Patents

Alongside the open source work I have spent a lot of time on patents: 41 inventions so far, 16 of them granted. Most were filed during the Red Hat years and a few more since moving to IBM. The full record is on my [Google Scholar profile](https://scholar.google.com/citations?user=4QEpZp4AAAAJ){:target="_blank"}, which also still carries a paper from a previous life, "FAN: Fast NetFlow analyser", from IEEE INFOCOM 2013.

They fall into a few groups. A lot of messaging and event streaming: serializers, schemas, the claim check pattern, message sizing on mesh networks, compression for idempotent stores. Then secrets and key management, which is the same problem I ended up solving in the open with the Camel Vault property function. Then a long run of connected vehicle work: over the air updates, keyless entry, vehicle control on serverless functions. Lately it has been supply chain and AI: SBOM compliance, working out the minimal dependency update that clears a CVE, data provenance for foundational model training, and mediating access to quantum services.

The part that takes the time is not the writing. It is prior art search: reading through everything already published in an area before you can argue that the thing in front of you is actually new. I have done a lot of that, on my own submissions and reviewing other people's, and it turns out to be a habit worth having well outside the patent process. It is the same instinct that tells you a brand new CVE is really the same bug shape you already saw two years ago.

<h3>Granted (16)</h3>

<ul class="patent-list">
  <li class="patent-item">
    <span class="patent-title">Message format indicator for resource-constrained devices</span>
    <span class="patent-meta">US 12,556,620 &middot; granted 2026</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Idling and waking a sender node for event message delivery</span>
    <span class="patent-meta">US 12,554,535 &middot; granted 2026</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Cloud to on-premises storage migration</span>
    <span class="patent-meta">US 12,481,447 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Vehicle control using serverless functions</span>
    <span class="patent-meta">US 12,472,961 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Adaptive asymmetric-key compression for idempotent data stores</span>
    <span class="patent-meta">US 12,457,095 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Intra-vehicle over-the-air updates</span>
    <span class="patent-meta">US 12,436,756 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Adaptive compression for idempotent data stores in computer messaging</span>
    <span class="patent-meta">US 12,425,494 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Managing message sizes in a mesh network</span>
    <span class="patent-meta">US 12,363,585 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Cross-domain data access</span>
    <span class="patent-meta">US 12,284,217 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Sizing service for cloud migration away from only cloud storage</span>
    <span class="patent-meta">US 12,260,224 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Application profiling to resize and reconfigure compute instances</span>
    <span class="patent-meta">US 12,248,386 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Object creation from schema for event streaming platform</span>
    <span class="patent-meta">US 12,197,401 &middot; granted 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Creation of message serializer for event streaming platform</span>
    <span class="patent-meta">US 11,995,096 &middot; granted 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Secrets rotation for vehicles</span>
    <span class="patent-meta">US 11,991,278 &middot; granted 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Cloud-based keyless entry system</span>
    <span class="patent-meta">US 11,731,585 &middot; granted 2023 &middot; continuation US 12,214,747</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Dynamic runtime application integration</span>
    <span class="patent-meta">US 11,641,306 &middot; granted 2023</span>
  </li>
</ul>

<details class="patent-more">
  <summary>Filed and pending (25)</summary>
<ul class="patent-list">
  <li class="patent-item">
    <span class="patent-title">Identifying minimal dependency updates to mitigate known CVEs</span>
    <span class="patent-meta">Application 18/890,363 &middot; 2026</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Auditable data provenance for training dataset prediction in large foundational models</span>
    <span class="patent-meta">Application 18/890,281 &middot; 2026</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Deployment of configuration files generated from serverless functions</span>
    <span class="patent-meta">Application 18/888,776 &middot; 2026</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Dynamic mediation of access to quantum services</span>
    <span class="patent-meta">Application 18/817,964 &middot; 2026</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Mitigating vulnerabilities in a software program based on its usage</span>
    <span class="patent-meta">Application 18/790,484 &middot; 2026</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Event sourcing for quantum debugging and failover mechanisms</span>
    <span class="patent-meta">Application 18/738,906 &middot; 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Techniques for validating safety agreement compliance using SBOMs</span>
    <span class="patent-meta">Application 18/639,683 &middot; 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Migrating ransomware activity of an operating system</span>
    <span class="patent-meta">Application 18/637,578 &middot; 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Mitigating ransomware activity of a host system using a kernel monitor</span>
    <span class="patent-meta">Application 18/531,073 &middot; 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Claim check mechanism for a message payload</span>
    <span class="patent-meta">Application 18/470,928 &middot; 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Matching commands to attack patterns</span>
    <span class="patent-meta">Application 18/362,654 &middot; 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Sizing service for cloud migration to physical machine</span>
    <span class="patent-meta">Application 19/073,894 &middot; 2025</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Intra-vehicle over-the-air updates based on install timestamp</span>
    <span class="patent-meta">Application 18/326,927 &middot; 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Intra-vehicle over-the-air updates incorporating update compatibility</span>
    <span class="patent-meta">Application 18/326,797 &middot; 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Secrets management topology migrator</span>
    <span class="patent-meta">Application 18/198,402 &middot; 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Key rotation based on traffic state</span>
    <span class="patent-meta">Application 18/160,504 &middot; 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Migrating secrets from a cloud environment to a local system</span>
    <span class="patent-meta">Application 18/116,995 &middot; 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">User-customized vehicle control using serverless functions</span>
    <span class="patent-meta">Application 17/899,098 &middot; 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Proactive integrity checks</span>
    <span class="patent-meta">Application 17/874,138 &middot; 2024</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Adjusting the size of a resource pool</span>
    <span class="patent-meta">Application 17/680,725 &middot; 2023</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Authenticating electronic key devices</span>
    <span class="patent-meta">Application 17/672,609 &middot; 2023</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Providing system updates in automotive contexts</span>
    <span class="patent-meta">Application 17/343,253 &middot; 2022</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Multi-strategy compression scheme</span>
    <span class="patent-meta">Application 17/231,766 &middot; 2022</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Simplifying creation and publishing of schemas</span>
    <span class="patent-meta">Application 17/231,773 &middot; 2022</span>
  </li>
  <li class="patent-item">
    <span class="patent-title">Managing event delivery in a serverless computing environment</span>
    <span class="patent-meta">Application 17/022,528 &middot; 2022</span>
  </li>
</ul>
</details>

## Interests

Beyond integration, I'm passionate about:

- Security and cryptography
- Cloud-native technologies
- Developer experience and tooling

## This Blog

This is my personal space where I write about open source, Apache Camel, programming, and technology in general. Lately it leans heavily towards security: what actually broke, and why, rather then the score somebody assigned to it.

## Elsewhere

Best place to reach me is GitHub or LinkedIn. I read everything, I don't always reply quickly.

<div class="about-social">
  <a class="about-social-link" href="https://github.com/oscerd" target="_blank" rel="noopener noreferrer">
    <svg class="about-social-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" aria-hidden="true" focusable="false">
      <path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"/>
    </svg>
    <span>GitHub<span class="about-social-handle">@oscerd</span></span>
  </a>
  <a class="about-social-link" href="https://www.linkedin.com/in/andrea-cosentino-439119100/" target="_blank" rel="noopener noreferrer">
    <svg class="about-social-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" aria-hidden="true" focusable="false">
      <path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433c-1.144 0-2.063-.926-2.063-2.065 0-1.138.92-2.063 2.063-2.063 1.14 0 2.064.925 2.064 2.063 0 1.139-.925 2.065-2.064 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/>
    </svg>
    <span>LinkedIn<span class="about-social-handle">Andrea Cosentino</span></span>
  </a>
  <a class="about-social-link" href="https://twitter.com/oscerd2" target="_blank" rel="noopener noreferrer">
    <svg class="about-social-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" aria-hidden="true" focusable="false">
      <path d="M18.244 2.25h3.308l-7.227 8.26 8.502 11.24H16.17l-5.214-6.817L4.99 21.75H1.68l7.73-8.835L1.254 2.25H8.08l4.713 6.231zm-1.161 17.52h1.833L7.084 4.126H5.117z"/>
    </svg>
    <span>Twitter / X<span class="about-social-handle">@oscerd2</span></span>
  </a>
  <a class="about-social-link" href="/atom.xml">
    <svg class="about-social-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg" aria-hidden="true" focusable="false">
      <path d="M4.257 3.006C13.4 3.006 20.994 10.6 20.994 19.743h-3.32c0-7.4-6.017-13.417-13.417-13.417zm0 6.69c5.45 0 10.047 4.597 10.047 10.047h-3.32c0-3.72-3.007-6.727-6.727-6.727zM6.54 16.2a2.28 2.28 0 110 4.56 2.28 2.28 0 010-4.56z"/>
    </svg>
    <span>RSS<span class="about-social-handle">Subscribe to the feed</span></span>
  </a>
</div>

<!--
  Want an email address here too? Drop in another .about-social-link, for example:

  <a class="about-social-link" href="mailto:you@apache.org">
    <span>Email<span class="about-social-handle">you@apache.org</span></span>
  </a>
-->
