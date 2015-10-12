---
layout: post
title: Forward Secrecy
comments: true
---

Traditionally, when someone sends a message, the message is encrypted using the receiver’s public key (or by a session key which is encrypted by the receiver’s public key). It looks secure at first glance, since only the receiver can decrypt the message with the private key. However, if an attack is able to record all the exchanged messages, and somehow obtains the private key at some point in the future, he can decrypt all the recorded messages. To solve this kind of issue, here comes the so called **forward secrecy**.

With forward secrecy, the attacker won’t be able to decrypt any messages recorded, even if peers’ private keys are compromised at some point in the future. To achieve this, instead of encrypting all the messages using the same key, peers would negotiate a session key through an ephemeral key exchange (e.g. Diffie–Hellman key exchange).

## Forward secrecy in action

The following uses [OTR](https://otr.cypherpunks.ca/otr-wpes.pdf) as an example to illustrate how forward secrecy is achieved.

1. When Alice wants to start a conversation with Bob, she picks a random value `x<sub>1</sub>`, computes `A=g^x<sub>1</sub>`, and sends her half of key exchange message `A, Sign<sub>k<sub>A</sub></sub>(A)` to Bob.
2. Upon reception, Bob picks a random value `y<sub>1</sub>`, computes `B=g^y<sub>1</sub>`, and sends his half of key exchange message `B, Sign<sub>k<sub>B</sub></sub>(B)` back to Alice.

At this point, both Alice and Bob can compute the session key `s=Hash(B^x<sub>1</sub>)=Hash(A^y<sub>1</sub>)`, and they both can also verify the messages are from the correct peer with the attached signature.

Moreover, since the session keys are not associated with the peers’ long term public / private key pairs, the attackers won’t be able to decrypt the messages even if he somehow obtains the private keys in the future. OTR takes one step further by starting a new key exchange on each message sent in a conversation and discarding the old session key once a new one is negotiated, to reduce the possible time window for attacks.

## Forward secrecy, asynchronously

As one can easily notice, the above method works perfect in any synchronous environment, where several handshake messages need to be exchanged before any real messages can be sent. This is not an ideal situation e.g. for emails where users exchange mails asynchronously, or mobile platforms where users are not always online.

To tackle with this issue, Moxie [proposed](https://whispersystems.org/blog/asynchronous-security/) a simple trick.

1. Upon registration, Bob generates 100 signed key exchange messages (called “prekeys”), and sends his half of key exchange messages for each prekey to the server.
2. When Alice wants to start a conversation with Bob, she connects to the server and fetches Bob’s next prekey.
3. Alice generates her half of key exchange message, computes the session key using the fetched prekey, and encrypts the text.
4. Alice sends the message to Bob, which includes the ID of the fetched prekey, her half of key exchange message, and the encrypted text.
5. When Bob reads the message later, he can computes the session key and decrypt the message.

With this setup, the peers don’t need to be online at the same time, and forward secrecy is guaranteed by the fact that the server never sends out the same prekey twice, and client never accepts the same prekey twice.
