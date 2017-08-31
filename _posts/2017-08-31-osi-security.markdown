---
layout: post
title: "Network Security - OSI"
date: 2017-08-31 08:30:00
categories: school network-security
---

As with most concepts, it is often best to create a limited vocabulary when
attempting to explain complicated concepts.  Within the security arena, this
vocabulary starts with the [OSI Security Architecture][osi] defined within
X.800 ITU-T Recommendation.

At a high level the OSI Security Architecture is comprised of three key concepts:

1. Security Attack
2. Security Mechanism
3. Security Service

Each one of these broad concepts is further defined in the OSI Security
Architecture model.

### Security Attacks

Generally, a security attack is any action that compromises data.  There are two
broad classifications of attacks: Active, and Passive.  The difference between
the two are pretty apparent by the naming.  Passive attacks do not actively
manipulate or change data, they merely observe.  Active attacks do actively
manipulate or change data.

The two types of passive attacks are:

1. Release of message data
2. Release of traffic analysis

Release of message data is pretty clear, the actual message or communication
was intercepted, and is known by the interceptor.  Traffic analysis is just
information about the message, such as length of message, where the message
came from and is destine, etc.

The four types of active attacks are:

1. Masquerade
2. Replay
3. Modification of messages
4. Denial of service

Masquerade is when the attacker attempts to pretend they are the initiator of
a message.  Replay is when a message is captured and reused in the future to
perform an action.  Modification of messages falls into the message integrity
realm, where the contents of the message is modified.  Denial of service is
when the attack attempts to block legitimate communications to the service.

### Security Mechanisms

Security mechanisms are tools one can use to add security to systems and
services.  Briefly outlined below are various specific security mechanisms:

* Encipherment
    * Use of algorithms to transform data to hide transmission contents
* Digital Signature
    * Use of algorithms which allows proof of authenticity of author
* Access Control
    * Mechanisms to enforce access rights
* Data Integrity
    * Mechanisms used to assure integrity of data
* Authentication Exchange
    * Mechanisms to ensure identity of an entity
* Traffic Padding
    * Insertions of bits in message gaps to thwart traffic analysis
* Routing Control
    * Enables selection of secure routes for certain data
* Notarization
    * Use of trusted third party to assure properties of data

Here is a list of various pervasive security mechanisms:

* Trusted Functionality
* Security Label
* Event Detection
* Security Audit Trail
* Security Recovery

### Security Services

Security services are services that ensures security for a system or
data transmission.  Below are a few security services defined:

* Authentication
    * Peer Entity Authentication
        * Provide confidence in identity of connected entities
    * Data-Origin Authentication
        * Provide assurance that the source is as claimed
* Access Control
    * Prevention of unauthorized use of resources
* Data Confidentiality
    * Connection Confidentiality
    * Connection-less Confidentiality
    * Selective-Field Confidentiality
    * Traffic-Flow Confidentiality
* Data Integrity
    * Connection Integrity with Recovery
    * Connection Integrity without Recovery
    * Selective-Field Connection Integrity
    * Connection-less Integrity
    * Selective-Field Connection-less Integrity
* Non-repudiation
    * Non-repudiation Origin
    * Non-repudiation Destination

By having a common structured vocabulary, we are able to clearly talk about
particular security concepts.

Most of this content was learned from the [Network Security Essentials][netsec]
book, chapter 1.

[osi]: http://www.itu.int/rec/T-REC-X.800-199103-I/e
[netsec]: https://www.amazon.com/Network-Security-Essentials-Applications-Standards/dp/0133370437/
