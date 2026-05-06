---
layout: post
title: "Apache Camel is preparing for Post-Quantum Cryptography"
---

Post-Quantum Cryptography has moved from research papers to production standards. NIST finalized [FIPS 203 (ML-KEM)](https://csrc.nist.gov/pubs/fips/203){:target="_blank"}, [FIPS 204 (ML-DSA)](https://csrc.nist.gov/pubs/fips/204){:target="_blank"}, and [FIPS 205 (SLH-DSA)](https://csrc.nist.gov/pubs/fips/205){:target="_blank"} in August 2024, and the Java ecosystem is catching up fast. JDK 24 shipped with native ML-KEM support via the `javax.crypto.KEM` API ([JEP 496](https://openjdk.org/jeps/496){:target="_blank"}), and JDK 27 will bring PQC directly into TLS with [JEP 527](https://openjdk.org/jeps/527){:target="_blank"}.

As Chair of the Apache Camel PMC, I wanted to start preparing Camel for this transition early. Over the past weeks I have been working on the core SSL/TLS infrastructure and building proof-of-concept examples that show Camel can negotiate PQC TLS handshakes across three different JDK versions.

In this post I will walk through what we changed in Camel's core, why it matters, and how you can try it yourself.

## Why PQC matters for integration

If you are building enterprise integrations, chances are you are moving sensitive data between systems over TLS. The "harvest now, decrypt later" threat is real: adversaries can record encrypted traffic today and decrypt it once a sufficiently powerful quantum computer becomes available. For regulated industries, the migration deadline is approaching fast. [NSA's CNSA 2.0](https://media.defense.gov/2022/Sep/07/2003071834/-1/-1/0/CSA_CNSA_2.0_ALGORITHMS_.PDF){:target="_blank"} guidance requires PQC for TLS in national security systems by 2033.

For Apache Camel users, this means the framework they use to wire systems together needs to support PQC at the transport layer. That is the direction we are heading, though there is still a good amount of work ahead.

## What changed in Camel core

The PQC work started in Camel 4.19.0 and continued into 4.20. It focuses on making PQC TLS configuration as simple as adding a few properties.

### Named Groups and Signature Schemes

TLS 1.3 uses [named groups](https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-8){:target="_blank"} to negotiate key exchange algorithms. PQC key exchange requires new named groups like `X25519MLKEM768`, a hybrid that combines classical X25519 with the post-quantum ML-KEM-768 algorithm. Both run together, so security is maintained even if one is broken.

Before these changes, there was no way to configure named groups through Camel's SSL configuration. We added `namedGroups` and `signatureSchemes` to both `SSLContextParameters` and the `camel.ssl.*` configuration properties:

```properties
camel.ssl.namedGroups=X25519MLKEM768,x25519
camel.ssl.signatureSchemes=ML-DSA,ECDSA,RSA
```

We also added include/exclude filters for fine-grained control:

```properties
camel.ssl.namedGroupsInclude=X25519MLKEM768,x25519
camel.ssl.namedGroupsExclude=ffdhe2048
```

### Auto-configuration on JDK 25+

On JDK 25 and later (where `X25519MLKEM768` is available in the JVM), Camel will automatically reorder the named groups to prefer PQC key exchange. If you have not explicitly configured named groups, Camel sets the ordering to:

```
X25519MLKEM768, x25519, secp256r1, secp384r1, [JVM defaults]
```

No configuration required. If your JVM supports it, Camel prefers PQC. You will see a log line confirming it:

```
INFO  SSLContextParameters - Auto-configured PQC named groups: [X25519MLKEM768, x25519, secp256r1, secp384r1, ...]
```

This only kicks in when you have not set `namedGroups` or `namedGroupsFilter` explicitly, so it will not override your choices.

### Self-signed certificates and provider selection

For development and testing, we added `camel.ssl.selfSigned=true`, which generates a self-signed certificate at startup so you can enable HTTPS without providing a keystore. Combined with `camel.ssl.trustAllCertificates=true`, this gives you a working TLS setup with zero external files.

We also added `camel.ssl.provider` so you can select a specific JSSE provider. This is what makes PQC work on JDK 21 and 24 through BouncyCastle:

```properties
camel.ssl.provider=BCJSSE
```

### Tracking issues

- [CAMEL-23154](https://issues.apache.org/jira/browse/CAMEL-23154){:target="_blank"} - Add PQC named groups support to SSLContextParameters
- [CAMEL-23158](https://issues.apache.org/jira/browse/CAMEL-23158){:target="_blank"} - Add PQC named groups and signature schemes to SSL configuration properties
- [CAMEL-23159](https://issues.apache.org/jira/browse/CAMEL-23159){:target="_blank"} - Add signatureSchemes to SSLContextParameters

## Three proof-of-concept examples

To validate that the approach works in practice, I built three self-contained examples that each perform a real PQC TLS 1.3 handshake using `X25519MLKEM768`. The examples are available in the [camel-pqc-tls](https://github.com/oscerd/camel-pqc-tls){:target="_blank"} repository.

Each example creates an SSLServerSocket and an SSLSocket, performs a TLS 1.3 handshake with PQC key exchange, and verifies the result. No external servers, no mock objects. A real handshake that either succeeds or fails.

All three examples use the same `camel.ssl.*` property-based configuration. The JDK version and provider differ, but the approach is consistent.

### JDK 27: Native PQC in TLS

JDK 27 adds PQC named groups directly to SunJSSE via [JEP 527](https://openjdk.org/jeps/527){:target="_blank"}. This is the cleanest path: no third-party TLS providers, no security property workarounds. Everything is built into the JDK.

The application bootstrap is minimal:

```java
public class PQCSSLContextApplication {
    public static void main(String[] args) throws Exception {
        Main main = new Main();
        main.run(args);
    }
}
```

The entire PQC configuration lives in `application.properties`:

```properties
camel.server.enabled=true
camel.server.port=8443
camel.server.useGlobalSslContextParameters=true

camel.ssl.enabled=true
camel.ssl.secureSocketProtocol=TLSv1.3
camel.ssl.selfSigned=true
camel.ssl.trustAllCertificates=true
camel.ssl.namedGroups=X25519MLKEM768,x25519
camel.ssl.cipherSuites=TLS_AES_256_GCM_SHA384,TLS_AES_128_GCM_SHA256
```

That is it. No keystores to generate, no shell scripts to run. Camel generates a self-signed certificate at startup, configures PQC named groups, and serves HTTPS on port 8443.

The handshake verification in the route uses Groovy to access Camel's global SSLContext:

```groovy
def sslCtx = camelContext.getSSLContextParameters().createSSLContext(camelContext)
def serverSocket = sslCtx.getServerSocketFactory().createServerSocket(0)
def clientSocket = sslCtx.getSocketFactory().createSocket("localhost", serverSocket.getLocalPort())
def sslParams = clientSocket.getSSLParameters()
sslParams.setNamedGroups(["X25519MLKEM768"] as String[])
clientSocket.setSSLParameters(sslParams)
clientSocket.startHandshake()
```

The client offers only `X25519MLKEM768` with no classical fallback, so a successful handshake definitively proves PQC was negotiated.

### JDK 21 and JDK 24: BouncyCastle JSSE

For JDK 21 and 24, the JDK's SunJSSE does not yet support PQC named groups in TLS. [BouncyCastle's JSSE provider](https://www.bouncycastle.org/documentation/documentation-java/tls/bcjsse/){:target="_blank"} (BCJSSE) bridges this gap.

The configuration is almost identical to JDK 27, with two additions: `camel.ssl.provider=BCJSSE` to select the BouncyCastle TLS provider, and `secp256r1` in the named groups for ECDSA certificate verification:

```properties
camel.server.enabled=true
camel.server.port=8443
camel.server.useGlobalSslContextParameters=true

camel.ssl.enabled=true
camel.ssl.provider=BCJSSE
camel.ssl.selfSigned=true
camel.ssl.trustAllCertificates=true
camel.ssl.secureSocketProtocol=TLSv1.3
camel.ssl.namedGroups=X25519MLKEM768,secp256r1
```

The bootstrap class still needs to register the BouncyCastle providers before Camel starts, and remove `ECDH` from `jdk.tls.disabledAlgorithms` (which BCJSSE interprets broadly, preventing EC credentials from working):

```java
public class PQCSSLContextApplication {
    public static void main(String[] args) throws Exception {
        String disabled = Security.getProperty("jdk.tls.disabledAlgorithms");
        if (disabled != null) {
            disabled = disabled.replaceAll(",\\s*ECDH\\b", "");
            Security.setProperty("jdk.tls.disabledAlgorithms", disabled);
        }
        Security.addProvider(new BouncyCastleProvider());
        Security.insertProviderAt(new BouncyCastleJsseProvider(), 1);

        Main main = new Main();
        main.run(args);
    }
}
```

The route itself uses `camelContext.getSSLContextParameters().createSSLContext(camelContext)` to get the SSLContext, just like the JDK 27 example. The only difference is that on JDK 21, the standard `SSLParameters.setNamedGroups()` API does not exist yet, so the verification route uses BouncyCastle's `BCSSLParameters` to set named groups on the client socket:

```groovy
def sslCtx = camelContext.getSSLContextParameters().createSSLContext(camelContext)
def serverSocket = sslCtx.getServerSocketFactory().createServerSocket(0)
// ...
def bcClientParams = new org.bouncycastle.jsse.BCSSLParameters()
bcClientParams.setNamedGroups(["X25519MLKEM768", "secp256r1"] as String[])
((org.bouncycastle.jsse.BCSSLSocket) cs).setParameters(bcClientParams)
cs.startHandshake()
```

Earlier versions of these examples had to generate certificates programmatically, build keystores in memory, and create the SSLContext manually in Groovy. All of that is now handled by Camel's `camel.ssl.*` configuration.

### JDK 24 bonus: native ML-KEM

What makes the JDK 24 example unique is the `/api/verify-kem` endpoint, which exercises JDK 24's native `javax.crypto.KEM` API directly. This endpoint runs cross-provider interoperability tests between the JDK's built-in ML-KEM implementation and BouncyCastle's, including key re-encoding through standard X.509/PKCS#8 formats.

One detail worth noting: cross-provider key interoperability requires re-encoding keys through standard formats. When you generate an ML-KEM key pair with one provider and pass the key objects directly to another provider's `KEM.getInstance()`, you get an `InvalidKeyException`. The fix is to export the keys via `getEncoded()` and re-import them through `X509EncodedKeySpec` and `PKCS8EncodedKeySpec`:

```groovy
def bcKf = KeyFactory.getInstance("ML-KEM", "BC")
def reEncodedPub = bcKf.generatePublic(
    new X509EncodedKeySpec(jdkKp.getPublic().getEncoded()))
def reEncodedPriv = bcKf.generatePrivate(
    new PKCS8EncodedKeySpec(jdkKp.getPrivate().getEncoded()))
```

This works because both providers use the same standard key encoding formats, even though their internal key representations differ.

## Verifying PQC at the network level

Running the examples and seeing `pqcVerified: true` is one thing. Verifying at the network level that PQC key exchange actually happened is another.

You can use `tshark` to capture and inspect the TLS handshake:

```bash
tshark -i lo -f "tcp port 43567" -Y "tls.handshake" \
    -V -o tls.keylog_file:/dev/null 2>/dev/null | \
    grep -E "(Handshake Protocol|named_group|key_share|supported_groups)"
```

In the capture you will see the TLS 1.3 HelloRetryRequest flow. The client's first ClientHello advertises `X25519MLKEM768` in its `supported_groups` extension. The server agrees on the PQC group and sends a HelloRetryRequest asking the client to provide a key share for `X25519MLKEM768`. The client's second ClientHello includes a 1216-byte key share, which is the combined X25519 (32 bytes) + ML-KEM-768 (1184 bytes) public key. That 1184-byte ML-KEM component is the unmistakable fingerprint of a post-quantum key exchange.

## Side-by-side comparison

| Aspect | JDK 27 Native | JDK 24 + BouncyCastle | JDK 21 + BouncyCastle |
|--------|--------------|----------------------|----------------------|
| TLS provider | SunJSSE | BCJSSE 1.83 | BCJSSE 1.83 |
| Configuration | `camel.ssl.*` properties | `camel.ssl.*` properties | `camel.ssl.*` properties |
| Certificates | `camel.ssl.selfSigned=true` | `camel.ssl.selfSigned=true` | `camel.ssl.selfSigned=true` |
| Native ML-KEM (KEM API) | Yes | Yes (JEP 496) | No |
| REST transport | HTTPS on 8443 | HTTPS on 8443 | HTTPS on 8443 |
| Extra dependencies | None | `bcprov`, `bctls` | `bcprov`, `bctls` |

## Running the examples

All three examples follow the same pattern: select the right JDK, build, and run.

### JDK 27

```bash
cd pqc-ssl-context
sdk use java 27.ea.11-open
mvn clean compile exec:exec
curl -k https://localhost:8443/api/verify-pqc
```

### JDK 24

```bash
cd pqc-kem-jdk24
sdk use java 24.0.1-tem
mvn clean compile exec:exec
curl -k https://localhost:8443/api/verify-pqc
curl -k https://localhost:8443/api/verify-kem
```

### JDK 21

```bash
cd pqc-ssl-context-jdk21
sdk use java 21.0.10-tem
mvn clean compile exec:exec
curl -k https://localhost:8443/api/verify-pqc
```

All three return `"pqcVerified": true` when the PQC TLS handshake succeeds.

## What is not done yet

I want to be clear: this work is not finished. What we have so far is the foundation, and there are real gaps remaining.

The changes so far are in Camel's core SSL layer. Individual Camel components that use TLS (HTTP, Kafka, JMS, AMQP, and many others) still need to adopt these settings for PQC to actually work end-to-end with each connector. Some components delegate to their own TLS stacks (Netty, Vert.x, the Kafka client library) and will need specific integration work to pass through the PQC named groups. We have started this for Netty, which now has a PQC fallback that auto-applies named groups, but the rest is still ahead of us.

The `camel.ssl.selfSigned=true` option is useful for demos and development, but production deployments need proper certificates and keystores. The property-based configuration (`camel.ssl.keyStore`, `camel.ssl.trustStore`) is already there for that.

The auto-configuration on JDK 25+ is a nice default, but it only helps when the JVM and the remote peer both support PQC named groups. In practice, most peers today will not, and the hybrid approach (X25519MLKEM768 falling back to X25519) handles that gracefully. But we have not tested this against a wide variety of real-world TLS endpoints yet.

The BouncyCastle JSSE path (JDK 21/24) works but requires a bootstrap class to register providers and tweak security properties. We would like to make this smoother, perhaps through auto-detection or a Camel extension, but that is not built yet.

Finally, there is the question of PQC at the application layer (signing, encryption, key encapsulation), not just the transport layer. The `camel-pqc` component covers some of this, but integrating PQC signatures and key management into Camel's broader security model is a longer-term effort.

Contributions are welcome if any of this interests you.

## References

- [FIPS 203 - ML-KEM (Module-Lattice-Based Key-Encapsulation Mechanism)](https://csrc.nist.gov/pubs/fips/203){:target="_blank"} - NIST
- [JEP 496: Quantum-Resistant Module-Lattice-Based Key Encapsulation Mechanism](https://openjdk.org/jeps/496){:target="_blank"} - OpenJDK
- [JEP 527: Quantum-Resistant Module-Lattice-Based Key Agreement for TLS](https://openjdk.org/jeps/527){:target="_blank"} - OpenJDK
- [CAMEL-23154: Add PQC named groups support to SSLContextParameters](https://issues.apache.org/jira/browse/CAMEL-23154){:target="_blank"} - Apache Camel
- [CAMEL-23158: Add PQC named groups and signature schemes to SSL configuration properties](https://issues.apache.org/jira/browse/CAMEL-23158){:target="_blank"} - Apache Camel
- [CAMEL-23159: Add signatureSchemes to SSLContextParameters](https://issues.apache.org/jira/browse/CAMEL-23159){:target="_blank"} - Apache Camel
- [BouncyCastle JSSE Provider](https://www.bouncycastle.org/documentation/documentation-java/tls/bcjsse/){:target="_blank"} - BouncyCastle
- [CNSA 2.0 Algorithm Guidance](https://media.defense.gov/2022/Sep/07/2003071834/-1/-1/0/CSA_CNSA_2.0_ALGORITHMS_.PDF){:target="_blank"} - NSA
