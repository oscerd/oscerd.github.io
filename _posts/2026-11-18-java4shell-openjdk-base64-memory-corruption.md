---
layout: post
title: "Java4Shell: memory corruption in OpenJDK's Base64 decoder"
---

Every so often a vulnerability is interesting less for what it lets you do, which here is unauthenticated remote code execution, than for where it lives. `CVE-2026-114938`, nicknamed Java4Shell by the team at Calif who found it, is a memory-corruption bug in `java.util.Base64`. Memory corruption, in a managed language, in one of the first calls almost every Java service makes before it has authenticated anyone.

This is the deep dive I promised in [this week's roundup](/2026/11/18/java-vulnerabilities-of-the-week-12-18-november-2026.html){:target="_blank"}, pulled out on its own because it earns it. I am taking the mechanism from Calif's write-up, which publishes the vulnerable pseudocode and a self-contained proof of concept; the OpenJDK advisory confirms the component and the score but, as the project usually does, no detail.

## Why Java had a memory bug at all

Java's memory safety is a property of bytecode running over the managed heap, and a lot of what runs underneath that bytecode is not bytecode. HotSpot's C2 compiler watches which methods run hot and replaces some of them with intrinsics: routines the JVM substitutes for the method body, the fastest of which are hand-written assembly that uses CPU-specific instructions to move many bytes at once. `Base64.Decoder.decode` is one of them. On x86-64 processors with AVX-512 VBMI, decoding runs through an intrinsic built on the `VMOVDQU8` instruction, which writes a run of bytes to memory under a 64-bit mask: a set bit means the byte at that position gets written.

## The bug

The intrinsic handles the input in 256-byte then 64-byte blocks, and for a tail shorter than 64 bytes it does a single masked write through `evmovdqub`. The mask is computed from the input and output lengths and then adjusted for the `=` padding. In pseudocode, from Calif:

```
output_len := (input_length / 4) * 3
if input[input_length - 1] == '=':
    output_len -= 1
    if input[input_length - 2] == '=':
        output_len -= 1
output_mask := bzhiq(-1, output_len)
```

`bzhiq(-1, n)` clears every bit above position `n`, so for a small `output_len` you get a mask with just the low bits set, which is exactly the bytes you mean to write. The trouble is the padding subtraction. For a tail that decodes to nothing, such as `=`, `==`, `X=` or `XY=`, `output_len` comes out as zero, and then the padding branch decrements it, so it underflows to -1 or -2. Read as an unsigned value that is enormous, well past 64, and `bzhiq` leaves a value that large untouched, so `output_mask` comes back as all ones. Every one of the 64 bytes gets written.

Now the sizes. For an input of 64 `A` characters followed by one `=`, 65 bytes, the decoder sizes its output buffer as `ceil(65 / 4) * 3 - 1`, which is 50 bytes. The masked write puts 64 bytes starting at offset 48, so it runs from offset 48 to 111, and 62 of those bytes, offsets 50 through 111, are past the end of the buffer. The buffer is on the Java heap, so those 62 bytes land on top of whatever object sat next to it. The decoder then notices the input was malformed and throws `IllegalArgumentException`, which is all the caller sees, but by then the neighbour is already overwritten. On OpenJDK 17 the trigger is 128 `A` characters and a `=`, because the threshold to reach that code path is higher.

## Why it is pre-auth, and nearly everywhere

Base64 is how you carry binary data through text, so decoding it is one of the first things a server does with a request: the `Authorization: Basic` header, each segment of a JWT, a SAML assertion, a session cookie. That decoding happens before anything has established who the caller is. Calif report verified pre-auth RCE, in default configurations, in Keycloak, Atlassian Jira and Confluence, JFrog Artifactory, Jenkins and PingFederate, and post-auth RCE in Kafka, Solr, Elasticsearch, Gerrit and Spinnaker. There is even a client-side variant, where a Java program that only opens a TLS connection to a hostile server can be hit. The trigger is a single `AAAA...A=` string.

## From a fixed write to code execution

Turning the overwrite into RCE is the hard part, and its why the two published scores disagree: Calif put it at 9.2 on CVSS 4.0, the OpenJDK advisory at 8.1 on CVSS 3.1 with high attack complexity. The write is almost entirely fixed (the first byte is controllable, the next four bits, and the rest is constant), so Calif reach for the classic move of overwriting an adjacent buffer's length field. A buffer whose length has been corrupted keeps accepting bytes, which upgrades the fixed write into a bounded, chosen-value write over its neighbour; corrupt a second array's length and read it back through the response and you have a heap-disclosure primitive, a Heartbleed for the Java heap that leaks keys, tokens and other data. From there they repair the objects they damaged to keep the JVM running and escalate into the code cache for execution. None of it is point and shoot: every target needs bespoke heap-grooming profile, and the remote version is sensitive to network timing, to the point that Calif rented a second machine in the same region as the target. That difficulty is real, but it is the kind that erodes once a technique is published.

## What to do

The fix is the November 2026 Critical Patch Update for your JDK. The bug affects OpenJDK 17.0.6 and later, up to that update, on x86-64 CPUs with AVX-512 VBMI (Intel Ice Lake and newer Xeons, AMD Zen 4 and newer); 8 and 11 never had the affected intrinsic. On Linux you can check the CPU with:

```
grep -m1 '^flags' /proc/cpuinfo | tr ' ' '\n' | grep -E '^(avx512f|avx512bw|avx512vbmi|bmi2)$'
```

All four flags present means you are in range. If you cannot apply the update today, start the JVM with `-XX:-UseBASE64Intrinsics` to drop the hardware-accelerated path, or `-XX:UseAVX=2` on x86-64 to turn AVX-512 off; either one takes you off the vulnerable code. Calif publish a bounded, self-contained trigger that keeps every write inside a 4096-byte array and prints whether your runtime is affected, so you can tell without putting a live service at risk.

## The lesson

The thing worth keeping is the boundary. "Java is memory-safe" is true of your code and of the managed heap, and it stops being the whole story the moment the JIT swaps in an intrinsic, because from there you are running assembly that is only as safe as the assembly. You will not find this by reading your own source, which is a plain `Base64.getDecoder().decode(s)` with nothing wrong with it, and that is exactly why the fix is a runtime upgrade rather then a dependency bump. Calif make a wider point worth repeating as theirs: the AVX-512 Base64 path exists for the few applications that decode very large volumes, and every other application ships and runs it anyway. For a shared runtime, the useful question is less "what can we add" than "what can we turn off", and the two mitigation flags above are a version of that you can run this afternoon.

## References

- [Java4Shell: Write once, RCE everywhere](https://calif.io/java4shell/){:target="_blank"} - Calif
- [oss-security: CVE-2026-114938 OpenJDK pre-auth RCE via memory-corruption in Base64 decoder](https://www.openwall.com/lists/oss-security/2026/11/17/31){:target="_blank"} - Openwall oss-security
- [OpenJDK Vulnerability Advisory: 2026/11/17](https://openjdk.org/groups/vulnerability/advisories/2026-11-17){:target="_blank"} - OpenJDK Vulnerability Group
- [Oracle Critical Patch Update Advisory, November 2026](https://www.oracle.com/security-alerts/cspunov2026.html){:target="_blank"} - Oracle
- [CVE-2026-114938](https://www.cve.org/CVERecord?id=CVE-2026-114938){:target="_blank"} - CVE.org
- [Java vulnerabilities of the week: 12 to 18 November 2026](/2026/11/18/java-vulnerabilities-of-the-week-12-18-november-2026.html){:target="_blank"} - oscerd.github.io
