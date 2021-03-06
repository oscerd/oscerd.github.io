---
layout: post
title: Cryptographic Pills Episode 1 - Modern Cryptography
---

In the last days I started to look at the Cryptography's Theory again. I guess it's an interesting subject for anyone involved in the IT world: for developers and security specialists understand some key concepts is essential to manage particular scenarios. Yesterday there was the Google's announce about the [first SHA-1 collision](https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html): hash functions will be in one of the articles I currently have in mind. I'll try to write about Cryptographic pills in episodes and I hope you'll appreciate the subject. Let's start!

### In the beginning

Until the late 20th century Cryptography was considered an art form. Constructing good codes, breaking existing ones and be able to understand the algorithm behind a code relies on creativity and personal skill. There wasn't so much theory around the subject. With the Cryptography's spread in areas different from Military and Defense the whole picture radically changed: a rich theory and rigorous study made the Cryptography an applied Science. Today Cryptography is everywhere.

### Private-key Encryption

Cryptography was historically concerned with secret communications. In the past we were talking about ciphers, today we talk about encryption schemes: the concepts are the same. Two parts want to communicate in a secret manner and they decide to use a particular encryption scheme. An assumption in this scenario is that the communicating parties must have a way to initially sharing a key in a secret manner. 

A private-key Encryption scheme is based on three algorithms [1]:

1. A Key-generation algorithm __Gen__ is a probabilistic algorithm that outputs a key _k_ chosen according to some distribution that is determined by the scheme.
2. The Encryption Algorithm __Enc__ takes as input a key _k_ and a plaintext message _m_ and outputs a cyphertext _c_. We denote by the encryption of the plaintext _m_ using the key _k_ in the following way
<div>
$$Enc_k(m)$$
</div>
3. The Decryption Algorithm __Dec__ takes as input a key _k_ and a cyphertext message _c_ and outputs a plaintext _m_. We denote by the decryption of the cyphertext _c_ using the key _k_ in the following way
<div>
$$Dec_k(c)$$
</div>

- The set of all possible keys generated by the __Gen__ algorithm is denoted as _K_ and it is called the Key Space.
- The set of all "legal" messages (supported by the Encryption Algorithm __Enc__) is denoted as _M_ and it is called the Message Space.
- Since any Cyphertext is obtained by encrypting some plaintext under some key, the sets _K_ and _M_ together define a set of all possible Cyphertexts denoted with _C_. 

### Kerckhoffs's principle

From the definitions above it's clear that if an eavesdropping adversary knows the __Dec__ algorithm and the chosen key _k_, he will be able to decrypt the messages exchanged between the two communicating parts. The key must be secret and this is a fundamental point, but do we need to maintain secret the Encryption and Decryption algorithms too? In the late 19th century August Kerckhoffs gave his opinion:

	The cipher method must not be required to be secret, and it must be able to fall in the hands of the enemy without incovenience

It's something related to Open Source and Open Standards too. Today any Encryption Scheme is usually demanded to be made public. The main reasons for this can be summarized in the following points:

1. The public scrutiny make the design stronger.
2. If flaws exist it can be found by someone hacking on the scheme
3. Public design enables the establishment of standards.

The Kerckhoffs' principle is obvious, but in same cases it was ignored with disastrous results. Only publicly tried and tested algorithm should be used by organizations. Kerchoffs was one of the first person with an Open vision about these things.

### What's next

In the next articles I would like to talk about the theory behind the Cryptographic schemes strenght. How we can say a scheme is secure? What are the attack scenarios? What is the definition of a secure scheme?

[1] Introduction to Modern Cryptography, Chapman & Hall/Crc, Jonathan Katz, Yehuda Lindell, Chapter 1.
