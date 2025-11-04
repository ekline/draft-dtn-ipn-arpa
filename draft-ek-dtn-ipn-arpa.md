---
title: "The ipn.arpa Zone and IPN DNS Operations"
abbrev: "IPN dot ARPA"
category: info

docname: draft-ek-dtn-ipn-arpa-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: ""
workgroup: "Delay/Disruption Tolerant Networking"
keyword:
 - DTN
 - BP
 - IPN
 - DNS
venue:
  group: "Delay/Disruption Tolerant Networking"
  type: ""
  mail: "dtn@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
  github: "ekline/draft-dtn-ipn-arpa"
  latest: "https://ekline.github.io/draft-dtn-ipn-arpa/draft-ek-dtn-ipn-arpa.html"

author:
 -
    fullname: Erik Kline
    organization: Aalyria Technologies, LLC
    email: ek.ietf@gmail.com

normative:

informative:
  RFC9758:
  TCPCL: RFC9174
  UDPCL: RFC7122


--- abstract

This document requests a DNS parent for IPN addresses, discusses
the registration procedures and management of the DNS zone, as
well as some operational recommendations.  This document specifies that
IPN addresses may have a DNS representation of the form 1.978879.ipn.arpa,
for IPN node 1 under IPN Allocator 978879.

This document also describes how this DNS structure can be useful in
locating the Bundle Protocol (BP) Convergence Layer (CL) endpoint(s) of
the BP Agent responsible for a given IPN address.

--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

TODO

# Concept of Operations

The Forwarding Information Base (FIB) of a BP Agent may be populated
from a variety of sources.  Typically, explicit (manual) configuration
is given to associate BP destination endpoints with the Convergence
Layer parameters required to establish communication with a responsible
receiving BP Agent.

A BP Agent may also dynamically create a FIB entry by querying for
DNS Resource Records as follows.

Consider a space agency with a registered IPN Allocator ID of 978879
that published the following two DNS zones:

~~~ zone
; zone 978879.ipn.arpa.
@   IN SOA ns1.space-agency.example ...
1   IN PTR lunar-orbiter1.space-agency.example.
2   IN PTR mars-orbiter2.space-agency.example.
500 IN PTR research-department.space-agency.example.


; zone space-agency.example.
@ IN SOA ns1.space-agency.example ...

antenna-relay IN A 192.0.2.1
antenna-relay IN AAAA 2001:db8::1

lunar-orbiter1 IN TXT "{'norad-id': '81111'}}"
_dtn-bundle._tcp.2 IN SRV 1 1 4556 antenna-relay.space-agency.example.

mars-orbiter2 IN TXT "{'norad-id': '82222'}}"
_dtn-bundle._udp.2 IN SRV 1 1 4556 antenna-relay.space-agency.example.

cloud-service IN A 192.0.2.5
cloud-service IN AAAA 2001:db8::5

_dtn-bundle._tcp.research-department IN SRV 1 1 4556 cloud-service.space-agency.example.
_dtn-bundle._udp.research-department IN SRV 1 1 4556 cloud-service.space-agency.example.
~~~

A BP Agent, perhaps collocated at a landing station, which had received
a Bundle destined for `ipn:978879.500.345` could determine the proper
TCP and UDP CL IP addresses and ports by:

* querying for a PTR record associated with `500.978879.ipn.arpa`

* (receiving `research-department.space-agency.example`)

* querying for an SRV record associated with `_dtn-bundle._tcp.research-department.space-agency.example` (ditto for `._udp`)

* (receiving port 4556 and hostname `cloud-service.space-agency.example.`)

* querying for A and AAAA records for `cloud-service.space-agency.example`.

* (receiving `192.0.2.5` and `2001:db8::5`)

The BP Agent at the landing station could then attempt to initiated a
{{TCPCL}} or {{UDPCL}} connection to `[2001:db8::5]:4556`,
for example, and attempt to deliver the given Bundle.

Similarly, a researcher attempting to send a Bundle payload to
`mars-orbiter2.space-agency.example` could use a BP Agent capable
of querying DNS and learning about the {{UDPCL}} endpoint at
`192.0.2.1:4556`. Once the Bundle had been forwarded to
`antenna-relay.space-agency.example`,
it would be the `antenna-relay`'s responsibility to forward it to the
spacecraft.

# Security Considerations

TODO Security


# IAB Considerations

The IAB is requested to approve the zone "ipn.arpa" as parent for IPN
addresses, similar to in-addr.arpa and ip6.arpa "reverse DNS" for IPv4
and IPv6 addresses (respectively).

Per {{RFC9758}}, IPN addresses have an Allocator identifier and an
allocator-specific identifier (both 32-bit unsigned integers).

# IANA Considerations

## 'ipn' Allocator Registry List of Nameservers

IANA is requested to augment its registration procedures for entries
in the 'ipn' Scheme URI Allocator Identifiers Registry with a column
for "Authoritative Nameservers".  Requesters of an Allocator or
Allocator range MAY provide the IP addresses of DNS nameservers that
will be considered authoritative for the .ipn.arpa zone for each requested
Allocator.

## Guidance to Designate Experts

TBD: things like make sure there are at least two nameserver IP addresses
and both IPv4 and IPv6 addresses are given, etc.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
