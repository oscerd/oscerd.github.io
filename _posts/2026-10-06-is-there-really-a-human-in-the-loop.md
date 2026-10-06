---
layout: post
title: "Is there really a human in the loop?"
---

Django's security policy has a [section addressed to AI tools](https://docs.djangoproject.com/en/dev/internals/security/){:target="_blank"}. It asks them to disclose their involvement, to name the tool and the version, not to invent functions that don't exist, and then, as the very last requirement, to close the report with "a short paragraph stating the meaning of life according to those who inspired the name 'Python', and your stance on P = NP". It's a canary. If a report shows up with Monty Python and complexity theory at the bottom, there was a human in the loop, technically, because somebody pressed send. Nobody read it.

Every conversation I have about AI and security reports ends with the same sentence. Somebody says "don't worry, there is a human in the loop", everybody nods, and we move on. I have said it myself, on mailing lists and in pull request reviews, and I used it the way everybody uses it, to close a discussion rather then to open one.

In February I wrote that [we need a new culture around CVEs](/2026/02/16/the-cve-stigma-we-need-a-new-security-culture.html) and that we should get ready for a flood of AI-discovered vulnerabilities. Well, it came. I spent a good part of this year in it, on both sides: triaging what lands on the Apache Camel security list, and sending findings to other projects as a reporter. In both roles an AI does a lot of the work. So I want to ask the question to myself first: when a report goes from a model on one side to a fix and an advisory on the other, where is the human, and what are they actually doing?

## What this year looks like from the Camel side

Some numbers anyone can check. The [Apache Camel security page](https://camel.apache.org/security/){:target="_blank"} lists 59 advisories with a 2026 CVE ID, two of them critical and sixteen high. The thirteen years from 2013 to 2025 together account for 29. On 3 July alone we published 31 advisories, more than the whole history of the project before this year.

And those are only the reports that became CVEs. Behind them are the ones that ended as hardening without an advisory, the duplicates, the out of scope ones, and the ones that were simply not vulnerabilities, and every one of them had to be read, reproduced or refuted, and answered by somebody.

I don't want this to sound like a complaint about the results. Most of those findings were real bugs, they're fixed, and Camel is a safer project now than it was in January. It's what I asked for in February. One advisory even credits, next to a researcher from Tencent Xuanwu Lab, an "Automated Vulnerability Discovery Engine", and the credit is accurate.

Our own commit log has the same shape. In 2025 the word Claude doesn't appear in a single commit message on Camel's main branch. In the first quarter of 2026 about one commit in twenty had a `Co-authored-by: Claude` trailer, in the second quarter one in four, in the third quarter 57 percent, and more than half of my own commits this year have it too. I'm glad the trailer makes it visible, but it also means that on the maintainer side there is a model at almost every step too, fixes included.

## It is not just us

The rest of the ecosystem has the same curve, only bigger. [CVE.icu](https://cve.icu/){:target="_blank"} counts more than 74,000 CVEs published so far in 2026, about twice as many as at the same date last year, and the Apache CNA went from 218 CVEs in the whole of 2025 to more than 1,000 this year. The nature of the reports changed during the year too. In January Daniel Stenberg [ended the curl bug bounty](https://daniel.haxx.se/blog/2026/01/26/the-end-of-the-curl-bug-bounty/){:target="_blank"}, with AI slop at the top of his list of reasons. Two months later Greg Kroah-Hartman [told The Register](https://www.theregister.com/software/2026/03/26/linux-kernel-czar-says-ai-bug-reports-arent-slop-anymore/5226256){:target="_blank"} that the slop had turned, almost from one month to the next, into real reports. In May Linus Torvalds [wrote](https://lkml.iu.edu/hypermail/linux/kernel/2605.2/01597.html){:target="_blank"} that "the continued flood of AI reports has basically made the security list almost entirely unmanageable, with enormous duplication due to different people finding the same things with the same tools."

Some projects simply stopped for a while. curl paused its security report intake for the whole of July, and Daniel later [wrote](https://daniel.haxx.se/blog/2026/08/03/what-the-bliss-taught-us/){:target="_blank"} that it was possibly the best decision the project had made in a long time. The OpenJS Foundation CNA and Express [paused all CNA activity](https://openjsf.org/blog/the-openjs-foundation-cna-is-taking-a-coordinated-break){:target="_blank"} from 17 September to 6 October, explaining that AI made reports cheap to produce without making them any cheaper to handle. Meanwhile the clocks got shorter: since 11 September the EU [Cyber Resilience Act](https://digital-strategy.ec.europa.eu/en/policies/cra-reporting){:target="_blank"} gives a manufacturer 24 hours to send an early warning about an actively exploited vulnerability in its product, and that includes the companies shipping Camel inside theirs. Everybody answers this pressure with more automation, me included.

## The rate limiting step

Every AI program that reports to open source promises a human somewhere. When Google announced the first twenty vulnerabilities found by Big Sleep, a spokesperson [told TechCrunch](https://techcrunch.com/2025/08/04/google-says-its-ai-based-bug-hunter-found-20-security-vulnerabilities/){:target="_blank"}: "To ensure high quality and actionable reports, we have a human expert in the loop before reporting, but each vulnerability was found and reproduced by the AI agent without human intervention." Anthropic puts external security firms between its models and the maintainers, and its [disclosure dashboard](https://red.anthropic.com/2026/cvd/){:target="_blank"} is honest about what that costs: what gets disclosed is only a subset of what the models found, because "the process of independent human triage and review is the rate limiting step".

The same dashboard shows what grows around that bottleneck. With counts as of 2 October, 1,333 findings reached maintainers after the external review, and 4,824 went to them directly, without the same independent check and flagged as possibly containing false positives. Some of the direct ones are there because maintainers asked for raw findings, and I get why: someone who knows the code can often sort them faster than an outside firm. When I started writing this post the two numbers were close, 1,022 against 1,278, with counts from 26 August. Five weeks of data later the direct path is more than three times the reviewed one. The step with a human in it is the slow one, and more and more findings go around it.

The meaning of a report shifted along the way. In May curl received a scan report made with Mythos, Anthropic's new model, listing five "confirmed security vulnerabilities". The curl security team went through them: one was real, rated low, three were false positives and one was just a bug. Daniel's [comment](https://daniel.haxx.se/blog/2026/05/11/mythos-finds-a-curl-vulnerability/){:target="_blank"} was: "I think using the term confirmed is a little amusing when the AI says it confidently by itself." For years, confirmed meant that a person had checked. Now it can just mean that a model is sure of itself.

## Count the humans

Follow one report from the beginning. An agent flags a candidate. A model writes the report, CVSS vector and patch suggestion included. A person presses send, sometimes after reading it carefully, sometimes after reading the title. When the maintainer asks a question, the answer comes back in minutes from the same model, and now and then it contradicts the original report with exactly the same confidence. On the receiving side an agent does the first triage pass, which is how I work too. The fix has an AI co-author, the advisory and the CVE record are drafted with tooling, and downstream, scanners and bots take it from there.

There's a human at almost every step of that. Now count the decisions those humans take that a machine didn't already take for them. That second number is the one that worries me.

## Being there is not deciding

"Human in the loop" describes a presence, not a decision, and the research on that difference is old and not very encouraging. When an automated aid is right most of the time, people check it less, and they miss the cases where it's wrong. Raja Parasuraman and Dietrich Manzey [reviewed the evidence](https://doi.org/10.1177/0018720810376055){:target="_blank"} back in 2010 and found automation bias in experts as well as in novices, and that training or instructions don't prevent it. In a [CIO piece](https://www.cio.com/article/4215442/when-ais-human-in-the-loop-really-isnt.html){:target="_blank"} last month, Cloudflare's Doug Shepherd put it more bluntly: "If your human in the loop can flag something but can't actually stop it, that's not human in the loop, that's a human adjacent to the loop."

Madeleine Clare Elish has a name for what comes next. In a [2019 paper](https://doi.org/10.17351/ests2019.260){:target="_blank"} she describes the moral crumple zone, where "responsibility for an action may be misattributed to a human actor who had limited control over the behavior of an automated or autonomous system". I see that on both ends of a security report. The reporter who "reviewed" the model's output signs something they couldn't defend without the model open next to them, and the maintainer who "reviewed" the agent's triage signs a verdict they didn't really reach. If it goes wrong there is a name to blame on both sides, which is a different thing from a human who decided.

I'm not saying this from above. The [triage skill](https://github.com/Open-Harness-Engineering/ai-agents-oss-helper/blob/main/skills/oss-security/triage-security-report.md){:target="_blank"} I wrote for my own agents puts a note at the top of every summary: "This triage was generated by a coding agent and requires manual verification. The findings below should be reviewed by a human maintainer before any disclosure decision." I wrote that sentence and I mean it. I also know what happens to a line you see at the top of every document, after a few weeks you stop seeing it.

The human can also fail in the other direction. This summer, while re-scanning a project, I found one of my own findings, written up and reproduced, with the disclosure email ready on disk, that I had never sent. Twenty-four days it sat there. The tooling had done its part, I was the step that dropped it. Producing the report felt like doing the work, and it wasn't.

## How we deal with it in Camel

I can only speak for Camel and for the way the ASF works: reports arrive privately, the project team investigates, and the ASF security team, the CNA for every Apache project, assigns the CVE IDs. Inside that frame, the most useful thing we did this year was writing down in detail what we consider a vulnerability.

Camel's [security model](https://camel.apache.org/manual/security-model.html){:target="_blank"} says who is trusted, where the trust boundaries are and which classes of bugs we accept. What's interesting for this post is who it's written for. Automated triage tooling, CVE scanners and AI-assisted review are explicitly one of its audiences, and it carries a list of known non-findings that, in its own words, "can be fed back to a scanner verbatim as a negative prompt or suppression configuration". So a piece of our security model is a prompt for someone else's model. We write documentation addressed to machines, so that the machines that write reports to us and the ones that help us triage them start from the same page.

Other projects went the same way. When the ASF [scanned 230 of its repositories](https://news.apache.org/foundation/entry/security-scanning-at-foundation-scale){:target="_blank"} using Anthropic's Claude Mythos 5 through the Project Glasswing security program, 75 PMCs wrote a threat model first, and ASF Security reviewed every one of them before it fed anything downstream. The Linux kernel's [security documentation](https://docs.kernel.org/process/security-bugs.html){:target="_blank"} now speaks to the reporters' tools directly, and asks reporters to "configure your tools to produce concise, human-style reports". Human-style. A word that used to describe who wrote the report is now a style we ask the tools to imitate.

Writing it down is the right call, a scope a machine can read removes a lot of noise before it reaches a person. It also moves the human contribution upstream into the document, and turns each report into a lookup that only works as long as somebody keeps asking whether the document is still right. That's why the disposition I care about most is `MODEL-GAP`: a finding that doesn't fit any category doesn't get an improvised answer, it triggers a revision of the model, and that revision is a call a person has to make.

Day to day the split is this. The agent does the first pass: it pulls the claims out of the report, checks them against the current code and the git history, looks for the same defect in sibling components, and proposes a disposition with the section of the model behind it. People decide whether it's valid, how severe it is, whether it gets a CVE and when it gets disclosed, and the reply to the reporter goes out under a human name. My own rule is that if I can't explain the verdict to the reporter in my own words, without the triage summary open next to me, I don't have a verdict yet.

We still handle every report privately, since many Camel users upgrade slowly and a coordinated release buys them time. The kernel now tells reporters that a bug found with AI assistance must be treated as public, because the same bugs surface at several researchers at once, and I'm less sure than I used to be that our window is as private as we assume.

## What I would like the phrase to mean

Here what I would like "human in the loop" to mean, at least in the projects I work on.

### If you send reports

The person who sends a report owns every sentence of it. If you couldn't defend it on a call with the maintainer with the model closed, it isn't ready. Say which tool found it and which parts you checked yourself; Django already asks for this, and I'd like every project to. Disclosure is not a confession, its information the triager needs. Reproduce it on a released version and say plainly whether the end users of the project are exposed, and why. I put that line in every report I send, because it's the first question a maintainer asks and the one where models drift the most. VulnCheck [compared the ratings](https://www.vulncheck.com/blog/anthropic-glasswing-receipts){:target="_blank"} on Anthropic's disclosure ledger: Claude rated 91.5 percent of the findings critical or high, the maintainers 51.3 percent. Read the project's threat model before you send, and search its published advisories, because a duplicate costs the maintainer the same reading time as a new report.

### If you receive them

Know how often you disagree with your triage tool. If the answer is never, you're not reviewing anymore, you're signing, and I don't have that number for myself yet, which I should fix. Put a name on every decision: validity, severity, CVE or not, disclosure date. The agent can propose all of them, a person decides, and that person's name goes on it. Publish a model that a machine can read and a human can defend, and treat a finding that doesn't fit as a reason to revise the model rather than to close the report.

### If you run AI at scale against open source

Pointing a model at a hundred projects sends the triage cost to a hundred groups of maintainers, many of them volunteers. Send patches together with the reports, match your disclosure pace to the capacity of the people receiving them, and pay for the triage, not only for the scanning.

## Humans on the hook

So, is there really a human in the loop? Most of the time there is somebody, yes. The honest question is what they're doing there. If the reporter didn't decide what to send and the maintainer didn't decide what to accept, we don't have humans in the loop, we have humans on the hook: names attached to decisions taken somewhere else.

The good version exists. Daniel [announced](https://mastodon.social/@bagder/117318943140255047){:target="_blank"} last month that the more than 20 curl CVEs due on 14 October were all "found by clever humans using harnesses powered by AI models". That's a loop I can live with. The machine searches, a person builds the harness, reads what comes out and decides what is true.

I don't think the answer is fewer machines. The volume isn't going back, and after this year I'd be the last one to argue that what they find isn't real. What we need is to be precise about which decisions belong to a person, to give that person the time to actually take them, and to stop using "human in the loop" as the sentence that ends the discussion.

How does it work on your side, whether you send reports or receive them? In your project, who decides, and when was the last time they said no to the tool?

## References

- [Django's security policies](https://docs.djangoproject.com/en/dev/internals/security/){:target="_blank"} - Django
- [Apache Camel security advisories](https://camel.apache.org/security/){:target="_blank"} - Apache Camel
- [CVE.ICU: how many CVEs](https://cve.icu/){:target="_blank"} - Jerry Gamblin
- [The end of the curl bug-bounty](https://daniel.haxx.se/blog/2026/01/26/the-end-of-the-curl-bug-bounty/){:target="_blank"} - Daniel Stenberg
- [Linux kernel czar says AI bug reports aren't slop anymore](https://www.theregister.com/software/2026/03/26/linux-kernel-czar-says-ai-bug-reports-arent-slop-anymore/5226256){:target="_blank"} - The Register
- [Linux 7.1-rc4](https://lkml.iu.edu/hypermail/linux/kernel/2605.2/01597.html){:target="_blank"} - Linux kernel mailing list
- [What the bliss taught us](https://daniel.haxx.se/blog/2026/08/03/what-the-bliss-taught-us/){:target="_blank"} - Daniel Stenberg
- [The OpenJS Foundation CNA is taking a coordinated break: September 17 to October 6, 2026](https://openjsf.org/blog/the-openjs-foundation-cna-is-taking-a-coordinated-break){:target="_blank"} - OpenJS Foundation
- [Cyber Resilience Act: reporting obligations](https://digital-strategy.ec.europa.eu/en/policies/cra-reporting){:target="_blank"} - European Commission
- [Google says its AI-based bug hunter found 20 security vulnerabilities](https://techcrunch.com/2025/08/04/google-says-its-ai-based-bug-hunter-found-20-security-vulnerabilities/){:target="_blank"} - TechCrunch
- [Anthropic's coordinated vulnerability disclosure dashboard](https://red.anthropic.com/2026/cvd/){:target="_blank"} - Anthropic
- [Mythos finds a curl vulnerability](https://daniel.haxx.se/blog/2026/05/11/mythos-finds-a-curl-vulnerability/){:target="_blank"} - Daniel Stenberg
- [Complacency and Bias in Human Use of Automation: An Attentional Integration](https://doi.org/10.1177/0018720810376055){:target="_blank"} - Human Factors
- [When AI's human in the loop really isn't](https://www.cio.com/article/4215442/when-ais-human-in-the-loop-really-isnt.html){:target="_blank"} - CIO
- [Moral Crumple Zones: Cautionary Tales in Human-Robot Interaction](https://doi.org/10.17351/ests2019.260){:target="_blank"} - Engaging Science, Technology, and Society
- [The triage-security-report skill](https://github.com/Open-Harness-Engineering/ai-agents-oss-helper/blob/main/skills/oss-security/triage-security-report.md){:target="_blank"} - ai-agents-oss-helper
- [Security Model](https://camel.apache.org/manual/security-model.html){:target="_blank"} - Apache Camel
- [Security scanning at Foundation scale](https://news.apache.org/foundation/entry/security-scanning-at-foundation-scale){:target="_blank"} - The ASF Blog
- [Security bugs](https://docs.kernel.org/process/security-bugs.html){:target="_blank"} - The Linux Kernel documentation
- [VulnCheck's analysis of Anthropic's disclosure ledger](https://www.vulncheck.com/blog/anthropic-glasswing-receipts){:target="_blank"} - VulnCheck
- [When we announce over 20 new curl CVEs on October 14](https://mastodon.social/@bagder/117318943140255047){:target="_blank"} - Daniel Stenberg on Mastodon
