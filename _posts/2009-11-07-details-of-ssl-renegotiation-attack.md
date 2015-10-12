---
layout: post
title: Details of SSL Renegotiation Attack
comments: true
---

This MITM attack is against all versions of the SSL/TLS protocols, allowing an attacker to inject any amount of data into the beginning of the application protocol stream.

I assume you understand how SSL/TLS works, especially the renegotiation part, which is the weak point leading to the following three attacks. Also, a basic understanding of the HTTP protocol is recommended.

## 1st scenario: when the client certificate is required
In the ideal case, the client certificate is requested when the server sends its certificate. However, in the real world, it’s requested only when the client requests something that needs the client certificate. Instead of a brand new handshake procedure, this would trigger the server to start a renegotiation.

Unfortunately, as HTTP has no way to instruct the client to resubmit the request after the renegotiation, the server has to reply with the requested content after the renegotiation, resulting a gap of authentication.

The sequence of this attack is summarized below:

1. client –> attacker : handshake1
2. attacker –> server: handshake2
3. attacker –> server: request something
4. attacker <– server: renegotiation, requesting client certificate
5. client <–> attacker <–> server: attacker forwards handshake1, and handshake3 is built which is only readable by the server and the client
6. client <– attack <– server: the server responds with the content requested at step 3

Now, the client gets what he hasn’t required…

## 2nd scenario: different cipher suites used

Some websites are configured that different resources are protected by different cipher suites. If the client requests a resource protected by another cipher suite, a renegotiation is needed. However, the request from the client is buffered by the server for future use, resulting to this attack. Just have a look at the following example.

The attacker sends a request:

{% highlight http %}
GET /index.html HTTP/1.1
Host: server_address
Connection: keep-alive
GET /evil.html HTTP/1.1
x-ignore:
{% endhighlight %}

The second request is completed by the client:

{% highlight http %}
GET /good.html HTTP/1.1
Host: server_address
Cookie: client_cookie
{% endhighlight %}

Unfortunately, the second request is consider by the server as below:

{% highlight http %}
GET /evil.html HTTP/1.1
x-ignore: GET /good.html HTTP/1.1
Host: server_address
Cookie: client_cookie
{% endhighlight %}

The client get */evil.html* instead of */good.html*...

## 3rd scenario: client initiated renegotiation

This attack is based on the fact that the client can also launch a renegotiation, and the server would buffer the previous request and use them after the renegotiation. It’s reasonable to believe that this is the most dangerous scenario as it requires nothing special from the server. Here is the details:

1. The attacker builds an SSL connection with the server, and sends the request:

{% highlight http %}
GET /evil.html HTTP/1.1
R
x-ignore:
{% endhighlight %}

Here, the *R* would trigger a renegotiation, while the *x-ignore:* is an unfinished header line without line termination, which would be buffered and used by the server later.

2. The attacker forwards the messages, and builds a connection between the client and the server.

3. The client sends a request to the server:

{% highlight http %}
GET /index.html HTTP/1.1
Host: server_address
Connection: keep-alive
{% endhighlight %}

However, due to the injected request from the attacker, the server considers the whole request as:

{% highlight http %}
GET /evil.html HTTP/1.1
R
x-ignore: GET /index.html HTTP/1.1
Host: server_address
Connection: keep-alive
{% endhighlight %}

It will return the */evil.html* rather than */index.html* as expected.

Now, I’d like to **remind** you again that all these attacks are against the SSL/TLS protocols, rather than a specific implementation, which means that either the client or the server could smell anything about the attacker. To **summarize**, the attacker tries to renegotiate the SSL connection before the client really communicates with the server, and the request made by the attacker before the negotiation will be replied to the client, which may lead to serious attacks. Quite easy to understand and implement, right?

A draft has already been [proposed](https://svn.resiprocate.org/rep/ietf-drafts/ekr/draft-rescorla-tls-renegotiate.txt) to fix this problem by cryptographically binding renegotiation handshakes to the enclosing TLS channel, which is not supported by current clients and servers. The fix is to include the digest data from the **FINISHED** message of the previous handshake in **CLIENT-HELLO** or **SERVER-HELLO** of the renegotiation.

Finally, I’d like to remind you about the vulnerabilities of DNS and BGP discovered last year, as I consider the attack against SSL is equally disastrous as those two.

**UPDATED**: The final fix for this issue is available as [RFC 5746](http://tools.ietf.org/html/rfc5746).
