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

Turning the overwrite into RCE is the hard part, and its why the two published scores disagree: Calif put it at 9.2 on CVSS 4.0, the OpenJDK advisory at 8.1 on CVSS 3.1 with high attack complexity. The write you get is almost entirely fixed. Of the 64 bytes, the first is fully attacker-controlled, the next carries four controlled bits, and the remaining 62 are constant, the high ones always `0x9a`. You do not get to write an address of your choosing; what you control is mostly where the write lands, not what it contains, and that is the gap between a crash and a working exploit.

The way across that gap, per Calif, is the length-field overwrite, an old move from browser exploitation. You arrange for the decoder's output buffer to sit just before an array whose length field falls inside the spill, and the fixed write clobbers that length. An array that now believes it is enormous keeps reading and writing far past its real end, which turns the single fixed spill into a bounded, chosen-value linear write; Calif call that array the writer. Do the same to a second array whose contents come back in a response and you have a read, the reader, a Heartbleed for the Java heap that leaks keys, tokens and whatever else is resident, before a single instruction of attacker code has run. They name that stage JavaBleed. From the writer and the reader they repair the objects the spill damaged so the JVM does not crash, then grow the bounded write into the HotSpot code cache and reach execution.

None of this is point and shoot. There is no universal payload: every target needs its own grooming profile, because the layout depends on the application's allocation pattern, on how the G1 collector hands out regions, and on the thread-local buffer each request writes into. The remote version is sensitive to round-trip time and jitter, enough that Calif rented a second machine in the same region as the target to steady it. That difficulty is what the 8.1 is measuring, and it is real. It is also the kind that erodes: the first reliable chain is expensive, and once the technique is written down the next one is cheaper.

## Where it bites, and where it stalls

Who is actually exposed comes down to whether an attacker can get the malformed string to the decoder and groom the heap around it, and a few ordinary things move that line. A proxy or server that buffers the whole request body before passing it on takes away the timing some grooming profiles lean on, and nginx buffers request bodies by default, so a good deal of traffic is shaped against the attacker without anyone deciding it should be.

That is not a control to rely on. Calif point out that Elasticsearch works from a single complete request, so buffering does not disturb its chain, and that once a connection has been upgraded to a WebSocket the later frames tunnel straight through and body buffering stops applying. Any path that streams, or that decodes Base64 from somewhere other than a buffered body, a header, a token, a form field, is still in range.

So the honest reading of the risk has three steps, not one. Triggering the corruption is trivial, the `AAAA...A=` string does it from any decode call. Turning it into a crash or a heap leak is harder but well within reach. A reliable, repeatable RCE against a named product is real engineering, done per target. That is why the first thing you are likely to see in the wild is scanners and leak proofs rather than mass exploitation, and its also why waiting is the wrong read: JavaBleed on its own, a quiet read of heap memory, already exposes the keys and tokens most Java services keep resident.

## What to do

The fix is the November 2026 Critical Patch Update for your JDK. The bug affects OpenJDK 17.0.6 and later, up to that update, on x86-64 CPUs with AVX-512 VBMI (Intel Ice Lake and newer Xeons, AMD Zen 4 and newer); 8 and 11 never had the affected intrinsic. On Linux you can check the CPU with:

```
grep -m1 '^flags' /proc/cpuinfo | tr ' ' '\n' | grep -E '^(avx512f|avx512bw|avx512vbmi|bmi2)$'
```

All four flags present means you are in range. If you cannot apply the update today, start the JVM with `-XX:-UseBASE64Intrinsics` to drop the hardware-accelerated path, or `-XX:UseAVX=2` on x86-64 to turn AVX-512 off; either one takes you off the vulnerable code.

Calif publish a bounded, self-contained trigger that keeps every write inside a 4096-byte array, so you can tell whether a runtime is affected without putting a live service at risk. The shape of it is short enough to show. Warm the decoder so the JIT installs the intrinsic, then decode the malformed input into an oversized array and look at what moved past offset 50:

```java
Base64.Decoder d = Base64.getDecoder();
byte[] good = "A".repeat(64).getBytes();
byte[] bad  = ("A".repeat(64) + "=").getBytes();
byte[] out  = new byte[4096];
for (int i = 0; i < 200_000; i++) d.decode(good, out); // reach the C2 intrinsic
java.util.Arrays.fill(out, (byte) 0x5a);
try { d.decode(bad, out); } catch (IllegalArgumentException expected) {}
// on an affected JDK, 62 bytes at offsets 50..111 are no longer 0x5a
```

Every write stays inside the 4096-byte array, so it is safe to run; on a patched JDK nothing past offset 50 changes. The full version, with the counting and the runtime print, is in Calif's write-up.

## The lesson

The thing worth keeping is the boundary. "Java is memory-safe" is true of your code and of the managed heap, and it stops being the whole story the moment the JIT swaps in an intrinsic, because from there you are running assembly that is only as safe as the assembly. You will not find this by reading your own source, which is a plain `Base64.getDecoder().decode(s)` with nothing wrong with it, and that is exactly why the fix is a runtime upgrade rather then a dependency bump. Calif make a wider point worth repeating as theirs: the AVX-512 Base64 path exists for the few applications that decode very large volumes, and every other application ships and runs it anyway. For a shared runtime, the useful question is less "what can we add" than "what can we turn off", and the two mitigation flags above are a version of that you can run this afternoon.

One last thing earns a mention, because it is the other half of the story. Calif found this by pointing an AI agent at the JIT, the intrinsics and the crypto code, and the find took around an hour of prompting and discarding dead ends. That is a bug in one of the most-read codebases in the industry, in CPU-specific assembly almost nobody reads for pleasure, surfaced cheaply by something that does not get bored halfway down the function. Whatever you make of the branding around these disclosures, the signal is hard to miss: the low-level, under-reviewed corners of mature projects are about to get a great deal more attention, and not only from the people who patch them.

## References

- [Java4Shell: Write once, RCE everywhere](https://calif.io/java4shell/){:target="_blank"} - Calif
- [oss-security: CVE-2026-114938 OpenJDK pre-auth RCE via memory-corruption in Base64 decoder](https://www.openwall.com/lists/oss-security/2026/11/17/31){:target="_blank"} - Openwall oss-security
- [OpenJDK Vulnerability Advisory: 2026/11/17](https://openjdk.org/groups/vulnerability/advisories/2026-11-17){:target="_blank"} - OpenJDK Vulnerability Group
- [Oracle Critical Patch Update Advisory, November 2026](https://www.oracle.com/security-alerts/cspunov2026.html){:target="_blank"} - Oracle
- [CVE-2026-114938](https://www.cve.org/CVERecord?id=CVE-2026-114938){:target="_blank"} - CVE.org
- [Java vulnerabilities of the week: 12 to 18 November 2026](/2026/11/18/java-vulnerabilities-of-the-week-12-18-november-2026.html){:target="_blank"} - oscerd.github.io
