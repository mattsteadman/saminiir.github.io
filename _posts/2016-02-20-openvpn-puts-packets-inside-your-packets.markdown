---
layout: post
title:  "OpenVPN puts packets inside your packets"
date:   2016-02-20 07:00:00
categories: networking
permalink: openvpn-puts-packets-inside-your-packets/
---

I realized I do not know exactly *how* OpenVPN does its magic in Linux.

I mean, sure, it encrypts and tunnels the packets to another endpoint, but can you describe the exact steps to achieve this? If not, read on. In fact, read on anyway for good measure.

This is the general idea I had pictured in my mind how OpenVPN works:

* Receive packets
* Encrypt them
* Tunnel them
* Send to destination

Several questions arise, though:

* How is tunneling achieved in OpenVPN's case?
* What is the mechanism for receiving and sending the packets?
* How is encryption and decryption of the data achieved?

Let's delve deeper into these questions.

# Contents
{:.no_toc}

* Will be replaced with the ToC, excluding the "Contents" header
{:toc}

# Tunneling 

What is tunneling? According to the magnificent TCP/IP Illustrated[^1]:

_Tunneling, generally speaking, is the idea of carrying lower-layer traffic in higher-layer (or equal-layer) packets. For example, IPv4 can be carried in an IPv4 or IPv6 packet; Ethernet can be carried in a UDP or IPv4 or IPv6 packet, and so on._

In other words, tunneling is achieved by injecting packets into the payload of other packets. Reminds me of [a certain meme](http://www.urbandictionary.com/define.php?term=Yo+Dawg).

Various protocols for establishing tunnels exist, the prominent ones being _Generic Routing Encapsulation_ (GRE), Microsoft's _Point-to-Point Tunneling Protocol_ (PPTP) and the _Layer 2 Tunneling Protocol_ (L2TP).

The result is that virtual links between computers in different networks can be set up. An example of this is a company's private network, which can be reached from the public network via tunneling.

Note, however, that tunneling itself does not guarantee secure communications. Encryption has to be applied to the payload, assuming you've succeeded to exchange cryptographic keys securely in an insecure environment.

# Encryption

When using tunneling, there's a high chance that the information you transmit is of delicate nature. Hence, cryptographic measures have to be taken into use to prevent anyone from snooping in on the data.

Since OpenVPN is an userspace application, _Transport Layer Security_ (TLS) is a natural choice for the security protocol[^2]. TLS operates in the layers above the networking stack, so implementing it in an application's scope is easier than more involved lower-level protocols that require specific[^swan] kernel modules. If you've ever browsed a site with the protocol HTTPS, then you have used TLS for authentication and encryption.

The basic steps for secure communication are implemented in OpenVPN as follows[^3]:

1. The X509 PKI (public key infrastructure) for session authentication
2. The TLS protocol for key exchange
3. The OpenSSL cipher-independent EVP interface for encrypting tunnel data
4. The HMAC-SHA1 algorithm for authenticating tunnel data

The X509 Public Key Infrastructure is an ITU-T standard which relies on Certificate Authorities (CA) for digital signatures of certificates. Through this, a client communicating to a server can verify that the server is really who it claims to be.

The TLS protocol is the ubiquitous privacy protocol suite used to encrypt, well, pretty much anything dealing with Internet. Namely, TLS is responsible of negotiating a secure communication channel between two parties. This mainly consists of certificate and key exchanges, which can be used to authenticate the other party and subsequently encrypt and decrypt the data. 

Key exchange is a fascinating topic as it revolves around the problem of two parties attempting to establish a secure communication channel in an insecure medium. Turns out, the Diffie-Hellman method (DH)[^dh] is one such clever approach, where two parties can agree upon a key without revealing that key to outsiders.

Another option is to use RSA[^rsa] public key cryptography for the key exchange. Whereas DH utilizes finite field arithmetic to achieve secure key exchange, RSA takes advantage of the integer factorization problem. This problem deals with the fact that it is hard to factor the numbers that a semiprime number is a product of.

To abstract upon the routine cryptographic functions, OpenVPN uses the EVP interface[^evp] of the OpenSSL library. It provides encryption/decryption, signing/verifying, key derivation, hash functions and message authentication code (MAC) functions.

Lastly, Message Authentication Code (MAC) is an important part of secure communications. It is used to verify message integrity, but also to authenticate the message's origins.

# OpenVPN protocol

The general view of the TLS handshaking is as follows[^tls-spec]:

{% highlight bash %}
        Client                                              Server

   ClientHello                      -------->
                                                               ServerHello
                                                               Certificate*
                                                         ServerKeyExchange*
                                                        CertificateRequest*
                                    <--------              ServerHelloDone
   Certificate*
   ClientKeyExchange
   CertificateVerify*
   [ChangeCipherSpec]
   Finished                         -------->
                                                        [ChangeCipherSpec]
                                    <--------                     Finished

   GenerateTLSBindingKey                             GenerateTLSBindingKey

   Application Data                 <------->             Application Data
{% endhighlight %}

The _ClientHello_ message is pretty self-explanatory. The client is required to send it to the server as its first message. It includes data like the TLS version requested, a randomized struct consisting of the current sender's UNIX time and a 28-byte securely generated random number, requested cipher suite and requested compression algorithm.

The server responds with a ServerHello, Certificate, CertificateRequest TLS records.

The _ServerHello_ record is very much like the ClientHello message, except it has the responsibility of, among other things, choosing the ciphersuite that satisfies both parties' requirements. In my OpenVPN test session, `TLS_RSA_WITH_AES_256_CBC_SHA` was chosen as the ciphersuite. This means that RSA is used for the key exchange.

The _Certificate_ record is required whenever the key exchange method is anything other than Anonymous Diffie-Hellman. This is because both parties have to validate the identity of each other to prevent Man-in-the-Middle attacks. Finally, the certificate record contains the certificates of the server..

The _ServerKeyExchange_ record is only sent when "the server Certificate message (if sent) does not contain enough data to allow the client to exchange a premaster secret."[^tls-spec] Thus, when using public-key cryptograpghy, you are unlikely to see this message.

The _CertificateRequest_ record sent by the server is followed by the _ServerHelloDone_ message, which indicates to the client that the server has finished its part of the key exchange.

Next, the client sends a _Certificate_ TLS record to the server, containing the client's certificate used for authentication.

The _ClientKeyExchange_ message is always sent next by the client. It contains either the RSA-encrypted secret or Diffie-Hellman parameters for establishing a shared secret between the two parties.

The _CertificateVerify_ message is sent by the client if explicit verification of the client's certificates is needed.

Finally, the _Finished_ message is sent by both parties and it is the first encrypted message. Once both parties receive and validate the message, Application Data starts to flow in the encrypted tunnel between the parties. Before the Finished messages, _ChangeCipherSpec_ message is used to indicate any changes in the cipher specifications.

# TUN/TAP devices

The tunneling of data in OpenVPN is achieved through TUN/TAP devices[^4]. Simply put, TUN/TAP devices expose the operating system's network traffic as virtual interfaces. This traffic can then be operated upon by the userspace application that is bound to the TUN/TAP virtual interface.

A TUN device operates on IP packets (layer 3), and a TAP device operates on Ethernet frames (layer 2). The distinction is important, since operating on different networking layers enable different use cases. For example, if one wants Ethernet bridging, OpenVPN has to utilize TAP devices. For simple routing of traffic, TUN devices are a cheaper choice. A handy summary of the differences between bridging and routing is presented
[here](https://community.openvpn.net/openvpn/wiki/309-what-is-the-difference-between-bridging-and-routing).

With OpenVPN and TUN/TAP devices, the Linux Kernel networking stack is involved and does the heavy-lifting of the traffic. The only purpose of the TUN/TAP device is to allow the user-space application, OpenVPN, to operate on the raw Ethernet frames or IP packets. This is where tunneling and encryption is applied. The Ethernet frame or IP packet is encrypted and wrapped inside another IP packet with appropriate headers for delivery, and vice versa.

The specific source code for how the TUN/TAP device is opened in OpenVPN is in the source file [tun.c](https://github.com/OpenVPN/openvpn/blob/v2.3.10/src/openvpn/tun.c#L1504). It closely reflects the [TUN/TAP documentation](https://www.kernel.org/doc/Documentation/networking/tuntap.txt) of the Linux Kernel.

# Conclusion

Implementing secure communications is a fascinating matter but not a trivial one. The fundamental problem is that the Internet as we know it, is an unsecure medium. Anyone can snoop in on the data flowing through different networks, so establishing a secure communication channel is tricky. 

As a bonus, I found out there's a module named [gremlin](https://github.com/OpenVPN/openvpn/blob/v2.3.10/src/openvpn/gremlin.c) in OpenVPN, which can be enabled with the `--gremlin` flag in debug mode. It will wreak havoc on your VPN tunnels, and is probably used to test OpenVPN. :-)

# Sources

[^1]:<https://en.wikipedia.org/wiki/TCP/IP_Illustrated#Volume_1:_The_Protocols>
[^2]:<https://community.openvpn.net/openvpn/wiki/WhyChooseTLSAsOpenvpnsUnderlyingAuthenticationAndKeyNegotiationProtocol>
[^3]:<https://community.openvpn.net/openvpn/wiki/OverviewOfOpenvpn>
[^4]:<https://www.kernel.org/doc/Documentation/networking/tuntap.txt>
[^5]:<https://community.openvpn.net/openvpn/wiki/309-what-is-the-difference-between-bridging-and-routing>
[^swan]:<https://wiki.strongswan.org/projects/strongswan/wiki/KernelModules>
[^tls-spec]:<https://tools.ietf.org/html/rfc5246>
[^evp]:<https://wiki.openssl.org/index.php/EVP>
[^dh]:<https://www.ietf.org/rfc/rfc2631.txt>
[^rsa]:<https://www.ietf.org/rfc/rfc2437.txt>
