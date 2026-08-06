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
    <span class="timeline-when">Now</span>
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
      plus <a href="/2018/06/28/building-syndesis-with-camel-snapshot.html">Syndesis</a> and the Camel 2.x releases.
      This is also the ServiceMix and Karaf period, which is where I learned how much of integration is really classloading.
    </span>
  </li>
</ul>

## Current Focus

I'm currently working on expanding the Camel ecosystem through subprojects like:

- **Camel Spring Boot** - Spring Boot integration for Apache Camel
- **Camel Quarkus** - Supersonic, subatomic Java integrations
- **Kamelets** - Reusable route snippets for easy integrations

I'm also actively contributing to Syndesis, Hawtio, Fabric8, and various other open source projects.

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
