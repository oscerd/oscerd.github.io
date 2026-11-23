---
layout: post
title: "AI security, week of 16 to 22 November 2026"
---

This is the seven days to 22 November 2026, and the boundary that broke most often was the one around the local machine: several AI tools that run on `127.0.0.1` read "bound to loopback" as "only local callers", and a web page the operator visited could reach them through the browser. The anatomy walks that chain, DNS rebinding against a local MCP server for Todoist, Obsidian or Jupyter, where the page lists and rewrites your data with the server's own credential and no prompt. The other half of the week was supply chain: a credential stealer shipped in two PyPI releases of Microsoft's `agent-framework`, and researchers seeding canary prompts found the inference providers behind an LLM gateway reading and copying what passed through. The window does not overlap [last week's post](/2026/11/16/ai-security-week-of-9-15-november-2026.html){:target="_blank"}, though two threads continue: NLTK is back with 23 new CVE records (all in the same `3.10.4` I noted last time for a different one), and ARTEX turns up again inside one of those provider breaches.

## At a glance

- **Patch now:** Arize Phoenix 15.7.0 (`CVE-2026-53998`, Critical, 10.0), NLTK 3.10.4 (23 CVEs including the Critical `CVE-2026-128138`), MCP Python SDK 1.30.0 or 2.2.0 (`CVE-2026-115563`), and the Todoist 10.3.3 and Jupyter 1.0.3 MCP servers (`CVE-2026-76807` and `CVE-2026-77359`, both Critical).
- **Read closely:** the DNS rebinding anatomy, where binding a tool to localhost is mistaken for authenticating its callers.
- **Watch:** MONAI Label (`GHSA-qc22-rmgc-vg94`, critical path traversal, no fixed release yet, already exploited by Cairn agents per Amazon), and `axon-mcp` (`CVE-2026-76818`, code pulled with no integrity check, no fix published).

## By layer

### Inference and serving

- **LLMKube 0.9.0** `CVE-2026-68903` (High, 7.7): `Model.spec.source` took an absolute path or `file://`, which the controller both opened and mounted with `hostPath`, so a namespace user reads its service-account token or node files. Caching is on by default. [Advisory](https://github.com/advisories/GHSA-jw3m-8q7m-f35r){:target="_blank"}

### Agent frameworks and MCP

- **MCP Python SDK 1.30.0 or 2.2.0** `CVE-2026-115563` (High, 7.5): the OAuth client let a malicious MCP server name the authorization server, so it receives the `client_secret` and PKCE verifier meant for the real one. [Advisory](https://github.com/modelcontextprotocol/python-sdk/security/advisories/GHSA-qx49-fqc8-xw99){:target="_blank"}
- **k8s-mcp-server 1.1.0** `CVE-2026-130007` (High): the default SSE transport binds `0.0.0.0` and runs `tools/call` with no auth and no `Origin` check, so anyone who reaches the port invokes its Kubernetes tools, the write ones included. [Advisory](https://github.com/advisories/GHSA-m932-63jc-4jqq){:target="_blank"}

### AI applications and UIs

- **Arize Phoenix 15.7.0** `CVE-2026-53998` (Critical, 10.0): the `applyChatTemplate` GraphQL query rendered an attacker string as a Python f-string, so an unauthenticated caller walks object internals to read `os.environ`, including `PHOENIX_SECRET`, the key that signs every JWT. [Advisory](https://github.com/Arize-ai/phoenix/security/advisories/GHSA-gfv3-m769-5xm3){:target="_blank"}
- **CodeWhale 0.9.0** `CVE-2026-83816` (High, 8.1): the "parallel read-only" classifier auto-approved `fd`, `rg` and `git grep`, whose own `-x`, `--pre` and `--open-files-in-pager` flags run any program, so an injected tool call executes commands with no prompt. [Advisory](https://github.com/advisories/GHSA-3mgh-93r5-r7jf){:target="_blank"}

### MLOps and data pipelines

- **NLTK 3.10.4** `CVE-2026-128138` (Critical, 9.8): the sharpest of 23 CVEs disclosed this week, where NLTK's `pathsec` sandbox and native-tool wrappers were escaped through symlinks, hardlinks and path traversal, and `java()`, Graphviz and prover9 could run attacker binaries. [Advisory](https://github.com/advisories/GHSA-x8q7-6c7m-52gw){:target="_blank"}
- **Jupyter MCP Server 1.0.3** `CVE-2026-77359` (Critical): `/mcp` rejected forged `Host` and `Origin` and required a token, but the `PUT /api/connect` and `DELETE /api/stop` management routes beside it did neither, so a web page repoints the server's Jupyter backend. [Advisory](https://github.com/datalayer/jupyter-mcp-server/security/advisories/GHSA-9x8x-crjq-g64h){:target="_blank"}

**Also patched:** sqlite-mcp 0.1.2 (`CVE-2026-130008`, read-only bypass through `;`-chained statements and `ATTACH`), the mssql-mcp servers 0.5.0 and 0.6.0 (`CVE-2026-63129`, SQL injection in the write tools), code-review-mcp-server 1.5.3 (`CVE-2026-68934`, command injection via `folderPath`), safari-mcp 2.22.15 (JavaScript injection through tool arguments), and CodeWhale 0.9.0 again (`CVE-2026-83817`, automation `auto_approve` defaulting to true).

## Case files

### Microsoft Agent Framework: a credential stealer in two PyPI releases (17 to 18 November)

`agent-framework` is Microsoft's Python SDK for building agents. The attacker controlled two PyPI versions, 1.26.1 and 1.26.2, published 17 November at 20:35 and 21:08 UTC.

Each wheel added a `.pth` startup file (`agent_framework_meta.pth`, renamed `af_runtime_diag.pth` in the second) that Python runs at every interpreter start, so the stealer ran on install without anyone importing the package. It forked into the background and harvested environment variables naming a `TOKEN`, `SECRET`, `KEY`, `AZURE`, `AWS` or `GITHUB`, common credential files (`~/.azure`, `.npmrc`, `.git-credentials`), other processes' `/proc/*/environ`, and Azure and GitHub Actions identity tokens, then posted them to a `webhook.site` URL. The rest of the wheel matched the clean 1.26.0.

Both releases were removed from PyPI and 1.26.0 is the current clean release. The GitHub advisory `GHSA-764m-3ppc-5m73` (18 November) and OpenSSF's malicious-packages entry both say to treat any host that installed 1.26.1 or 1.26.2 as exposed and rotate every credential it could reach, CI and Actions secrets included. How the publishing access was obtained has not been explained.

OWASP LLM Top 10 2025: LLM03 Supply Chain. MITRE ATLAS: AML.T0010 AI Supply Chain Compromise, with AML.T0055 Unsecured Credentials for the stolen tokens.

### OpenRouter: prompts, and the secrets inside them, leaked at downstream providers (19 to 22 November)

OpenRouter is a gateway that routes a prompt to whichever inference provider is cheapest or fastest. Nothing on the caller's side is the attack; the trust is that the chosen provider honours its data policy.

Dragos (HD Moore, report dated 19 November) seeded about two million prompts over July to November, each carrying a unique canary URL and credential, and watched who used them. Data from at least three providers was exposed three ways: intrusions none of them had noticed, resellers that simply turned on request logging for a cheap model, and providers that ignored zero retention. In the largest, a couple hundred seeded AWS keys were copied from requests the provider rejected before any model ran, so taken at the front door, then used in tens of thousands of API calls. An unnamed Chinese AI lab fetched the canary URLs from more than half the prompts sent to its models, two thirds of them marked zero retention.

OpenRouter's post of 22 November says it off-boarded three providers, removed 125 endpoints from zero-retention eligibility and notified affected customers, and that the findings showed no breach of OpenRouter itself; it did not name the providers. Dragos's lesson is cheap: seed your own canary credential where your agents pick things up and treat any use as an incident, since a privacy flag only binds the first hop.

OWASP LLM Top 10 2025: LLM02 Sensitive Information Disclosure. MITRE ATLAS: AML.T0024 Exfiltration via AI Inference API, with AML.T0057 LLM Data Leakage.

## Anatomy of the week: a web page that drove a local MCP server

Three MCP servers this week shared one shape: bound to `127.0.0.1`, no per-request authentication, holding the operator's own credential, and dispatching any request that arrives without checking where it came from. Todoist's `todoist-mcp-http`, `obsidian-sync-mcp` in its no-auth mode and the Jupyter server all did it. Here is the Obsidian and Todoist shape, from the advisories.

1. The operator starts a local MCP server bound to `127.0.0.1`, holding the Obsidian vault path or the Todoist API key. This is the documented local setup.
2. In an ordinary browser, the operator opens an attacker page: an ad, a link, a tab left open.
3. The page is served from `attacker.example`, whose DNS the attacker runs, answered with the attacker's own IP on a very short TTL.
4. The page's JavaScript waits, then fetches `attacker.example` again; the DNS now answers `127.0.0.1`. The browser rebinds the name to loopback but still treats the page as same origin.
5. `fetch('http://attacker.example:<port>/mcp', ...)` now lands on the victim's local server, and the browser sends `Host: attacker.example`.
6. The server validates neither `Host` nor `Origin`, so it dispatches the MCP method with the operator's own credential. No token is required, and a browser would not attach one to a cross-site request anyway.
7. The page calls `tools/list`, then `tools/call`: read the whole vault or list the tasks, then `write_note` or `delete_object`. No prompt, no click past visiting the page.

### Where it could have stopped

- Input handling: reject any request whose `Host` is not `localhost` or `127.0.0.1`. That defeats the rebinding, and is the check `obsidian-sync-mcp` 0.5.7 adds, the same one the MCP SDK's transport-security setting applies for localhost servers.
- Output and origin: reject a cross-origin `Origin` that is not on an allowlist. This closes a simpler variant that needs no rebinding, a direct cross-origin `fetch` to `127.0.0.1` whose response a wildcard `Access-Control-Allow-Origin: *` hands back.
- Tool permissions: require a bearer token. A browser does not attach one to a forged cross-site request, so it closes both variants at once, and is the workaround for all three servers. Todoist's one token was its own server-side Todoist key, not a caller credential.
- Coverage: the Jupyter server put the check on `/mcp` but not on the `/api/connect` and `/api/stop` routes beside it, so the guard has to cover every route that changes state, not only `/mcp`.

The pattern to look for is a local service that reads "bound to `127.0.0.1`" as authentication. It is not: the browser sits on the same host and will carry a page's request to loopback for it. To find it, list every HTTP service you run on localhost that holds a credential or a tool, and confirm each validates `Host` and `Origin`, or demands a token, before it acts, on every route.

## From the research side

- [Amazon on Cairn agents finding a zero-day](https://aws.amazon.com/blogs/security/chinese-nexus-actors-cairn-ai-agents-found-a-zero-day-on-their-own/){:target="_blank"} (Amazon Threat Intelligence, 21 November): autonomous agents built on the open-source Cairn pentest framework found a path-traversal zero-day in MONAI Label, the medical-imaging annotation tool, and reached a root shell in about eleven minutes by writing a reverse shell into a file Python runs at startup. It continues the agentic-attacker thread from earlier weeks, and is a separate report from the 16 November one on agents hunting exposed tools. The advisory `GHSA-qc22-rmgc-vg94` has no fixed release yet, so track it and enable authentication, off by default.
- [Signal on blocking screen capture by AI assistants](https://signal.org/blog/protecting-your-privacy/){:target="_blank"} (Signal, 22 November): Signal added settings that stop OS assistants such as Siri's Onscreen Awareness and Gemini from reading chat content as it appears on screen, a "Block Siri AI" toggle on iOS and an automatic Screen Security mode on Android. If you build a sensitive app, assume an on-device assistant can read the screen, and use the platform's screen-security flags.
- [Apple splits Full Disk Access in macOS 27.3](https://www.macrumors.com/2026/11/16/macos-27-3-full-desk-access/){:target="_blank"} (reported 16 November): a beta splits the broad permission and lets a grant be narrowed to folders or set to ask each time, answering AI agents that request full access. Existing grants carry over, so review what agents and apps already hold on your Macs.

## Checklist for Monday

- [ ] Upgrade Arize Phoenix to 15.7.0 or later, and if it is network-reachable confirm authentication is on; `applyChatTemplate` leaked secrets unauthenticated on a default install.
- [ ] Upgrade NLTK to 3.10.4 (last week's `CVE-2026-126303` and this week's 23 CVEs all clear there) and run `pip list | grep nltk` across your images.
- [ ] Update the MCP Python SDK to 1.30.0 or 2.2.0, and if you use `ClientCredentialsOAuthProvider` or `PrivateKeyJWTOAuthProvider` pass `issuer=` so a server cannot redirect your credentials.
- [ ] For every MCP server you run on localhost (Todoist 10.3.3, Obsidian 0.5.7, Jupyter 1.0.3, and your own), require a token or validate `Host` and `Origin`; binding to `127.0.0.1` does not keep the browser out.
- [ ] Put k8s-mcp-server on 1.1.0 and sqlite-mcp on 0.1.2, and check no MCP server binds `0.0.0.0` without auth or runs "read-only" while taking writes.
- [ ] Search build logs and lockfiles for `agent-framework` 1.26.1 or 1.26.2, and if either is present rotate every credential that host and its CI could reach.
- [ ] If you route model traffic through a gateway, pin the upstream providers yourself, seed a canary credential into a prompt path and alert on its use, and treat a zero-retention flag as covering only the first hop.
- [ ] Track MONAI Label `GHSA-qc22-rmgc-vg94` for a fixed release, and turn on authentication if you run it (off by default).

## References

By layer:

- [GHSA-jw3m-8q7m-f35r: LLMKube arbitrary host file read via hostPath](https://github.com/advisories/GHSA-jw3m-8q7m-f35r){:target="_blank"} - GitHub Advisory Database
- [GHSA-qx49-fqc8-xw99: MCP Python SDK OAuth credentials to a server-chosen authorization server](https://github.com/modelcontextprotocol/python-sdk/security/advisories/GHSA-qx49-fqc8-xw99){:target="_blank"} - Model Context Protocol project
- [GHSA-m932-63jc-4jqq: k8s-mcp-server missing authentication on default SSE transport](https://github.com/advisories/GHSA-m932-63jc-4jqq){:target="_blank"} - GitHub Advisory Database
- [GHSA-gfv3-m769-5xm3: Arize Phoenix format-string injection in applyChatTemplate](https://github.com/Arize-ai/phoenix/security/advisories/GHSA-gfv3-m769-5xm3){:target="_blank"} - Arize Phoenix project
- [GHSA-3mgh-93r5-r7jf: CodeWhale exec_shell classifier auto-approves exec flags](https://github.com/advisories/GHSA-3mgh-93r5-r7jf){:target="_blank"} - GitHub Advisory Database
- [GHSA-x8q7-6c7m-52gw: NLTK java() JVM/JAR sandbox bypass](https://github.com/advisories/GHSA-x8q7-6c7m-52gw){:target="_blank"} - GitHub Advisory Database
- [GHSA-7mxv-7h3q-9324: NLTK dot2img runs Graphviz without the trusted-exec chokepoint](https://github.com/advisories/GHSA-7mxv-7h3q-9324){:target="_blank"} - GitHub Advisory Database
- [GHSA-9x8x-crjq-g64h: Jupyter MCP Server management routes hijack the backend via DNS rebinding](https://github.com/datalayer/jupyter-mcp-server/security/advisories/GHSA-9x8x-crjq-g64h){:target="_blank"} - Jupyter MCP Server project
- [GHSA-24pw-rf5p-mgp7: Jupyter MCP Server unauthenticated management routes](https://github.com/advisories/GHSA-24pw-rf5p-mgp7){:target="_blank"} - GitHub Advisory Database
- [GHSA-q433-mp6c-8v4f: sqlite-mcp read-only bypass via statement chaining](https://github.com/advisories/GHSA-q433-mp6c-8v4f){:target="_blank"} - GitHub Advisory Database
- [GHSA-2m9m-6cr5-9x25: mssql-mcp SQL injection in write tools](https://github.com/advisories/GHSA-2m9m-6cr5-9x25){:target="_blank"} - GitHub Advisory Database
- [GHSA-jq3q-62jc-fqq9: code-review-mcp-server command injection](https://github.com/advisories/GHSA-jq3q-62jc-fqq9){:target="_blank"} - GitHub Advisory Database
- [GHSA-4rj4-rcrq-gr92: safari-mcp JavaScript injection through tool arguments](https://github.com/advisories/GHSA-4rj4-rcrq-gr92){:target="_blank"} - GitHub Advisory Database
- [GHSA-8hp3-xxrg-v9c8: CodeWhale automation auto_approve defaults to true](https://github.com/advisories/GHSA-8hp3-xxrg-v9c8){:target="_blank"} - GitHub Advisory Database
- [GHSA-r7wg-f7r2-8wf7: axon-mcp download of code without integrity check](https://github.com/advisories/GHSA-r7wg-f7r2-8wf7){:target="_blank"} - GitHub Advisory Database
- [Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog){:target="_blank"} - CISA

Case files:

- [GHSA-764m-3ppc-5m73: malicious code in agent-framework (PyPI)](https://github.com/advisories/GHSA-764m-3ppc-5m73){:target="_blank"} - GitHub Advisory Database
- [MAL-2026-20101: agent-framework credential stealer](https://github.com/ossf/malicious-packages/blob/main/osv/malicious/pypi/agent-framework/MAL-2026-20101.json){:target="_blank"} - OpenSSF
- [Cybercriminals compromise Microsoft's Agent Framework PyPI package to steal credentials](https://www.bleepingcomputer.com/news/security/microsofts-agent-framework-pypi-package-compromised-to-steal-credentials/){:target="_blank"} - BleepingComputer
- [Data processing incidents on inference providers](https://openrouter.ai/blog/announcements/data-processing-incidents-on-inference-providers/){:target="_blank"} - OpenRouter
- [Three breaches, endless leaks: your AI supply chain needs canaries](https://www.dragos.com/blog/ai-supply-chain-security-canary-tokens){:target="_blank"} - Dragos
- [Security event: prompt leakage for requests in June and July of 2026](https://www.together.ai/blog/security-event-prompt-leakage-for-requests-in-june-and-july-of-2026){:target="_blank"} - Together AI
- [November preliminary incident report](https://phala.com/posts/phala-november-preliminary-incident-report){:target="_blank"} - Phala
- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/){:target="_blank"} - OWASP GenAI Security Project
- [MITRE ATLAS matrix](https://atlas.mitre.org/){:target="_blank"} - MITRE

Anatomy:

- [GHSA-833v-v58w-294h: Todoist MCP Server DNS rebinding](https://github.com/advisories/GHSA-833v-v58w-294h){:target="_blank"} - GitHub Advisory Database
- [GHSA-mx6p-3fg7-v6pj: obsidian-sync-mcp missing Host/Origin validation](https://github.com/advisories/GHSA-mx6p-3fg7-v6pj){:target="_blank"} - GitHub Advisory Database

Research:

- [Chinese-nexus actors' Cairn AI agents found a zero-day on their own](https://aws.amazon.com/blogs/security/chinese-nexus-actors-cairn-ai-agents-found-a-zero-day-on-their-own/){:target="_blank"} - Amazon Threat Intelligence
- [GHSA-qc22-rmgc-vg94: MONAI Label path traversal in datastore upload endpoints](https://github.com/Project-MONAI/MONAILabel/security/advisories/GHSA-qc22-rmgc-vg94){:target="_blank"} - Project MONAI
- [Protecting your privacy](https://signal.org/blog/protecting-your-privacy/){:target="_blank"} - Signal
- [macOS 27.3 warns against giving AI apps full Mac control](https://www.macrumors.com/2026/11/16/macos-27-3-full-desk-access/){:target="_blank"} - MacRumors

Notes on sources:

- Dating: for every entry the fix shipped before its advisory or CVE record appeared in this window. The releases I confirmed on their registries are NLTK 3.10.4 (21 October), MCP Python SDK 1.30.0 and 2.2.0 (7 September), Arize Phoenix 15.7.0 (13 May), Jupyter MCP Server 1.0.3 (11 July), Todoist MCP 10.3.3 (24 June), obsidian-sync-mcp 0.5.7 (10 July), CodeWhale 0.9.0 (16 July), code-review-mcp-server 1.5.3 (6 July), safari-mcp 2.22.15 (10 October) and the mssql-mcp servers (29 June). I include each item because its GitHub Advisory Database or CVE record first appeared between 16 and 22 November; the agent-framework malware advisory, the three DNS-rebinding advisories, the MCP SDK advisory and the NLTK batch are all dated inside the window.
- NLTK: last week's post listed `3.10.4` for a different CVE (`CVE-2026-126303`); this week's 23 records are new and all resolve in that same release. `CVE-2026-128138` completes hardening begun in 3.10.3, and `CVE-2026-126296` restores an earlier Huntr assignment (`CVE-2026-15367`).
- Arize Phoenix: the advisory text names v14.8.0 as the latest confirmed-vulnerable release, while the GitHub Advisory Database range ends at 15.7.0, which I follow as the fix.
- axon-mcp and MONAI Label: no fixed version is published, and the entries say so.
- OpenRouter: the vendor post names no providers and gives no customer guidance; the three mechanisms, the canary method and the figures (approximate, as Dragos words them) are from the Dragos report, and the four provider notices OpenRouter links (Atlas Cloud, Novita, Phala, Together) are not identified as the off-boarded three.
- Amazon: the Cairn and MONAI Label report is a separate publication from the 16 November one on agents hunting exposed AI tools that I cited last week, and the attribution to a Chinese-nexus actor is Amazon's, at moderate confidence.
- Signal: the feature names and the threat are as BleepingComputer relays Signal's post of 22 November; I cite Signal's post as the primary but read the coverage for the detail.
- CISA KEV: the catalogue (version 2026.11.20) lists none of the CVEs in this post.
- MITRE ATLAS: the technique IDs are from the current `dist/ATLAS.yaml` on `mitre-atlas/atlas-data`.
