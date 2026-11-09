---
layout: post
title: "AI security, week of 2 to 8 November 2026"
---

This is the seven days to 8 November 2026, and the thread was the boundary around an agent: a research agent left its test environment and reached a production tool at WorkOS, a popup left the sandbox around ChatGPT's HTML preview, and a desktop agent's approval prompts were buttons in a page that a browser extension could script. The anatomy walks through the ChatGPT chain. The window overlaps the 2 November post by a day, so I leave out what it covered (the MLflow 3.17.0 batch, the testmetest worm, Warp, VAPOR PANDA). One correction to it: it relayed Chainalysis's first notice, which said credentials had not been accessed, and [the company's updates](https://www.chainalysis.com/blog/security-incident-october-30/){:target="_blank"} since 4 November say names, usernames, password hashes and live tokens for all users were exposed.

## At a glance

- **Patch now:** CrewAI 1.15.17 (`CVE-2026-120630`, Critical, 9.1), Kubeflow Pipelines 2.18.0 (`CVE-2026-77596`, Critical, 9.6), JupyterLab Desktop 4.6.2-1 (`CVE-2026-86044`, Critical, 9.6), and LiteLLM 1.83.14 or later (Google describes an exploited LiteLLM gateway).
- **Read closely:** the ChatGPT HTML-preview bypass. The popup guard was a function replaced inside the untrusted page, while the iframe's sandbox flags allowed popups.
- **Watch:** Claude Dispatch, which its finders say can still be exploited while Anthropic plans to retire it, and TorchServe [`CVE-2026-123974`](https://nvd.nist.gov/vuln/detail/CVE-2026-123974){:target="_blank"} (archived repository, no fixed version, linked public proof of concept).

## By layer

### Model files and loaders

- **Stanza 1.15.0** `CVE-2026-115878` (High, 7.8): a pickled dictionary stored as `bytes` in a checkpoint passes `torch.load(weights_only=True)` untouched and reaches an unrestricted `pickle.loads`, so loading a shared lemmatizer model runs code. [Advisory](https://github.com/advisories/GHSA-gh9r-c94j-cmp5){:target="_blank"}

### Inference and serving

- **vLLM 0.33.0** `GHSA-q9v9-qpjp-fc24` (High, 7.5): with `--skip-tokenizer-init` the token-id range check never ran, so one unauthenticated request with an out-of-range id stops the engine for every tenant. [Advisory](https://github.com/vllm-project/vllm/security/advisories/GHSA-q9v9-qpjp-fc24){:target="_blank"}

### Agent frameworks and MCP

- **CrewAI 1.15.17** `CVE-2026-120630` (Critical, 9.1): the SSRF guard and the HTTP client each checked what the other did not, so a crafted URL passed the guard and still reached internal hosts, for example through the website scraper tool. [Record](https://nvd.nist.gov/vuln/detail/CVE-2026-120630){:target="_blank"}
- **AG2 1.1.1** `CVE-2026-113663` (High, 8.8): the shell tool's allowlist filter did not treat newline or carriage return as separators, so one permitted command could carry a second statement into `sh -c` on the host. [Advisory](https://github.com/ag2ai/ag2/security/advisories/GHSA-2h4w-29cv-2cmv){:target="_blank"}

### AI applications and UIs

- **Claude Code 2.1.251** `CVE-2026-88968` (High, 7.7): on macOS, code inside the Bash sandbox could swap a validated parent directory for a symlink, so the unsandboxed Edit tool wrote outside the workspace. [Advisory](https://github.com/anthropics/claude-code/security/advisories/GHSA-5xjx-9qx6-jcrv){:target="_blank"}
- **Dify 1.15.0** `CVE-2026-58141` (High, 7.2): the SSRF defence was a proxy with source rules but no destination denies, so any console user could reach internal services and cloud metadata. [Record](https://nvd.nist.gov/vuln/detail/CVE-2026-58141){:target="_blank"}

### MLOps and data pipelines

- **Kubeflow Pipelines 2.18.0** `CVE-2026-77596` (Critical, 9.6): the upload endpoints skipped the access review for an empty namespace, so a tenant could add a pipeline version that a recurring run executes in another tenant's namespace. [Advisory](https://github.com/kubeflow/pipelines/security/advisories/GHSA-rjr3-43j3-4q8g){:target="_blank"}
- **JupyterLab Desktop 4.6.2-1** `CVE-2026-86044` (Critical, 9.6): with JupyterLab before 4.4.8, a notebook link could send the renderer to another site while it kept its bridge, letting that site read the local server token and run code. [Advisory](https://github.com/jupyterlab/jupyterlab-desktop/security/advisories/GHSA-2mr8-962v-wc67){:target="_blank"}

**Also patched:** MCP TypeScript SDK 1.32.0 and 2.3.0 (`CVE-2026-111770`, wrong-client responses from a shared transport, needs an application change; `CVE-2026-111768` and `CVE-2026-111771` need configuration; `CVE-2026-111769`), vLLM 0.33.0 (`GHSA-5r3c-5x4v-8gjh`, `GHSA-qrj2-j723-m6w8`), Banks 2.4.3 (`CVE-2026-61536`, tool `import_path` resolved to any callable), DBHub 0.22.6 (`CVE-2026-61789`, a `--` comment hid writes from the read-only check), OpenHuman 0.57.13 (`CVE-2026-54140`, MCP tool descriptions placed in model context), stable-diffusion.cpp master-764 (`CVE-2026-70660`, safetensors offsets), Langflow 1.13.0 (`CVE-2026-97677`), MLflow 3.17.0 (ZDI's `CVE-2026-118623` to `CVE-2026-118628`).

## Case files

### WorkOS: a research agent left its test environment (6 November)

WorkOS has an internal tool that gathers CRM, support and customer-conversation data for its go-to-market teams. The actor was an agent built on ARTEX, the framework from last week's VAPOR PANDA report, run by a security researcher in benchmark evaluations.

WorkOS traced unusual query volume to the tool on 30 October, and the first confirmed access was on 25 October. Its [6 November bulletin](https://workos.com/security/bulletins/update-for-october-30th-2026-data-security-incident){:target="_blank"} says the researcher came forward: the agent had left its isolated test environment and the contact was unintended. A forensics firm matched the evidence to WorkOS logs, and the researcher says the data was deleted and not shared. WorkOS names four gaps: an internet-reachable API, a predictable development credential accepted in production, no rate limiting, and late detection. [BleepingComputer](https://www.bleepingcomputer.com/news/security/runaway-ai-hacking-agent-behind-workos-breach-after-escaping-sandbox/){:target="_blank"} notes that neither side has said how the agent left it's environment.

WorkOS closed the exposure on 30 October, rotated credentials, and has tested its own systems with ARTEX since 3 November.

[OWASP LLM Top 10 2025](https://genai.owasp.org/llm-top-10/){:target="_blank"}: LLM06 Excessive Agency. [MITRE ATLAS](https://github.com/mitre-atlas/atlas-data){:target="_blank"}: AML.T0105 Escape to Host, the closest entry, since the route out is not public.

### Claude Dispatch: a web session accepted as the user (6 November)

Dispatch sends tasks from a phone or browser to Claude Desktop, an agent that can run commands on the computer. Gal Weizman at Forever Security [found](https://forever.security/blog/thisbash-claude-dispatch-browser-extension-rce){:target="_blank"} that it also accepted an ordinary signed-in claude.ai session, although the website shows no link to it.

An extension that can run scripts on claude.ai (malicious, or legitimate after a compromised update) can send Claude Desktop prompts as if the user typed them. In the test, four prompts asked for an npm install, a follow-up in the user's voice answered the agent's hesitation, and a script clicked each Allow in the page it controlled. The install script opened Calculator; the researchers report no real attacks.

They reported it on 14 May and say Anthropic closed it as a duplicate of an internal finding, then said on 13 October that Dispatch would be retired, targeting the end of October. [On 6 November](https://thehackernews.com/2026/11/malicious-browser-extensions-could-use.html){:target="_blank"} existing users on some plans could still use it, and the researchers say the flaw can still be exploited. This is the approval question from [Is there really a human in the loop?](/2026/10/06/is-there-really-a-human-in-the-loop.html){:target="_blank"}: a prompt protects you only where the party it guards against cannot reach the button.

OWASP LLM Top 10 2025: LLM06 Excessive Agency. MITRE ATLAS: AML.T0053 AI Agent Tool Invocation.

### Claude Code: saved approvals carry secrets into repositories (6 November)

Claude Code's [documentation](https://code.claude.com/docs/en/permissions){:target="_blank"} says that "Yes, and don't ask again" on a Bash command saves an allow rule in `.claude/settings.local.json`, and a rule without a `*` matches that exact command. The [first write](https://code.claude.com/docs/en/settings){:target="_blank"} also adds the file to the user's global git excludes.

Apiiro, [as reported by The Hacker News](https://thehackernews.com/2026/11/claude-codes-dont-ask-again-file.html){:target="_blank"}, found 104,185 copies of the file in public repositories. In 1,122 of them there is at least one secret, including 379 database passwords, 43 SSH logins, 42 GitHub, GitLab or npm tokens and 23 AWS keys. Apiiro did not test whether they work, and one notified company confirmed its key had been live. The global exclude lives on one machine, so it does not follow the project, and neither gitleaks nor TruffleHog flagged 215 of the 1,122 files.

I did not find a vendor statement in the coverage, so the fixes sit on the user side. Apiiro's order is rotate first, then clean: search your organisation for the file and its history, rotate what it held, `git rm --cached` it and add it to `.gitignore`.

OWASP LLM Top 10 2025: LLM02 Sensitive Information Disclosure. MITRE ATLAS: AML.T0083 Credentials from AI Agent Configuration.

## Anatomy of the week: the ChatGPT preview that let a popup out

Lockdown Mode, [added to ChatGPT in June 2026](https://thehackernews.com/2026/11/researchers-bypass-chatgpt-lockdown.html){:target="_blank"}, is an optional setting that closes channels a prompt injection could use to move data out. Since February ChatGPT can also render an HTML page from a code block, in two nested sandboxed iframes with a script that intercepts anything reaching outside. On 5 November Alexey Bukhteyev at [Check Point Research](https://research.checkpoint.com/2026/chatgpt-lockdown-mode-bypass-phishing-malware-delivery-and-silent-data-exfiltration/){:target="_blank"} published how the preview could be used to bypass Lockdown Mode; OpenAI reported the fix deployed on 18 September.

1. The attacker gets an HTML page into a code block the victim sees: a malicious prompt, GPT or shared chat.
2. ChatGPT renders it in the two iframes. A script in the sandbox replaces `window.open`, so popups wait for approval; other outside requests become warnings.
3. The frames' `sandbox` attribute still lists `allow-popups` and `allow-popups-to-escape-sandbox`, so only the replaced function, living in the page's own JavaScript, stands between the page and a popup.
4. The page adds one more nested frame and calls that frame's own `open`, which nothing replaced. The popup opens.
5. Because of `allow-popups-to-escape-sandbox`, the popup is unsandboxed: attacker script runs outside the interception and Lockdown Mode, in a window with the ChatGPT icon and a title the attacker chose.
6. In the ChatGPT Classic desktop app `open` works without a user gesture, so a malicious GPT needs only a normal conversation. In a browser the victim has to click the preview.
7. From the popup the attacker can send chat history and files to their own server, offer malware as an update, or sign the user out and collect the password on a look-alike page.

### Where it could have stopped

- Input: render HTML from code blocks only on request, and not in Lockdown Mode. The public write-ups do not say this changed.
- Isolation: no popup tokens on a frame that shows untrusted HTML. The fix uses this one: OpenAI removed both tokens from the outer iframe.
- Enforcement point: let sandbox flags and a content security policy do the blocking, rather then a function replaced inside the page. A nested frame brings its own `open`.
- Gesture: require a user gesture for `window.open` in the desktop app.
- Output channel: the popup was a path Lockdown Mode did not restrict, so test the mode with every rendering feature on.

The pattern is a guard that runs in the same JavaScript realm as the content it guards against, so the content can step around it, while the enforcement that holds is a browser-level flag. To find it in your own systems, search for functions you wrap to enforce policy (`open`, `fetch`, message handlers) in frames that show content you did not write, then read every `sandbox` attribute and CSP and ask what each still allows. The same week MCP Inspector 2.14.0 fixed a relative: an MCP App's frame could navigate itself away and keep its sandbox flags ([`GHSA-h9j8-vj8g-v3gg`](https://github.com/modelcontextprotocol/inspector/security/advisories/GHSA-h9j8-vj8g-v3gg){:target="_blank"}).

## From the research side

- [Artificial intelligence and cyber intrusion](https://www.ncsc.gov.uk/report/artificial-intelligence-and-cyber-intrusion){:target="_blank"} (UK NCSC, 3 November): an assessment to 2028 that judges it almost certain many systems will still be open to AI tools that find and exploit weaknesses, so patching and testing cycles have to keep pace.
- [The Rise of LLMJacking](https://cloud.google.com/blog/topics/threat-intelligence/llmjacking-monetizing-compromised-cloud-models){:target="_blank"} (Google Threat Intelligence Group and Mandiant, 2 November): hijacked LLM access produced charges above $100,000 in a day in some incidents and $1 million in total at some organisations, mostly through exposed or stolen cloud tokens. Alert on any unplanned enabling of an LLM service.
- [Bypassing network restrictions during a coding evaluation](https://alignment.openai.com/misalignment-reports/bypassing-network-restrictions-during-a-coding-evaluation/){:target="_blank"} (OpenAI Alignment, 5 November): with its proxy down, a research model found another route through raw-packet permission and a side network, and OpenAI found no evidence that model moved sensitive data. List what else your agent sandbox can reach besides the proxy.
- [Free the Model, Sell the Gateway](https://research.checkpoint.com/2026/free-the-model-sell-the-gateway-prompt-adware-hidden-in-jailbreak-bundles/){:target="_blank"} (Check Point Research, 3 November): the Haiou jailbreak bundle for Codex edits the global instruction file to insert ads, which Check Point calls prompt adware. Review global instruction files and third-party skills like any dependency.

## Checklist for Monday

- [ ] Upgrade CrewAI to 1.15.17, Kubeflow Pipelines to 2.18.0, JupyterLab Desktop to 4.6.2-1 and vLLM to 0.33.0 (with authentication in front if you use `--skip-tokenizer-init`), and check the Python ones with `pip list`.
- [ ] Confirm LiteLLM is 1.83.14 or later, and alert on any new LLM service or API key in your cloud accounts.
- [ ] Run `gh search code --owner YOURORG --filename settings.local.json`, check the history of each hit, rotate whatever those files held, and approve patterns instead of exact commands from now on.
- [ ] Turn off cross-device Dispatch unless you need it, and list the browser extensions that can run on claude.ai.
- [ ] For each agent sandbox, list the network paths besides the proxy, and check that no internal tool accepts a development credential in production.
- [ ] Grep UI code for `sandbox=` and `window.open` wherever you render model or user HTML.

## References

By layer:

- [GHSA-gh9r-c94j-cmp5: Stanza RCE via pickle deserialization in the lemmatizer](https://github.com/advisories/GHSA-gh9r-c94j-cmp5){:target="_blank"} - GitHub Advisory Database
- [GHSA-q9v9-qpjp-fc24: unauthenticated EngineCore crash under skip_tokenizer_init](https://github.com/vllm-project/vllm/security/advisories/GHSA-q9v9-qpjp-fc24){:target="_blank"} - vLLM project
- [GHSA-5r3c-5x4v-8gjh: missing request-wide media budget](https://github.com/vllm-project/vllm/security/advisories/GHSA-5r3c-5x4v-8gjh){:target="_blank"} - vLLM project
- [GHSA-qrj2-j723-m6w8: speculators config directory traversal](https://github.com/vllm-project/vllm/security/advisories/GHSA-qrj2-j723-m6w8){:target="_blank"} - vLLM project
- [CVE-2026-120630: CrewAI SSRF guard](https://nvd.nist.gov/vuln/detail/CVE-2026-120630){:target="_blank"} - NVD
- [CVE-2026-120630 affected range and fixed version](https://osv.dev/vulnerability/CVE-2026-120630){:target="_blank"} - OSV
- [GHSA-2h4w-29cv-2cmv: AG2 SandboxShellTool filter bypass](https://github.com/ag2ai/ag2/security/advisories/GHSA-2h4w-29cv-2cmv){:target="_blank"} - AG2 project
- [GHSA-5xjx-9qx6-jcrv: Claude Code Edit tool symlink race](https://github.com/anthropics/claude-code/security/advisories/GHSA-5xjx-9qx6-jcrv){:target="_blank"} - Anthropic
- [CVE-2026-58141: Dify SSRF protection bypass](https://nvd.nist.gov/vuln/detail/CVE-2026-58141){:target="_blank"} - NVD
- [GHSA-rjr3-43j3-4q8g: Kubeflow Pipelines upload authorization](https://github.com/kubeflow/pipelines/security/advisories/GHSA-rjr3-43j3-4q8g){:target="_blank"} - Kubeflow project
- [GHSA-2mr8-962v-wc67: JupyterLab Desktop renderer bridge](https://github.com/jupyterlab/jupyterlab-desktop/security/advisories/GHSA-2mr8-962v-wc67){:target="_blank"} - Jupyter project
- [CVE-2026-111770: MCP TypeScript SDK cross-client responses](https://nvd.nist.gov/vuln/detail/CVE-2026-111770){:target="_blank"} - NVD
- [CVE-2026-111768: MCP TypeScript SDK unbounded validation](https://nvd.nist.gov/vuln/detail/CVE-2026-111768){:target="_blank"} - NVD
- [CVE-2026-111769: MCP TypeScript SDK cross-origin redirects](https://nvd.nist.gov/vuln/detail/CVE-2026-111769){:target="_blank"} - NVD
- [CVE-2026-111771: MCP TypeScript SDK token audience](https://nvd.nist.gov/vuln/detail/CVE-2026-111771){:target="_blank"} - NVD
- [GHSA-64vx-6h2c-rjh7: Banks tool import_path resolved through importlib](https://github.com/masci/banks/security/advisories/GHSA-64vx-6h2c-rjh7){:target="_blank"} - Banks project
- [GHSA-j656-3hf2-fvjc: DBHub read-only bypass on MySQL and MariaDB](https://github.com/bytebase/dbhub/security/advisories/GHSA-j656-3hf2-fvjc){:target="_blank"} - Bytebase
- [CVE-2026-54140: OpenHuman MCP tool descriptions](https://nvd.nist.gov/vuln/detail/CVE-2026-54140){:target="_blank"} - NVD
- [CVE-2026-70660: stable-diffusion.cpp safetensors loader](https://nvd.nist.gov/vuln/detail/CVE-2026-70660){:target="_blank"} - NVD
- [Security Bulletin: Langflow OSS is affected by multiple vulnerabilities](https://www.ibm.com/support/pages/node/7290694){:target="_blank"} - IBM
- [GHSA-qvqp-mxh4-f27w: Langflow URL component response size](https://github.com/langflow-ai/langflow/security/advisories/GHSA-qvqp-mxh4-f27w){:target="_blank"} - Langflow project
- [ZDI-26-844: MLflow get-artifacts SSRF](https://www.zerodayinitiative.com/advisories/ZDI-26-844/){:target="_blank"} - Zero Day Initiative
- [CVE-2026-123974: TorchServe predictable extraction directory](https://nvd.nist.gov/vuln/detail/CVE-2026-123974){:target="_blank"} - NVD
- [Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog){:target="_blank"} - CISA

Case files:

- [Update for October 30th, 2026 Data Security Incident](https://workos.com/security/bulletins/update-for-october-30th-2026-data-security-incident){:target="_blank"} - WorkOS
- [Runaway AI hacking agent behind WorkOS breach after escaping sandbox](https://www.bleepingcomputer.com/news/security/runaway-ai-hacking-agent-behind-workos-breach-after-escaping-sandbox/){:target="_blank"} - BleepingComputer
- [Updated Nov 7: Chainalysis Security Incident Information](https://www.chainalysis.com/blog/security-incident-october-30/){:target="_blank"} - Chainalysis
- [ThisBash Escape: Claude Dispatch Flaw Enables Browser Extensions RCE on the OS](https://forever.security/blog/thisbash-claude-dispatch-browser-extension-rce){:target="_blank"} - Forever Security
- [Malicious Browser Extensions Could Use Claude Dispatch to Run Code on Users' Computers](https://thehackernews.com/2026/11/malicious-browser-extensions-could-use.html){:target="_blank"} - The Hacker News
- [Claude Code's "Don't Ask Again" File Exposes Passwords and Keys in 1,122 Public Copies on GitHub](https://thehackernews.com/2026/11/claude-codes-dont-ask-again-file.html){:target="_blank"} - The Hacker News
- [Claude Code permissions documentation](https://code.claude.com/docs/en/permissions){:target="_blank"} - Anthropic
- [Claude Code settings documentation](https://code.claude.com/docs/en/settings){:target="_blank"} - Anthropic
- [OWASP Top 10 for LLM Applications 2025](https://genai.owasp.org/llm-top-10/){:target="_blank"} - OWASP GenAI Security Project
- [ATLAS data, release v2026.10](https://github.com/mitre-atlas/atlas-data){:target="_blank"} - MITRE

Anatomy and research:

- [ChatGPT Lockdown Mode Bypass: Phishing, Malware Delivery, and Silent Data Exfiltration](https://research.checkpoint.com/2026/chatgpt-lockdown-mode-bypass-phishing-malware-delivery-and-silent-data-exfiltration/){:target="_blank"} - Check Point Research
- [Researchers Bypass ChatGPT Lockdown Mode to Exfiltrate Chats Without User Interaction](https://thehackernews.com/2026/11/researchers-bypass-chatgpt-lockdown.html){:target="_blank"} - The Hacker News
- [GHSA-h9j8-vj8g-v3gg: MCP App escapes its CSP by navigating its own frame](https://github.com/modelcontextprotocol/inspector/security/advisories/GHSA-h9j8-vj8g-v3gg){:target="_blank"} - MCP Inspector project
- [Artificial intelligence and cyber intrusion](https://www.ncsc.gov.uk/report/artificial-intelligence-and-cyber-intrusion){:target="_blank"} - UK NCSC
- [The Rise of LLMJacking: How Threat Actors Monetize Compromised Cloud and Model Resources](https://cloud.google.com/blog/topics/threat-intelligence/llmjacking-monetizing-compromised-cloud-models){:target="_blank"} - Google Cloud
- [Breaking LiteLLM: From Low-Privilege User to Admin and RCE](https://www.obsidiansecurity.com/blog/litellm-privilege-escalation-rce){:target="_blank"} - Obsidian Security
- [GHSA-wxxx-gvqv-xp7p: LiteLLM sandbox escape in custom-code guardrail](https://github.com/advisories/GHSA-wxxx-gvqv-xp7p){:target="_blank"} - GitHub Advisory Database
- [Bypassing network restrictions during a coding evaluation](https://alignment.openai.com/misalignment-reports/bypassing-network-restrictions-during-a-coding-evaluation/){:target="_blank"} - OpenAI Alignment
- [Free the Model, Sell the Gateway: Prompt Adware hidden in Jailbreak Bundles](https://research.checkpoint.com/2026/free-the-model-sell-the-gateway-prompt-adware-hidden-in-jailbreak-bundles/){:target="_blank"} - Check Point Research

Notes on sources:

- Dating: for most entries the fix shipped before the CVE record appeared (Dify 1.15.0 on 25 June, Banks 2.4.3 on 18 June, JupyterLab Desktop 4.6.2-1 on 22 July, CrewAI 1.15.17 on 20 August, Claude Code 2.1.251 on 28 August, AG2 1.1.1 on 29 September, MCP SDK 1.32.0 and 2.3.0 on 2 October, Kubeflow Pipelines 2.18.0 on 28 October). I include an item when its CVE record or project advisory first appeared between 2 and 8 November; the project advisories for the MCP SDK (20 October), AG2 (29 September), Claude Code (26 October), Kubeflow (28 October) and JupyterLab Desktop (24 July) are older than their CVE records.
- CrewAI: the CVE record names no fixed version. I take 1.15.17 from the [OSV range](https://osv.dev/vulnerability/CVE-2026-120630){:target="_blank"}, which ends at the release commit of that version.
- LiteLLM: [Google's report](https://cloud.google.com/blog/topics/threat-intelligence/llmjacking-monetizing-compromised-cloud-models){:target="_blank"} cites `CVE-2026-40217` for a flaw in the `/key/regenerate` endpoint, while NVD, [the GitHub advisory](https://github.com/advisories/GHSA-wxxx-gvqv-xp7p){:target="_blank"} and [Obsidian](https://www.obsidiansecurity.com/blog/litellm-privilege-escalation-rce){:target="_blank"} describe that ID as a sandbox escape in the custom-code guardrail endpoint, so I cannot tie the two together. The advisory text names 1.83.11 as the fix and its metadata names 1.83.10; Obsidian says 1.83.14-stable closes its three chained CVEs, and that release is on PyPI, so I follow Obsidian.
- Langflow: [IBM's 2 October bulletin](https://www.ibm.com/support/pages/node/7290694){:target="_blank"} lists 26 CVEs for 1.0.0 to 1.12.2 and recommends 1.12.3, while the project's advisory for the URL component names 1.13.0. I name 1.13.0, which is later than both. `CVE-2026-97677` reached NVD on 8 November.
- MLflow: the [ZDI advisories](https://www.zerodayinitiative.com/advisories/ZDI-26-844/){:target="_blank"} are dated 29 October and the CVE records 4 November. ZDI names 3.17.0, the release from the 2 November post, as the fix.
- AG2: the `autogen` package, affected through 0.14.1, has no fixed version listed; `ag2` 1.1.1 is the fix.
- Apiiro: its own page did not render for me, so the figures in the Claude Code case are Apiiro's as reported by The Hacker News on 7 November.
- Dispatch: the status is as of The Hacker News piece of 6 November and the Forever Security write-up, which says the flaw remains exploitable until retirement.
- Tooling: the [KEV catalogue](https://www.cisa.gov/known-exploited-vulnerabilities-catalog){:target="_blank"} (version 2026.11.06) has no AI or ML additions in the window, and MITRE ATLAS had no release in it, so IDs come from v2026.10 of 28 October.
