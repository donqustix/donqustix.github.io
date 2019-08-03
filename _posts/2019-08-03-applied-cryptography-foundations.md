---
layout: post
mathjax: true
mermaid: true
title:  "Applied Cryptography: Foundations"
date:   2019-08-03 18:31:23 +0300
category: cryptography
---

The following is my summary of the first section from the first chapter of *Applied Cryptography: Protocols, Algorithms and Source Code in C* (Schneier, 2015).

## 1.1 Terminology

The study of techniques for secure communication in presense of unauthorized parties is **cryptography**.

### Sender and Receiver

Suppose a sender wants to send a message to a receiver securately: the message shall not be read by anyone, except both the sender and receiver.

### Messages: Encryption and Decryption

A message is unencrypted information called **plaintext** to be sent to a receiver. The process of encoding plaintext in such a way as to deny its intelligible content to unauthorized parties is **encryption**.

A message containing encrypted information called **ciphertext** is an encrypted message. The process of encoding ciphertext back into plaintext is **decryption**.

This is all shown in Figure 1.1.

<div class="mermaid" align="center">
graph LR
   A(( ))-->|Plaintext| E[Encryption]
   E --> |Ciphertext| D[Decryption]
   D --> |Original Plaintext| B(( ))
</div>
<small>Figure 1.1 Encryption and decryption.</small>

From now on, we shall denote a message, plaintext, and ciphertext by $$M$$, $$P$$, and $$C$$ respectively.

The encryption function $$E$$ operates on $$M$$ to produce $$C$$:

$$E(M)=C$$
{: style="text-align: center;"}

The decryption function $$D$$ operates on $$C$$ to produce $$M$$:

$$D(C)=M$$
{: style="text-align: center;"}

Successful decryption of a message's plaintext after its encryption requires the following identity hold true:

$$D(E(M))=M$$
{: style="text-align: center;"}

### Authentication, Integrity, and Non-repudiation

In addition to providing confidentiality described above, cryptography also serves 3 more important aspects of information security:

* **Authentication** is the act of ascertaining an incoming message's origin to a receiver.
* **Integrity** is the assurance of the immutability of a message on its way to a receiver.
* **Non-repudiation** is the non-availability of possibility for a sender to successfully dispute its authorship of the sent message.

### Algorithms and Keys

A **cryptographic algorithm**, also called a **cipher**, is an algorithm used for performing encryption or decryption.

A **restricted algorithm** is an algorithm, the security of which is based on keeping the way it works a secret. In today's modern world, restricted algorithms have only historical interest, and they are nothing more but yet another layer of security.

The functional output of a cryptographic algorithm is determined by a piece of information called a **key**, denoted by $$K$$.

For encryption algorithms, a key specifies the transformation of plaintext into ciphertext, and vice versa for decryption algorithms (see Figure 1.2). 

<div class="mermaid" align="center">
graph LR
   A(( )) --> |Plaintext| E[Encryption]
   K1(( )) --> |Encryption Key| E
   E --> |Ciphertext| D[Decryption]
   K2(( )) --> |Decryption Key| D
   D --> |Original Plaintext| B(( ))
</div>
<small>Figure 1.2 Encryption and decryption with keys.</small>

The fact that encryption and decryption algorithms depend on the keys is denoted by the $$K_i$$ subscript:

$$E_{K_1}(M)=C\\D_{K_2}(C)=M$$
{: style="text-align: center;"}

And the following identity must hold true:

$$D_{K_2}(E_{K_1}(M))=M$$
{: style="text-align: center;"}

A **cryptosystem** is a tuple $$(\mathcal{P},\mathcal{C},\mathcal{K},\mathcal{E},\mathcal{D})$$ with the following properties:

1. $$\mathcal{P}$$ is a **plaintext space**, a set of all possible plaintexts.
2. $$\mathcal{C}$$ is a **ciphertext space**, a set of all possible ciphertexts.
3. $$\mathcal{K}$$ is a **keyspace**, a set of all possible keys.
4. $$\mathcal{E}=\{E_k : k \in \mathcal{K}\}$$ is a set of all encryption functions $$E_k : \mathcal{P} \rightarrow \mathcal{C}$$.
5. $$\mathcal{D}=\{D_k : k \in \mathcal{K}\}$$ is a set of all decryption functions $$D_k : \mathcal{C} \rightarrow \mathcal{P}$$.
6. For each $$e \in \mathcal{K}$$, there is $$d \in \mathcal{K}$$ such that $$\mathcal{D}_d(\mathcal{E}_e(p))=p$$ for each $$p \in \mathcal{P}$$.

The mathematical definition of a cryptosystem is modified depending on the type of the cryptographic algorithm.

There are two general types of cryptographic algorithms: symmetric- and public-key algorithms.

### Symmetric-Key Algorithms

**Symmetric-key algorithms** are cryptographic algorithms, the encryption key of which can be derived from the decryption key and vice versa. In most symmetric algorithms, the encryption and decryption keys are equal.

The security of symmetric-key algorithms is the key's security; as long as the communication needs to remain secret, the key must remain secret.

Symmetric-key algorithms are divided into two categories:

* **Stream algorithms**, or **stream ciphers**, are symmetric-key algorithms that operate on the plaintext a single byte at a time.
* **Block algorithms**, or **block ciphers**, are symmetric-key algorithms that operate on the plaintext in a group of bits called a **block**.

### Public-Key Algorithms

**Public-key algorithms**, also called **assymetric algorithms**, are cryptographic algorithms that use two different keys: the **public-key** that is used for encryption and can be made public; and the **private key** that is used for decryption and cannot be obtained from the public-key.

In public-key algorithms, effective security requires only the private key remain secret; the public-key can be openly distributed without compromising security.

### Cryptoanalysis

**Cryptoanalysis** is the study of techniques of recovering a message's plaintext, even if the key is unknown. Successful cryptoanalysis may recover the plaintext or the key, or find weakness in a cryptosystem that eventually lead to the previous problems.

An attempted cryptoanalysis is called an **attack**. The fundamental assumtion in cryptoanalysis states that the security of a cryptosystem depends entirely on the security of the key.

There are four (4) general types of cryptoanalytic attacks. Each of them assumes that the cryptoanalyst has complete knowledge of the encryption algorithm used:

1. #### Ciphertext-only attack

   Given: $$C_1=E_{k \in \mathcal{K}}(P_1),C_2=E_{k \in \mathcal{K}}(P_2),...,C_N=E_{k \in \mathcal{K}}(P_N)$$.  
   Deduce: either $$P_1,P_2,...,P_N$$; the keys $$k \in \mathcal{K}$$ used to encrypt the messages; or an algorithm to infer $$P_{N+1}$$ from $$C_{N+1}=E_{k \in \mathcal{K}}(P_{N+1})$$.

2. #### Known-plaintext attack

   Given: $$P_1,C_1=E_{k \in \mathcal{K}}(P_1),P_2,C_2=E_{k \in \mathcal{K}}(P_2),...,P_N,C_N=E_{k \in \mathcal{K}}(P_N)$$.  
   Deduce: either the keys $$k \in \mathcal{K}$$, or an algorithm to infer $$P_{N+1}$$ from $$C_{N+1}=E_{k \in \mathcal{K}}(P_{N+1})$$.

3. #### Choosen-plaintext attack

   Given: $$P_1,C_1=E_{k \in \mathcal{K}}(P_1),P_2,C_2=E_{k \in \mathcal{K}}(P_2),...,P_N,C_N=E_{k \in \mathcal{K}}(P_N)$$, where the cryptoanalyst gets to choose $$P_1,P_2,...,P_N$$.  
   Deduce: either the keys $$k \in \mathcal{K}$$, or an algorithm to infer $$P_{N+1}$$ from $$C_{N+1}=E_{k \in \mathcal{K}}(P_{N+1})$$.

4. #### Adaptive-chosen-plaintext attack

   This is a special case of a choosen-plaintext attack. Not only can the cryptoanalyst choose the plaintext that is encrypted, but he can also modify his choice based on the results of previous encryption.

5. #### Choosen-ciphertext attack

   Given: $$C_1,P_1=D_{k \in \mathcal{K}}(C_1),C_2,P_2=D_{k \in \mathcal{K}}(C_2),...,C_N,P_N=D_{k \in \mathcal{K}}(C_N)$$.
   Deduce: the keys $$k \in \mathcal{K}$$.

6. #### Choosen-key attack

   This one is strange.

7. #### Rubber-hose cryptoanalysis

   The most effective attack. The cryptoanalyst threatens, blackmails, tortures, or rapes someone until they give him the key.

### Security of Algorithms

Go read that book!
