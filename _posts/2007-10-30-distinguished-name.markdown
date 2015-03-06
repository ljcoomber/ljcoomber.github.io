---
layout: post
title:  "What's in a Distinguished Name"
date:   2007-10-30 19:33:27
categories: misc
---

This post first appeared on the [LShift web site](http://www.lshift.net/blog/2007/10/30/whats-in-a-distinguished-name/).

In one of our projects, we access a directory server via LDAP to obtain subject distinguished names (DNs) of X509 certificates. The team that provision the directory recently asked a simple question: what order should the components of the subject DN be in? Below follows my explanation of the answer. Please note that the referenced specifications are not necessarily the latest versions, but are the ones supported by my version of OpenSSL and Sun Directory Server.

The subject DN is defined in [RFC 2459](http://www.ietf.org/rfc/rfc2459.txt) by the [X.501](http://www.itu.int/rec/T-REC-X.501-199311-S/en) type Name as an ASN.1 structure. It consists of a sequence of Relative Distinguished Names (RDNs), which are themselves sets of attribute type and value pairs. For example, given the following DN:

    C=UK, ST=England, L=London, O=LShift Ltd., OU=Development, CN=Lee Coomber/UID=123456

‘C=UK’ is an RDN, ‘C’ is an attribute type, and ‘UK’ is an attribute value. Normally, an RDN only consists of one attribute type / value pair, but more are allowed such as in the example of the final RDN above which consists of the pairs ‘CN=Lee Coomber’ and ‘UID=123456′.

We are ultimately interested in how to compare two DNs and this is defined in [RFC 2252](http://www.ietf.org/rfc/rfc2252.txt)  as being a match type of “distinguishedNameMatch”. This is a standard match type defined in X501 which states that a presented DN matches a target DN if, and only if:

* the number of RDNs is the same
* each corresponding RDN has the same number of attribute value pairs
* each of the attribute pairs that are of the same type have matching values as determined by the match rule for that type

Note that the order of the attribute pairs within an RDN does not matter.

The actual format of the certificates is defined in ASN.1 which as a binary format is not readable by anyone you would want to sit and have a drink with. An (almost) human readable form can be obtained using the [OpenSSL](http://www.openssl.org/) command “asn1parse”. Using the example “self-signed” certificate given above and the following command:

    $ openssl asn1parse -in server.crt

we can see the subject DN as part of the certificate in the diagram below:

![DN Breakdown][1]

There are two things to note. The first, is that the order shown reading top-down is the order of the RDNs as they are actually stored in the certificate. The second, is the unambiguous distinction between RDNs and attribute pairs. Given the match definition above we could determine whether a given certificate subject DN matches another by looking at the output of this command for the two certificates (ignoring the details of the attribute value comparisons for the purposes of this discussion).

We use strings to enter the DNs into the LDAP server and the method for representing DNs in this way is defined in [RFC 2253](http://www.ietf.org/rfc/rfc2253.txt). The main considerations are:

* RDN order is reversed
* RDNs are separated by commas
* attribute pair order does not matter
* attribute pairs in multi-valued RDNs are separated by the plus sign
* attribute type and value are separared by the equals sign
* reserved characters are escaped using a backslash

So given our example earlier, the string representation would be:

    UID=123456+CN=Lee Coomber,OU=Development,O=LShift Ltd.,L=London,ST=England,C=UK

OpenSSL provides a mechanism for doing this though be warned that by default it uses its own more “readable” mechanism for displaying DNs that are not compliant with the standards described here. To display DNs in accordance with RFC 2253 use the ‘nameopt’ command line switch with the parameter “RFC2253″. For example, to see the formatted subject DN of our test certificate:

    $ openssl x509 -subject -nameopt RFC2253 -noout -in server.crt subject= UID=123456+CN=Lee Coomber,OU=Development,O=LShift Ltd.,L=London,ST=England,C=UK

  [1]: /assets/distinguished-name/dn-breakdown.png

