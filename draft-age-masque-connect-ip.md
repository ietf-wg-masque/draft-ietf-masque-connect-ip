---
title: "IP Proxying Support for HTTP"
abbrev: "HTTP IP Proxy"
docname: draft-age-masque-connect-ip-latest
category: std

ipr: trust200902
area: TODO
workgroup: TODO Working Group
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: T. Pauly
    name: Tommy Pauly
    role: editor
    organization: Apple Inc.
    email: tpauly@apple.com
 -
    ins: D. Schinazi
    name: David Schinazi
    organization: Google LLC
    email: dschinazi.ietf@gmail.com
 -
    ins: A. Chernyakhovsky
    name: Alex Chernyakhovsky
    organization: Google LLC
    email: achernya@google.com
 -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    organization: Ericsson
    email: mirja.kuehlewind@ericsson.com

--- abstract

This document describes a method of proxying IP packets over HTTP using
the Extended CONNECT method. This protocol is similar to CONNECT-UDP, but
allows transmitting arbitrary IP packets, without being limited to just
TCP like CONNECT or UDP like CONNECT-UDP.

--- middle

# Introduction

This document describes a method of proxying IP packets over HTTP using
the Extended CONNECT method {{!RFC8441}}. This protocol is similar to
CONNECT-UDP {{?I-D.ietf-masque-connect-udp}}, but allows transmitting
arbitrary IP packets, without being limited to just TCP like CONNECT
{{!I-D.ietf-httpbis-semantics}} or UDP like CONNECT-UDP.

The Extended CONNECT protocol defined for this mechanism is "connect-ip",
which is also referred to as CONNECT-IP in this document.

The CONNECT-IP protocol allows endpoints to set up a path for proxying IP
packets using an HTTP proxy. This can be used for various solutions that
include general-purpose packet tunnelling, such as for a point-to-point or
point-to-network VPN, or for limited forwarding of packets to specific
hosts.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

In this document, we use the term "proxy" to refer to the HTTP server that
responds to the CONNECT-IP request. If there are HTTP intermediaries
(as defined in Section 2.3 of {{!RFC7230}}) between the client and the proxy,
those are referred to as "intermediaries" in this document.

# The CONNECT-IP Protocol

CONNECT-IP is defined as a protocol that can be used with the Extended CONNECT
method, where the value of the ":protocol" pseudo-header is "connect-ip".

The ":authority" pseudo-header field contains the host and port of the proxy,
not an individual endpoint with which a connection is desired.

CONNECT-IP uses variables in the path to determine the scope of the request
for packet proxying. The optional variable "target" contains a hostname or
IP address of a specific host to which the client wants to proxy packets. 

Any 2xx (Successful) response indicates that the proxy is willing to open an IP
forwarding path between it and the client. Any response other than a successful
response indicates that the tunnel has not been formed.

A proxy MUST NOT send any Transfer-Encoding or Content-Length header fields in
a 2xx (Successful) response to the Extended CONNECT request. A client MUST treat
a successful response containing any Content-Length or Transfer-Encoding
header fields as malformed.

The lifetime of the tunnel is tied to the CONNECT stream. Closing the stream
(in HTTP/3 via the FIN bit on a QUIC STREAM frame, or a QUIC RESET_STREAM frame)
closes the associated forwarding path.

# Examples

The following example shows a point-to-network VPN setup, where a client
receives a set of local addresses, and can send to any remote server
through the proxy.

~~~
[[ From Client ]]                       [[ From Server ]]

                                        SETTINGS
                                        SETTINGS_ENABLE_CONNECT_[..] = 1

HEADERS + END_HEADERS
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /vpn
:authority = server.example.com

                                        HEADERS + END_HEADERS
                                        :status = 200
~~~

The following example shows an IP flow forwarding setup, where a client
receives a single local address and remote address it can use for sending
packets.

~~~
[[ From Client ]]                       [[ From Server ]]

                                        SETTINGS
                                        SETTINGS_ENABLE_CONNECT_[..] = 1

HEADERS + END_HEADERS
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /proxy?target=target.example.com
:authority = server.example.com

                                        HEADERS + END_HEADERS
                                        :status = 200
~~~

# Security Considerations

TODO Security


# IANA Considerations

## CONNECT-IP HTTP Upgrade Token

This document registers an entry in the "HTTP Upgrade Tokens"
registry that was established by {{!RFC7230}}.

Value: connect-ip
Description: The CONNECT-IP Protocol
Expected Version Tokens:
References: This document
      
--- back

# Acknowledgments
{:numbered="false"}

The design of this method was inspired by discussions in the MASQUE working
group around {{?I-D.ietf-masque-ip-proxy-reqs}}. The authors would like to
thank participants in those discussions for their feedback.
