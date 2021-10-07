---
title: "IP Proxying Support for HTTP"
abbrev: "HTTP IP Proxy"
docname: draft-age-masque-connect-ip-latest
category: std

ipr: trust200902
area: TSV
workgroup: MASQUE

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
 -
    ins: M. Westerlund
    name: Magnus Westerlund
    organization: Ericsson
    email: magnus.westerlund@ericsson.com

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

The CONNECT-IP protocol allows endpoints to set up a tunnel for proxying IP
packets using an HTTP proxy. This can be used for various solutions that
include general-purpose packet tunnelling, such as for a point-to-point or
point-to-network VPN, or for limited forwarding of packets to specific
hosts.

Forwarded IP packets can be sent efficiently via the proxy using HTTP
Datagram support {{!I-D.ietf-masque-h3-datagram}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

In this document, we use the term "proxy" to refer to the HTTP server that
responds to the CONNECT-IP request. If there are HTTP intermediaries
(as defined in Section 2.3 of {{!RFC7230}}) between the client and the proxy,
those are referred to as "intermediaries" in this document.

# The CONNECT-IP Protocol

CONNECT-IP is defined as a protocol that can be used with the Extended CONNECT
method, where the value of the ":protocol" pseudo-header is "connect-ip".

The ":method" pseudo-header field will be set to CONNECT and the ":scheme"
pseudo-header field will be set to "https".

The ":authority" pseudo-header field contains the host and port of the proxy,
not an individual endpoint with which a connection is desired.

Variables specified via the path query parameter (sent in the ":path" pseudo-header
field) are used to determine the scope of the request, such as requesting full-tunnel
IP packet forwarding, or a specific proxied flow ({{scope}}).

Along with a request, the client can send a REGISTER_DATAGRAM_CONTEXT capsule
{{!I-D.ietf-masque-h3-datagram}} to negotiate support for sending IP packets
in HTTP Datagrams ({{packet-handling}}).

Any 2xx (Successful) response indicates that the proxy is willing to open an IP
forwarding tunnel between it and the client. Any response other than a
successful response indicates that the tunnel has not been formed.

A proxy MUST NOT send any Transfer-Encoding or Content-Length header fields in
a 2xx (Successful) response to the Extended CONNECT request. A client MUST treat
a successful response containing any Content-Length or Transfer-Encoding
header fields as malformed.

The lifetime of the forwarding tunnel is tied to the CONNECT stream. Closing
the stream (in HTTP/3 via the FIN bit on a QUIC STREAM frame, or a QUIC
RESET_STREAM frame) closes the associated forwarding tunnel.

Along with a successful response, the proxy can send capsules to assign addresses
and routes to the client ({{capsules}}). The client can also assign addresses and
routes to the proxy for network-to-network routing.

## Limiting Request Scope {#scope}

CONNECT-IP uses variables in the URL path to determine the scope of the request
for packet proxying. All variables defined here are optional, and have default
values if not included.

The defined variables are:

target:
: The variable "target" contains a hostname or IP address of a specific
host to which the client wants to proxy packets. If the "target" variable
is not specified, the client is requesting to communicate with any allowable
host. If the target is an IP address, the request will only support a single
IP version.

ipproto:
: The variable "ipproto" contains an IP protocol number, as defined in the
"Assigned Internet Protocol Numbers" IANA registry. If present, it specifies
that a client only wants to proxy a specific IP protocol for this request.
If the value is 0, or the variable is not included, the client is requesting
to use any IP protocol.

## Capsules

### ADDRESS_ASSIGN Capsule

The ADDRESS_ASSIGN capsule allows an endpoint to inform its peer that it has
assigned an IP address to it. It allows assigning a prefix which can contain
multiple addresses. This capsule uses a Capsule Type of 0xfff100. Its value
uses the following format:

~~~
ADDRESS_ASSIGN Capsule {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #addr-assign-format title="ADDRESS_ASSIGN Capsule Format"}

IP Version:

: IP Version of this address assignment. MUST be either 4 or 6.

IP Address:

: Assigned IP address. If the IP Version field has value 4, the IP Address
field SHALL have a length of 32 bits. If the IP Version field has value 6, the
IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix assigned, in bits. MUST be lesser or equal to the
length of the IP Address field, in bits.

### ADDRESS_REQUEST Capsule

The ADDRESS_REQUEST capsule allows an endpoint to request assignment of an IP
address from its peer. This capsule is not required for simple client/proxy
communication where the client only expects to receive one address from the proxy.
The capsule allows the endpoint to optionally indicate a preference for which
address it would get assigned. This capsule uses a Capsule Type of 0xfff101.
Its value uses the following format:

~~~
ADDRESS_REQUEST Capsule {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
}
~~~
{: #addr-req-format title="ADDRESS_REQUEST Capsule Format"}

IP Version:

: IP Version of this address request. MUST be either 4 or 6.

IP Address:

: Requested IP address. If the IP Version field has value 4, the IP Address
field SHALL have a length of 32 bits. If the IP Version field has value 6, the
IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix requested, in bits. MUST be lesser or equal to the
length of the IP Address field, in bits.

Upon receiving the ADDRESS_REQUEST capsule, an endpoint SHOULD assign an IP
address to its peer, and then respond with an ADDRESS_ASSIGN capsule to inform
the peer of the assignment.

### ROUTE_ADVERTISEMENT Capsule

The ROUTE_ADVERTISEMENT capsule allows an endpoint to communicate to its peer
that it is willing to route traffic to a given prefix. This indicates that the
sender has an existing route to the prefix, and notifies its peer that if the
receiver of the ROUTE_ADVERTISEMENT capsule sends IP packets for this prefix in
HTTP Datagrams, the sender of the capsule will forward them along its
preexisting route. This capsule uses a Capsule Type of 0xfff102. Its value uses
the following format:

~~~
ROUTE_ADVERTISEMENT Capsule {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
  IP Protocol (8),
}
~~~
{: #route-adv-format title="ROUTE_ADVERTISEMENT Capsule Format"}

IP Version:

: IP Version of this route advertisement. MUST be either 4 or 6.

IP Address:

: IP address of the advertised route. If the IP Version field has value 4, the
IP Address field SHALL have a length of 32 bits. If the IP Version field has
value 6, the IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: Length of the IP Prefix of the advertised route, in bits. MUST be lesser or
equal to the length of the IP Address field, in bits.

IP Protocol:

: The Internet Protocol Number for traffic that can be sent to this prefix.
If the value is 0, all protocols are allowed.

Upon receiving the ROUTE_ADVERTISEMENT capsule, an endpoint MAY start routing
IP packets in that prefix to its peer.

# Transmitting IP Packets using HTTP Datagrams {#packet-handling}

IP packets are sent using HTTP Datagrams {{!I-D.ietf-masque-h3-datagram}}.
The HTTP Datagram Payload contains a full IP packet, from the IP Version field
until the last byte of the IP Payload. In order to use HTTP Datagrams, the client
first decides whether or not to use HTTP Datagram Contexts and then register its
context ID (or lack thereof) using the corresponding registration capsule, see
{{!I-D.ietf-masque-h3-datagram}}.

When a CONNECT-IP endpoint receives an HTTP Datagram containing an IP packet,
it will parse the packet's IP header, perform any local policy checks (e.g., source
address validation), check their routing table to pick an outbound interface, and
then send the IP packet on that interface.

In the other direction, when a CONNECT-IP endpoint receives an IP packet, it
checks to see if the packet matches the routes mapped for a CONNECT-IP forwarding
tunnel, and performs the same forwarding checks as above before transmitting the
packet over HTTP Datagrams.

Note that CONNECT-IP endpoints will decrement the IP Hop Count (or TTL) upon
encapsulation but not decapsulation. In other words, the Hop Count is
decremented right before an IP packet is transmitted in an HTTP Datagram. This
prevents infinite loops in the presence of routing loops, and matches the
choices in IPsec {{?IPSEC=RFC4301}}.

Endpoints MAY implement additional filtering policies on the IP packets they
forward.

# Examples

The following example shows a point-to-network VPN setup, where a client
receives a set of local addresses, and can send to any remote server
through the proxy.

~~~
[[ From Client ]]                       [[ From Server ]]

SETTINGS
H3_DATAGRAM = 1

                                        SETTINGS
                                        SETTINGS_ENABLE_CONNECT_PROTOCOL = 1
                                        H3_DATAGRAM = 1

STREAM(44): HEADERS
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /vpn
:authority = server.example.com

STREAM(44): CAPSULE
Capsule Type = REGISTER_DATAGRAM_CONTEXT
Context ID = 0
Context Extension = {}

                                        STREAM(44): HEADERS
                                        :status = 200
                                        
                                        STREAM(44): CAPSULE
                                        Capsule Type = ADDRESS_ASSIGN
                                        IP Version = 6
                                        IP Address = 2001:db8::
                                        IP Prefix Length = 64
                                        IP Protocol = 0 // Any
                                        
                                        STREAM(44): CAPSULE
                                        Capsule Type = ROUTE_ADVERTISEMENT
                                        IP Version = 6
                                        IP Address = ::
                                        IP Prefix Length = 0
                                        IP Protocol = 0 // Any
                                        
DATAGRAM
Quarter Stream ID = 11
Context ID = 0
Payload = Encapsulated IP Packet

                                        DATAGRAM
                                        Quarter Stream ID = 11
                                        Context ID = 0
                                        Payload = Encapsulated IP Packet
~~~
{: #fig-tunnel title="VPN Tunnel Example"}

The following example shows an IP flow forwarding setup, where a client
requests to establish a forwarding tunnel to target.example.com using ICMP
(IP protocol 1), and receives a single local address and remote address
it can use for transmitting packets.

~~~
[[ From Client ]]                       [[ From Server ]]

SETTINGS
H3_DATAGRAM = 1
                                        SETTINGS
                                        SETTINGS_ENABLE_CONNECT_PROTOCOL = 1
                                        H3_DATAGRAM = 1

STREAM(52): HEADERS
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /proxy?target=target.example.com&ipproto=1
:authority = server.example.com

STREAM(52): CAPSULE
Capsule Type = REGISTER_DATAGRAM_CONTEXT
Context ID = 0
Context Extension = {}

                                        STREAM(52): HEADERS
                                        :status = 200
                                        
                                        STREAM(52): CAPSULE
                                        Capsule Type = ADDRESS_ASSIGN
                                        IP Version = 6
                                        IP Address = 2001:db8::1234:1234
                                        IP Prefix Length = 128
                                        IP Protocol = 1
                                        
                                        STREAM(52): CAPSULE
                                        Capsule Type = ROUTE_ADVERTISEMENT
                                        IP Version = 6
                                        IP Address = 2001:db8::3456
                                        IP Prefix Length = 128
                                        IP Protocol = 1
                                        
DATAGRAM
Quarter Stream ID = 11
Context ID = 0
Payload = Encapsulated IP Packet, ICMP ping

                                        DATAGRAM
                                        Quarter Stream ID = 11
                                        Context ID = 0
                                        Payload = Encapsulated IP Packet, ICMP
~~~
{: #fig-flow title="Proxied ICMP Flow Example"}

The following example shows a proxied UDP listen flow, where a client
receives can receive UDP packets via the proxy, and can send to any
UDP server through the proxy.

~~~
[[ From Client ]]                       [[ From Server ]]

SETTINGS
H3_DATAGRAM = 1

                                        SETTINGS
                                        SETTINGS_ENABLE_CONNECT_PROTOCOL = 1
                                        H3_DATAGRAM = 1

STREAM(44): HEADERS
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /proxy?ipproto=17
:authority = server.example.com

STREAM(44): CAPSULE
Capsule Type = REGISTER_DATAGRAM_CONTEXT
Context ID = 0
Context Extension = {}

                                        STREAM(44): HEADERS
                                        :status = 200
                                        
                                        STREAM(44): CAPSULE
                                        Capsule Type = ADDRESS_ASSIGN
                                        IP Version = 6
                                        IP Address = 2001:db8::1234:1234
                                        IP Prefix Length = 128
                                        IP Protocol = 17
                                        
                                        STREAM(44): CAPSULE
                                        Capsule Type = ROUTE_ADVERTISEMENT
                                        IP Version = 6
                                        IP Address = ::
                                        IP Prefix Length = 0
                                        IP Protocol = 17
                                        
...

                                        DATAGRAM
                                        Quarter Stream ID = 11
                                        Context ID = 0
                                        Payload = Encapsulated IP Packet
~~~
{: #fig-listen title="UDP Listen Flow Example"}

# Security Considerations

There are significant risks in allowing arbitrary clients to establish a tunnel
to arbitrary servers, as that could allow bad actors to send traffic and have
it attributed to the proxy. Proxies that support CONNECT-IP SHOULD restrict its
use to authenticated users. The HTTP Authorization header {{?AUTH=RFC7235}} MAY
be used to authenticate clients. More complex authentication schemes are out of
scope for this document but can be implemented using CONNECT-IP extensions.

Since CONNECT-IP endpoints can proxy IP packets send by their peer, they SHOULD
follow the guidance in {{!BCP38=RFC2827}} to help prevent denial of service
attacks.

# IANA Considerations

## CONNECT-IP HTTP Upgrade Token

This document registers an entry in the "HTTP Upgrade Tokens"
registry that was established by {{!RFC7230}}.

Value: connect-ip
Description: The CONNECT-IP Protocol
Expected Version Tokens:
References: This document

## Capsule Type Registrations {#iana-capsule-types}

This document will request IANA to add the following values to the "HTTP
Capsule Types" registry created by {{!I-D.ietf-masque-h3-datagram}}:

~~~
+----------+---------------------+---------------------+---------------+
|   Value  |        Type         |      Description    |   Reference   |
+----------+---------------------+---------------------+---------------+
| 0xfff100 |   ADDRESS_ASSIGN    | Address Assignment  | This document |
| 0xfff101 |   ADDRESS_REQUEST   | Address Request     | This document |
| 0xfff102 | ROUTE_ADVERTISEMENT | Route Advertisement | This document |
+----------+---------------------+---------------------+---------------+
~~~
      
--- back

# Acknowledgments
{:numbered="false"}

The design of this method was inspired by discussions in the MASQUE working
group around {{?I-D.ietf-masque-ip-proxy-reqs}}. The authors would like to
thank participants in those discussions for their feedback.
