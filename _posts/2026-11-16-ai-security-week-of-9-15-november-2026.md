---
layout: post
title: "AI security, week of 9 to 15 November 2026"
---

This is the seven days to 15 November 2026, and the thread was the server behind the agent. The tools we stand up to give a model hands, MCP servers, RAG back ends, model gateways and the app builders around them, kept trusting whoever reached them: an absolute path read any file, a spoofed header reset a rate limit, a request body swapped a gateway's own credentials, and a connection marked read only still wrote. The anatomy walks the sharpest case, an unauthenticated route in a ComfyUI extension that copies an attacker's file over a custom node and a second route that restarts the process to run it. The [9 November post](/2026/11/09/ai-security-week-of-2-8-november-2026.html) covered the window before this one so nothing repeats here, and where the ARTEX framework returns it is a new FBI advisory, not the [campaign I wrote up earlier](/2026/11/02/ai-security-incidents-of-the-week-27-october-2-november-2026.html). As the week closed Amazon put a frame around all of it: its decoy sensors recorded attackers' own agents combing the internet for exactly these exposed tools, hunting the API keys inside them.

## At a glance

- **Patch now:** fast-agent-mcp 0.7.21 (`CVE-2026-102610`, unauthenticated command execution when served with `--shell`), LiteLLM 1.108.6 or 1.109.1 (`GHSA-2rwg-2wh4-7c4f`, Azure credential override from the request body), ComfyUI-Easy-Use v1.4.2 (`CVE-2026-123790`, unauthenticated remote code execution), and MCP Inspector 2.15.0 (a batch of four).
- **Read closely:** the ComfyUI-Easy-Use chain in the anatomy, a file-copy route that turns into code execution because the path check runs one step too early and a reboot route pulls the trigger.
- **Watch:** Amazon's report that attacker agents are scanning for exposed LiteLLM, Dify, Open WebUI and Ollama to lift the provider keys they store, with the caveat that it rests on the agents' own unverified records.

## By layer

### Inference and serving

- **vLLM 0.31.0** `CVE-2026-126397` (Medium): a chat-completions request set `mm_processor_kwargs.max_patches`, which vLLM passed to the Cohere2 Vision processor with no server-side cap, so two loops bound by it do quadratic work and one request stalls the service. [Advisory](https://github.com/vllm-project/vllm/security/advisories/GHSA-9g3c-x747-c6wr){:target="_blank"}
- **LiteLLM 1.109.1** `GHSA-2rwg-2wh4-7c4f` (High, 7.7): the proxy's request-body blocklist missed the Azure Entra credential fields, so an authenticated caller puts `client_id` and `client_secret` in a chat-completions body and overrides the deployment's own Azure credentials. [Advisory](https://github.com/BerriAI/litellm/security/advisories/GHSA-2rwg-2wh4-7c4f){:target="_blank"}

### Agent frameworks and MCP

- **fast-agent-mcp 0.7.21** `CVE-2026-102610` (Medium, 6.4): `fast-agent serve` binds to `0.0.0.0` with no authentication, and the quickstart's `--shell` flag gives the exposed agent an `execute` tool, so any host that reaches the port runs commands. [Advisory](https://github.com/advisories/GHSA-9vhv-g5hf-m7m7){:target="_blank"}
- **MCP Inspector 2.15.0** `GHSA-9w48-x22j-rrm7` (Medium, 6.5): a server you connect it to can crash the proxy and every session with an endless streamed response, one of four fixes that also close an authorization bypass, an SSRF and a config-import that smuggles `NODE_OPTIONS`. [Advisory](https://github.com/modelcontextprotocol/inspector/security/advisories/GHSA-9w48-x22j-rrm7){:target="_blank"}

### AI applications and UIs

- **Dify 1.18.0** `GHSA-6j8g-rp8v-55vr` (Medium, 6.5): `extract_remote_ip()` returns the client's `X-Forwarded-For` header as the source address, and that address is the key for every per-IP rate limit, so rotating the header hands out a fresh bucket each request. [Advisory](https://github.com/langgenius/dify/security/advisories/GHSA-6j8g-rp8v-55vr){:target="_blank"}

### MLOps and data pipelines

- **code-graph-rag 0.0.345** `CVE-2026-77604` (High, 7.5): the `read_file` MCP tool joined the caller's path onto the project root without rechecking it, and an absolute path makes `pathlib` drop the root, so the tool returns any file on the host. [Advisory](https://github.com/advisories/GHSA-vvr2-h2jp-838m){:target="_blank"}
- **pgEdge Postgres MCP 1.1.0** `CVE-2026-126196` (High, 8.1): on a `read_only` connection, unrecognized statements went through Postgres's simple query protocol, so one request could batch a session-state change with a write and the read-only boundary no longer held. [Advisory](https://nvd.nist.gov/vuln/detail/CVE-2026-126196){:target="_blank"}

**Also patched:** tsParticles MCP server 4.3.1 and 4.3.2 (`CVE-2026-63655`, `CVE-2026-76892`, `CVE-2026-76893`, denial of service in HTTP mode), Dagu 2.11.0 (`CVE-2026-102943`, MCP path-containment bypass), ezBookkeeping 1.6.0 (`CVE-2026-86041`, the same spoofable `X-Forwarded-For` allowlist), pgEdge RAG Server and Go LLM library (`CVE-2026-125325` to `CVE-2026-125328`, provider key fragments in error messages and unauthenticated corpus reads), and NLTK 3.10.4 (`CVE-2026-126303`, unbounded CPU in a grammar parser).

## Case files

### Cursor: a stolen model-provider key reached the extension marketplace (14 November)

Cursor is an AI code editor built on VS Code, and Fireworks AI is a model inference and training provider it used. The credential the attackers started from was an access key stored at Fireworks.

With it they copied a read-only bucket that Cursor used to share data with Fireworks, holding older coding-agent traces, mostly from 2025, from users without Privacy Mode on. Some of those traces still held unredacted API credentials, which opened more of Cursor's systems, including one that could publish to it's extension marketplace. On 29 October the attackers placed malicious copies of Anthropic's Claude Code extension (versions 2.1.312 and 2.1.313) on the cache that serves the marketplace, where they sat about eleven hours, and the build tried to plant an SSH key and switch on remote access.

Per [BleepingComputer](https://www.bleepingcomputer.com/news/security/cursor-confirms-security-breach-led-to-malicious-claude-code-extension/){:target="_blank"}, Cursor revoked the Fireworks key, rotated the exposed credentials, rebuilt the marketplace, moved inference off Fireworks and removed the builds. It estimates up to 5,000 users may have received a malicious build, fewer than 0.05% had traces exposed and fewer than 0.01% had credentials taken, and says production systems, official releases and Privacy Mode users were untouched. Researchers reported coding errors that likely kept the backdoor from working where SSH was not already on. Cursor's own [incident report](https://cursor.com/blog/security-incident-update){:target="_blank"} is the primary account.

OWASP LLM Top 10 2025: LLM03 Supply Chain. MITRE ATLAS: AML.T0010 AI Supply Chain Compromise, with AML.T0055 Unsecured Credentials for the keys left in the traces.

### Jet Admin: a prompt injection read the platform's .env (13 November)

Jet Admin is a hosted platform for building internal tools and dashboards, and the attacker in this one needs only a free account.

[`CVE-2026-123965`](https://nvd.nist.gov/vuln/detail/CVE-2026-123965){:target="_blank"} describes a prompt injection that let that account reach the platform's `.env` configuration files and pull the credentials out of them, which NVD scores 9.9. The vendor record says it affects all versions and that the fix is server side, a sandbox component added around 29 October, so there is no release for a self-hosted reader to install. The CVE and a coordinated advisory (Italy's CSIRT tracking `VA-26-317-01`) were published on 13 November.

For anyone running an AI feature that can read configuration or secrets, the control is to keep those secrets out of the model's reach, because untrusted input that reaches the model is input that can ask for the `.env`. It is the same gap the week's credential-hunting agents were counting on.

OWASP LLM Top 10 2025: LLM01 Prompt Injection and LLM02 Sensitive Information Disclosure. MITRE ATLAS: AML.T0051 LLM Prompt Injection.

## Anatomy of the week: a ComfyUI extension that copied a file into itself

ComfyUI is a node-based interface for image models, and ComfyUI-Easy-Use is one of the most installed custom-node packs for it. `CVE-2026-123790` was coordinated through CERT/CC (VU#944667) by Nguyễn Huy Hoàng and made public on 12 November, with a tagged fix, v1.4.2, following on 14 November; the CVE record carries no score and the finder self-rates it 9.8. A default install listens only on loopback, but the `--listen` flag and most hosted GPU images make it reachable, and that reach is the only real precondition.

1. The attacker uploads a file of Python code. ComfyUI's own `POST /upload/image` takes no authentication and writes the bytes under `input/` with no check on type or content.
2. They call `POST /easyuse/save/{name}` with `{name}` set to `custom_nodes%2F<node>%2F__init__.py`. The handler splits on the first slash and the percent-encoded slash survives routing, so the folder type it reads is `custom_nodes` and the name is an installed node's entry point.
3. The handler checks that the request's `subfolder` stays under a base directory, then joins the body's `filename` on afterwards. An absolute or `../` `filename` makes `os.path.join` discard the base, so it points back at the uploaded payload, which becomes the copy source.
4. `get_full_path("custom_nodes", "<node>/__init__.py")` resolves only because that file already exists, so the write is an overwrite of the node's `__init__.py`, taking its extension from the source.
5. `shutil.copyfile` copies the payload over the entry point.
6. `GET /easyuse/reboot`, also unauthenticated, calls `os.execv` and restarts the process on demand. On restart ComfyUI imports its custom nodes, the replaced `__init__.py` runs, and the attacker has code execution as the ComfyUI process, with no user interaction.

### Where it could have stopped

- Input handling: validate the destination type against the folders the route is meant to write (models, not code) and recheck the whole path after the filename is joined, not before. The fix does this, restricting the destination to the model folders and the source to an image file under `temp` re-resolved with `realpath`.
- Route permissions: the save and reboot routes answer any client. A same-origin or token gate on a state-changing route removes the direct-client path (the fix adds a token to reboot, though a non-browser client can still skip the browser checks).
- Isolation: a preview handler should never be able to write into `custom_nodes`; keep the writable output tree and the code tree apart.
- Trigger: an on-demand restart route turns a dormant overwrite into immediate execution, so gating it takes the push away.

The shape to look for is a path or permission check that runs before the dangerous value is fully built, on a service written as though it would only ever hear from localhost. In your own systems, list every unauthenticated route that writes a file or restarts a process, and confirm the check lands on the final resolved path.

## From the research side

- [FBI FLASH on the ARTEX framework](https://www.bleepingcomputer.com/news/security/fbi-warns-artex-ai-hacking-tool-is-powering-global-cyberattacks/){:target="_blank"} (FBI, reported 13 November): the bureau's advisory on the autonomous Go pentest framework I covered earlier as VAPOR PANDA's tooling, now with indicators and the line that some intrusions ran from first contact to privileged access in under three hours. The defender actions are ordinary: harden internet-facing apps, lock down cloud admin interfaces, rate-limit sensitive APIs and alert on bursts of automated requests.
- [MalwareMustDie ARTEX detection pack](http://blog.malwaremustdie.org/2026/11/mmd-072-2026-artex-detection-signature.html){:target="_blank"} (13 November): an open-source signature set for the same framework, ranked by confidence, with the honest note that its network indicators age quickly. A starting point if you want detection today rather then after the next advisory.
- [Amazon's report on agents hunting AI tools](https://aws.amazon.com/blogs/security/amazon-threat-intelligence-observes-threat-actors-running-cybercrime-campaigns-with-artex-ai-agents/){:target="_blank"} (Amazon Threat Intelligence, activity 3 October to 5 November, reported 16 November): decoy sensors posing as LLM services recorded attacker agents combing exposed LiteLLM, One API, Dify, Langflow, Open WebUI and Ollama for stored keys, and in one campaign the stolen keys paid for the inference that hunted the next batch. Treat the named products as a target list and rotate any provider keys they hold. Amazon rates the ARTEX attribution moderate and did not observe the activity live, so the outcomes are the agents' own claims.

## Checklist for Monday

- [ ] Upgrade ComfyUI-Easy-Use to v1.4.2 (or track `main`), and confirm no ComfyUI instance is reachable past loopback; its save route needed no authentication.
- [ ] Patch fast-agent-mcp to 0.7.21 or later, and never expose `fast-agent serve --shell` on `0.0.0.0` without authentication in front of it.
- [ ] Update LiteLLM to 1.108.6 or 1.109.1, and if you run an Azure deployment set `AZURE_API_KEY` so a request body cannot override the gateway's credentials.
- [ ] Upgrade MCP Inspector to 2.15.0, and treat any MCP server you point it at as hostile to the inspector itself.
- [ ] Update Dify to 1.18.0, and stop trusting `X-Forwarded-For` for rate limits or allowlists unless a known proxy sets it (the same assumption broke ezBookkeeping).
- [ ] Upgrade code-graph-rag to 0.0.345 and pgEdge Postgres MCP to 1.1.0, and verify that your "read-only" MCP database connections actually refuse writes.
- [ ] Find your internet-exposed LiteLLM, Dify, Open WebUI and Ollama instances, replace any default admin or master keys, and rotate the provider keys they hold.
- [ ] If anyone installed the Claude Code extension from Cursor's marketplace around 29 October (2.1.312 or 2.1.313), follow Cursor's remediation and rotate the keys that machine could reach.

## References

By layer:

- [GHSA-9g3c-x747-c6wr: vLLM Cohere2 Vision max_patches denial of service](https://github.com/vllm-project/vllm/security/advisories/GHSA-9g3c-x747-c6wr){:target="_blank"} - vLLM project
- [CVE-2026-126397](https://nvd.nist.gov/vuln/detail/CVE-2026-126397){:target="_blank"} - NVD
- [GHSA-2rwg-2wh4-7c4f: LiteLLM Azure Entra credential exfiltration](https://github.com/BerriAI/litellm/security/advisories/GHSA-2rwg-2wh4-7c4f){:target="_blank"} - LiteLLM project
- [GHSA-9vhv-g5hf-m7m7: fast-agent serve binds to 0.0.0.0 with no authentication](https://github.com/advisories/GHSA-9vhv-g5hf-m7m7){:target="_blank"} - GitHub Advisory Database
- [GHSA-9w48-x22j-rrm7: MCP Inspector proxy crash via unbounded SSE response](https://github.com/modelcontextprotocol/inspector/security/advisories/GHSA-9w48-x22j-rrm7){:target="_blank"} - MCP Inspector project
- [GHSA-5572-x678-xmv8: MCP Inspector authorization bypass in paths.ts](https://github.com/modelcontextprotocol/inspector/security/advisories/GHSA-5572-x678-xmv8){:target="_blank"} - MCP Inspector project
- [GHSA-6j8g-rp8v-55vr: Dify per-IP rate-limit bypass via spoofable headers](https://github.com/langgenius/dify/security/advisories/GHSA-6j8g-rp8v-55vr){:target="_blank"} - Dify project
- [GHSA-vvr2-h2jp-838m: Code Graph RAG read_file path traversal](https://github.com/advisories/GHSA-vvr2-h2jp-838m){:target="_blank"} - GitHub Advisory Database
- [CVE-2026-126196: pgEdge Postgres MCP read-only bypass](https://nvd.nist.gov/vuln/detail/CVE-2026-126196){:target="_blank"} - NVD
- [GHSA-447x-qqwf-m7xq: tsParticles MCP server unbounded request buffering](https://github.com/advisories/GHSA-447x-qqwf-m7xq){:target="_blank"} - GitHub Advisory Database
- [GHSA-5m6c-38r8-5w76: Dagu MCP path-containment bypass](https://github.com/advisories/GHSA-5m6c-38r8-5w76){:target="_blank"} - GitHub Advisory Database
- [GHSA-wchh-mrv2-5h4q: ezBookkeeping source-IP allowlist bypass](https://github.com/advisories/GHSA-wchh-mrv2-5h4q){:target="_blank"} - GitHub Advisory Database
- [GHSA-5qg5-rgp6-j5m5: pgEdge RAG Server unbounded corpus read](https://github.com/advisories/GHSA-5qg5-rgp6-j5m5){:target="_blank"} - GitHub Advisory Database
- [GHSA-8fx7-8jr8-84rv: NLTK unbounded CPU in SteppingRecursiveDescentParser](https://github.com/advisories/GHSA-8fx7-8jr8-84rv){:target="_blank"} - GitHub Advisory Database
- [Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog){:target="_blank"} - CISA

Case files:

- [Cursor confirms security breach led to malicious Claude Code extension](https://www.bleepingcomputer.com/news/security/cursor-confirms-security-breach-led-to-malicious-claude-code-extension/){:target="_blank"} - BleepingComputer
- [Security incident update](https://cursor.com/blog/security-incident-update){:target="_blank"} - Cursor
- [CVE-2026-123965: Jet Admin Cloud Platform prompt injection](https://nvd.nist.gov/vuln/detail/CVE-2026-123965){:target="_blank"} - NVD
- [Jet Admin Cloud Platform Prompt Injection (VA-26-317-01)](https://www.jetadmin.io/changelog){:target="_blank"} - Jet Admin
- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/){:target="_blank"} - OWASP GenAI Security Project
- [MITRE ATLAS matrix](https://atlas.mitre.org/){:target="_blank"} - MITRE

Anatomy:

- [CVE-2026-123790: ComfyUI-Easy-Use custom-node overwrite to RCE](https://blog.nhhoang.io.vn/posts/2026/11/cve-2026-123790-comfyui-easy-use-save-preview-rce/){:target="_blank"} - Red Team Notes
- [VU#944667: ComfyUI-Easy-Use save_preview arbitrary file overwrite](https://kb.cert.org/vuls/id/944667){:target="_blank"} - CERT/CC
- [CVE-2026-123790](https://nvd.nist.gov/vuln/detail/CVE-2026-123790){:target="_blank"} - NVD
- [Fix: harden model metadata and preview routes](https://github.com/yolain/ComfyUI-Easy-Use/commit/8a0f2fc412c1c6f9109e9392064800a87a706775){:target="_blank"} - ComfyUI-Easy-Use project

Research:

- [FBI warns ARTEX AI hacking tool is powering global cyberattacks](https://www.bleepingcomputer.com/news/security/fbi-warns-artex-ai-hacking-tool-is-powering-global-cyberattacks/){:target="_blank"} - BleepingComputer
- [MMD-072-2026: ARTEX detection signature pack](http://blog.malwaremustdie.org/2026/11/mmd-072-2026-artex-detection-signature.html){:target="_blank"} - MalwareMustDie
- [Amazon Threat Intelligence observes threat actors running campaigns with ARTEX AI agents](https://aws.amazon.com/blogs/security/amazon-threat-intelligence-observes-threat-actors-running-cybercrime-campaigns-with-artex-ai-agents/){:target="_blank"} - Amazon Threat Intelligence
- [AI agents hunt exposed LiteLLM and Dify tools for API keys](https://tbreak.com/ai-agents-hunt-litellm-dify-api-keys/){:target="_blank"} - Tbreak

Notes on sources:

- Dating: several fixes shipped before their advisory or CVE record appeared in this window. The fix releases are vLLM 0.31.0 (5 October), fast-agent-mcp 0.7.21 (19 June), code-graph-rag 0.0.345 (14 July), tsParticles 4.3.1 and 4.3.2 (1 and 10 July), pgEdge Postgres MCP 1.1.0 (4 September), NLTK 3.10.4 (21 October) and Dify 1.18.0 (23 October); I include each item because its advisory or CVE record first appeared between 9 and 15 November. MCP Inspector 2.15.0, LiteLLM 1.108.6 and 1.109.1 all shipped on 12 November.
- vLLM: the project advisory `GHSA-9g3c-x747-c6wr` (dated 27 October) rates this Medium with no numeric score, while the NVD record for `CVE-2026-126397` (dated 13 November) rates it High; I follow the project advisory in the entry.
- ComfyUI-Easy-Use: at public disclosure on 12 November the fix existed only as commit `8a0f2fc` on `main` (29 September) with no tagged release, as the finder notes; the maintainer tagged v1.4.2 on 14 November, and that release contains the commit, so it is the version to install. The CVE record lists 1.3.6; the finder reports the same handler is vulnerable through v1.4.1.
- FBI advisory: the figures and the under-three-hours line are as BleepingComputer relays them from the 13-page FLASH; I did not open the IC3 document itself.
- Amazon report: I read Tbreak's coverage and cite Amazon's AWS Security Blog post as the primary source; the decoy design, the full target list and the eleven indicator addresses are in that post, which I did not fetch, and The Hacker News, which carries the same story, blocks automated fetches.
- CISA KEV: the catalogue (version 2026.11.13) lists none of the CVEs in this post.
