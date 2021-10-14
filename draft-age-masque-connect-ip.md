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

This document describes a method of proxying IP packets over HTTP using the
Extended CONNECT method. This protocol is similar to CONNECT-UDP, but allows
transmitting arbitrary IP packets, without being limited to just TCP like
CONNECT or UDP like CONNECT-UDP.

--- middle

# Introduction

This document describes a method of proxying IP packets over HTTP using the
Extended CONNECT method {{!RFC8441}}. This protocol is similar to CONNECT-UDP
{{?I-D.ietf-masque-connect-udp}}, but allows transmitting arbitrary IP packets,
without being limited to just TCP like CONNECT {{!I-D.ietf-httpbis-semantics}}
or UDP like CONNECT-UDP.

The Extended CONNECT protocol defined for this mechanism is "connect-ip", which
is also referred to as CONNECT-IP in this document.

The CONNECT-IP protocol allows endpoints to set up a tunnel for proxying IP
packets using an HTTP proxy. This can be used for various solutions that
include general-purpose packet tunnelling, such as for a point-to-point or
point-to-network VPN, or for limited forwarding of packets to specific hosts.

Forwarded IP packets can be sent efficiently via the proxy using HTTP Datagram
support {{!I-D.ietf-masque-h3-datagram}}.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In this document, we use the term "proxy" to refer to the HTTP server that
responds to the CONNECT-IP request. If there are HTTP intermediaries (as
defined in Section 2.3 of {{!RFC7230}}) between the client and the proxy, those
are referred to as "intermediaries" in this document.

# Configuration of Clients {#client-config}

Clients are configured to use IP Proxying over HTTP via an URI Template
{{!TEMPLATE=RFC6570}}. The URI template MAY contain two variables: "target" and
"ip_proto". Examples are shown below:

~~~
https://masque.example.org/{target}/{target_port}/
https://proxy.example.org:4443/masque?t={target}&p={ip_proto}
https://proxy.example.org:4443/masque{?target,ip_proto}
https://masque.example.org/?user=bob
~~~
{: #fig-template-examples title="URI Template Examples"}

# The CONNECT-IP Protocol

This document defines the "connect-ip" HTTP Upgrade Token. "connect-ip" uses
the Capsule Protocol as defined in {{!HTTP-DGRAM=I-D.ietf-masque-h3-datagram}}.

When sending its IP proxying request, the client SHALL perform URI template
expansion to determine the path and query of its request, see {{client-config}}.

When using HTTP/2 or HTTP/3, the following requirements apply to requests:

* The ":method" pseudo-header field SHALL be set to "CONNECT".
* The ":protocol" pseudo-header field SHALL be set to "connect-ip".
* The ":authority" pseudo-header field SHALL contain the host and port of the
  proxy, not an individual endpoint with which a connection is desired.
* The contents of the ":path" pseudo-header SHALL be determined by the URI
  template expansion, see {{client-config}}. Variables in the URI template can
  determine the scope of the request, such as requesting full-tunnel IP packet
  forwarding, or a specific proxied flow, see {{scope}}.

Along with a request, the client can send a REGISTER_DATAGRAM_CONTEXT capsule
{{HTTP-DGRAM}} to negotiate support for sending IP packets in HTTP Datagrams
({{packet-handling}}).

Any 2xx (Successful) response indicates that the proxy is willing to open an IP
forwarding tunnel between it and the client. Any response other than a
successful response indicates that the tunnel has not been formed.

A proxy MUST NOT send any Transfer-Encoding or Content-Length header fields in
a 2xx (Successful) response to the Extended CONNECT request. A client MUST
treat a successful response containing any Content-Length or Transfer-Encoding
header fields as malformed.

The lifetime of the forwarding tunnel is tied to the CONNECT stream. Closing
the stream (in HTTP/3 via the FIN bit on a QUIC STREAM frame, or a QUIC
RESET_STREAM frame) closes the associated forwarding tunnel.

Along with a successful response, the proxy can send capsules to assign
addresses and routes to the client ({{capsules}}). The client can also assign
addresses and routes to the proxy for network-to-network routing.

## Limiting Request Scope {#scope}

Unlike CONNECT-UDP requests, which require specifying a target host, CONNECT-IP
requests can allow endpoints to send arbitrary IP packets to any host. The
client can choose to restrict a given request to a specific host or IP protocol
by adding parameters to its request. When the server knows that a request is
scoped to a target host or protocol, it can leverage this information to
optimize its resource allocation; for example, the server can assign the same
public IP address to two CONNECT-IP requests that are scoped to different hosts
and/or different protocols.

CONNECT-IP uses URI template variables ({{client-config}}) to determine the
scope of the request for packet proxying. All variables defined here are
optional, and have default values if not included.

The defined variables are:

target:

: The variable "target" contains a hostname or IP address of a specific host to
which the client wants to proxy packets. If the "target" variable is not
specified, the client is requesting to communicate with any allowable host. If
the target is an IP address, the request will only support a single IP version.
If the target is a hostname, the server is expected to perform DNS resolution
to determine which route(s) to advertise to the client.

ipproto:

: The variable "ipproto" contains an IP protocol number, as defined in the
"Assigned Internet Protocol Numbers" IANA registry. If present, it specifies
that a client only wants to proxy a specific IP protocol for this request. If
the value is 0, or the variable is not included, the client is requesting to
use any IP protocol.

## Capsules

This document defines multiple new capsule types that allow endpoints to
exchange IP configuration information. Both endpoints MAY send any number of
these new capsules.

### ADDRESS_ASSIGN Capsule

The ADDRESS_ASSIGN capsule allows an endpoint to inform its peer that it has
assigned an IP address to it. The ADDRESS_ASSIGN capsule allows assigning a
prefix which can contain multiple addresses. Any of these addresses can be used
as the source address on IP packets originated by the receiver of this
capsule. This capsule uses a Capsule Type of 0xfff100. Its value uses the
following format:

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

: The number of bits in the IP Address that are used to define the prefix that
is being assigned. This MUST be less than or equal to the length of the IP
Address field, in bits. If the prefix length is equal to the length of the IP
Address, the receiver of this capsule is only allowed to send packets from a
single source address. If the prefix length is less than the length of the IP
address, the receiver of this capsule is allowed to send packets from any source
address that falls within the prefix.

If an endpoint receives multiple ADDRESS_ASSIGN capsules, all of the assigned
addresses or prefixes can be used. For example, multiple ADDRESS_ASSIGN
capsules are necessary to assign both IPv4 and IPv6 addresses.

### ADDRESS_REQUEST Capsule

The ADDRESS_REQUEST capsule allows an endpoint to request assignment of an IP
address from its peer. This capsule is not required for simple client/proxy
communication where the client only expects to receive one address from the
proxy. The capsule allows the endpoint to optionally indicate a preference for
which address it would get assigned. This capsule uses a Capsule Type of
0xfff101. Its value uses the following format:

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
preexisting route. Any of these addresses can be used as the destination address
on IP packets originated by the receiver of this capsule. This capsule uses a
Capsule Type of 0xfff102. Its value uses the following format:

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

: The number of bits in the IP Address that are used to define the prefix of
the advertised route. This MUST be lesser or equal to the length of the IP
Address field, in bits. If the prefix length is equal to the length of the IP
Address, this route only allows sending packets to a single destination
address. If the prefix length is less than the length of the IP address, this
route allows sending packets to any destination address that falls within the
prefix.

IP Protocol:

: The Internet Protocol Number for traffic that can be sent to this prefix. If
the value is 0, all protocols are allowed.

Upon receiving the ROUTE_ADVERTISEMENT capsule, an endpoint MAY start routing
IP packets in that prefix to its peer.

If an endpoint receives multiple ROUTE_ADVERTISEMENT capsules, all of the
advertised routes can be used. For example, multiple ROUTE_ADVERTISEMENT
capsules are necessary to provide routing to both IPv4 and IPv6 hosts. Routes
are removed using ROUTE_WITHDRAWAL capsules.

### ROUTE_WITHDRAWAL Capsule

The ROUTE_WITHDRAWAL capsule allows an endpoint to communicate to its peer that
it is not willing to route traffic to a given prefix. This capsule uses a
Capsule Type of 0xfff103. Its value uses the following format:

~~~
ROUTE_WITHDRAWAL Capsule {
  IP Version (8),
  IP Address (32..128),
  IP Prefix Length (8),
  IP Protocol (8),
}
~~~
{: #route-withdraw-format title="ROUTE_WITHDRAWAL Capsule Format"}

IP Version:

: IP Version of this route withdrawal. MUST be either 4 or 6.

IP Address:

: IP address of the withdrawn route. If the IP Version field has value 4, the
IP Address field SHALL have a length of 32 bits. If the IP Version field has
value 6, the IP Address field SHALL have a length of 128 bits.

IP Prefix Length:

: The number of bits in the IP Address that are used to define the prefix of
the withdrawn route. This MUST be lesser or equal to the length of the IP
Address field, in bits.

IP Protocol:

: The Internet Protocol Number for traffic for this route. If the value is 0,
all protocols are withdrawn for this prefix.

Upon receiving the ROUTE_WITHDRAWAL capsule, an endpoint MUST stop routing IP
packets in that prefix to its peer. Note that this capsule can be reordered
with DATAGRAM frames, and therefore an endpoint that receives packets for
routes it has rejected MUST NOT treat that as an error.

ROUTE_ADVERTISEMENT and ROUTE_WITHDRAWAL capsules are applied in order of
receipt: if a prefix is covered by multiple received ROUTE_ADVERTISEMENT and/or
ROUTE_WITHDRAWAL capsules, only the last received capsule applies as it
supersedes prior ROUTE_ADVERTISEMENT and ROUTE_WITHDRAWAL capsules for this
prefix.

# Transmitting IP Packets using HTTP Datagrams {#packet-handling}

IP packets are sent using HTTP Datagrams {{!I-D.ietf-masque-h3-datagram}}. The
HTTP Datagram Payload contains a full IP packet, from the IP Version field
until the last byte of the IP Payload. In order to use HTTP Datagrams, the
client first decides whether or not to use HTTP Datagram Contexts and then
register its context ID (or lack thereof) using the corresponding registration
capsule, see {{!I-D.ietf-masque-h3-datagram}}.

When a CONNECT-IP endpoint receives an HTTP Datagram containing an IP packet,
it will parse the packet's IP header, perform any local policy checks (e.g.,
source address validation), check their routing table to pick an outbound
interface, and then send the IP packet on that interface.

In the other direction, when a CONNECT-IP endpoint receives an IP packet, it
checks to see if the packet matches the routes mapped for a CONNECT-IP
forwarding tunnel, and performs the same forwarding checks as above before
transmitting the packet over HTTP Datagrams.

Note that CONNECT-IP endpoints will decrement the IP Hop Count (or TTL) upon
encapsulation but not decapsulation. In other words, the Hop Count is
decremented right before an IP packet is transmitted in an HTTP Datagram. This
prevents infinite loops in the presence of routing loops, and matches the
choices in IPsec {{?IPSEC=RFC4301}}.

Endpoints MAY implement additional filtering policies on the IP packets they
forward.

# Examples

CONNECT-IP enables many different use cases that can benefit from IP packet
proxying and tunnelling. These examples are provided to help illustrate some of
the ways in which CONNECT-IP can be used.

## Remote Access VPN

The following example shows a point-to-network VPN setup, where a client
receives a set of local addresses, and can send to any remote server through
the proxy. Such VPN setups can be either full-tunnel or split-tunnel.

~~~

+--------+ IP A         IP B +--------+              +---> IP D
|        |-------------------|        | IP C         |
| Client | IP Subnet C <-> * | Server |--------------+---> IP E
|        |-------------------|        |              |
+--------+                   +--------+              +---> IP ...

~~~
{: #diagram-tunnel title="VPN Tunnel Setup"}

In this case, the client does not specify any scope in its request. The server
assigns the client an IPv6 address prefix to the client (2001:db8::/64) and a
full-tunnel route of all IPv6 addresses (::/0). The client can then send to any
IPv6 host using a source address in its assigned prefix.

~~~
[[ From Client ]]             [[ From Server ]]

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
{: #fig-full-tunnel title="VPN Full-Tunnel Example"}

A setup for a split-tunnel VPN (the case where the client can only access a
specific set of private subnets) is quite similar. In this case, the advertised
route is restricted to 2001:db8::/32, rather than ::/0.

~~~
[[ From Client ]]             [[ From Server ]]

                              STREAM(44): CAPSULE
                              Capsule Type = ADDRESS_ASSIGN
                              IP Version = 6
                              IP Address = 2001:db8:1:1::
                              IP Prefix Length = 64

                              STREAM(44): CAPSULE
                              Capsule Type = ROUTE_ADVERTISEMENT
                              IP Version = 6
                              IP Address = 2001:db8::
                              IP Prefix Length = 32
                              IP Protocol = 0 // Any
~~~
{: #fig-split-tunnel title="VPN Split-Tunnel Capsule Example"}

## IP Flow Forwarding

The following example shows an IP flow forwarding setup, where a client
requests to establish a forwarding tunnel to target.example.com using SCTP (IP
protocol 132), and receives a single local address and remote address it can
use for transmitting packets. A similar approach could be used for any other IP
protocol that isn't easily proxied with existing HTTP methods, such as ICMP,
ESP, etc.

~~~

+--------+ IP A         IP B +--------+
|        |-------------------|        | IP C
| Client |    IP C <-> D     | Server |---------> IP D
|        |-------------------|        |
+--------+                   +--------+

~~~
{: #diagram-flow title="Proxied Flow Setup"}

In this case, the client specfies both a target hostname and an IP protocol
number in the scope of its request, indicating that it only needs to
communicate with a single host. The proxy server is able to perform DNS
resolution on behalf of the client and allocate a specific outbound socket for
the client instead of allocating an entire IP address to the client. In this
regard, the request is similar to a traditional CONNECT proxy request.

The server assigns a single IPv6 address to the client (2001:db8::1234:1234)
and a route to a single IPv6 host (2001:db8::3456), scoped to SCTP. The client
can send and recieve SCTP IP packets to the remote host.

~~~
[[ From Client ]]             [[ From Server ]]

SETTINGS
H3_DATAGRAM = 1
                              SETTINGS
                              SETTINGS_ENABLE_CONNECT_PROTOCOL = 1
                              H3_DATAGRAM = 1

STREAM(52): HEADERS
:method = CONNECT
:protocol = connect-ip
:scheme = https
:path = /proxy?target=target.example.com&ipproto=132
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

                              STREAM(52): CAPSULE
                              Capsule Type = ROUTE_ADVERTISEMENT
                              IP Version = 6
                              IP Address = 2001:db8::3456
                              IP Prefix Length = 128
                              IP Protocol = 132

DATAGRAM
Quarter Stream ID = 11
Context ID = 0
Payload = Encapsulated SCTP/IP Packet

                              DATAGRAM
                              Quarter Stream ID = 11
                              Context ID = 0
                              Payload = Encapsulated SCTP/IP Packet
~~~
{: #fig-flow title="Proxied SCTP Flow Example"}

## Proxied Connection Racing

The following example shows a setup where a client is proxying UDP packets
through a CONNECT-IP proxy in order to control connection establishement racing
through a proxy, as defined in Happy Eyeballs {{?RFC8305}}. This example is a
variant of the proxied flow, but highlights how IP-level proxying can enable
new capabilities even for TCP and UDP.

~~~

+--------+ IP A         IP B +--------+ IP C
|        |-------------------|        |<------------> IP E
| Client |  IP C<->E, D<->F  | Server |
|        |-------------------|        |<------------> IP F
+--------+                   +--------+ IP D

~~~
{: #diagram-racing title="Proxied Connection Racing Setup"}

As with proxied flows, the client specfies both a target hostname and an IP
protocol number in the scope of its request. When the proxy server performs DNS
resolution on behalf of the client, it can send the various remote address
options to the client as separate routes. It can also ensure that the client
has both IPv4 and IPv6 addresses assigned.

The server assigns the client both an IPv4 address (192.0.2.3) and an IPv6
address (2001:db8::1234:1234) to the client, as well as an IPv4 route
(198.51.100.2) and an IPv6 route (2001:db8::3456), which represent the resolved
addresses of the target hostname, scoped to UDP. The client can send and
recieve UDP IP packets to the either of the server addresses to enable Happy
Eyeballs through the proxy.

~~~
[[ From Client ]]             [[ From Server ]]

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
                              IP Version = 4
                              IP Address = 192.0.2.3
                              IP Prefix Length = 32

                              STREAM(44): CAPSULE
                              Capsule Type = ADDRESS_ASSIGN
                              IP Version = 6
                              IP Address = 2001:db8::1234:1234
                              IP Prefix Length = 128

                              STREAM(44): CAPSULE
                              Capsule Type = ROUTE_ADVERTISEMENT
                              IP Version = 4
                              IP Address = 198.51.100.2
                              IP Prefix Length = 32
                              IP Protocol = 17

                              STREAM(44): CAPSULE
                              Capsule Type = ROUTE_ADVERTISEMENT
                              IP Version = 6
                              IP Address = 2001:db8::3456
                              IP Prefix Length = 128
                              IP Protocol = 17
...

DATAGRAM
Quarter Stream ID = 11
Context ID = 0
Payload = Encapsulated IPv6 Packet

DATAGRAM
Quarter Stream ID = 11
Context ID = 0
Payload = Encapsulated IPv4 Packet

~~~
{: #fig-listen title="Proxied Connection Racing Example"}

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

This document registers an entry in the "HTTP Upgrade Tokens" registry that was
established by {{!RFC7230}}.

Value:

: connect-ip

Description:

: The CONNECT-IP Protocol

Expected Version Tokens:

: None

References:

: This document

## Capsule Type Registrations {#iana-capsule-types}

This document will request IANA to add the following values to the "HTTP
Capsule Types" registry created by {{!I-D.ietf-masque-h3-datagram}}:

|  Value   |        Type         |     Description     |   Reference   |
|:---------|---------------------|:--------------------|:--------------|
| 0xfff100 |   ADDRESS_ASSIGN    | Address Assignment  | This Document |
| 0xfff101 |   ADDRESS_REQUEST   | Address Request     | This Document |
| 0xfff102 | ROUTE_ADVERTISEMENT | Route Advertisement | This Document |
| 0xfff103 |  ROUTE_WITHDRAWAL   | Route Withdrawal    | This Document |
{: #iana-capsules-table title="New Capsules"}

--- back

# Acknowledgments
{:numbered="false"}

The design of this method was inspired by discussions in the MASQUE working
group around {{?I-D.ietf-masque-ip-proxy-reqs}}. The authors would like to
thank participants in those discussions for their feedback.
