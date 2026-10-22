---
title: "The ipn.arpa Zone and IPN DNS Operations"
abbrev: "IPN dot ARPA"
category: std

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
  RFC1034:
  RFC2782:
  RFC3172:

informative:
  RFC6116:
  RFC9758:
  TCPCL: RFC9174


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

# Concept of Operations {#concept-of-operations}

The Forwarding Information Base (FIB) of a BP Agent may be populated
from a variety of sources.  Typically, explicit (manual) configuration
is given to associate BP destination endpoints with the Convergence
Layer parameters required to establish communication with a responsible
receiving BP Agent.

A BP Agent may also dynamically create a FIB entry by querying for
DNS Resource Records as described in this section.

## Discovery Model

The ipn.arpa zone is an infrastructure namespace derived from IPN
identifiers.  For an IPN address with Allocator Identifier A and Node
Number N {{RFC9758}}, the derived DNS name is `N.A.ipn.arpa.`, where
N and A are each expressed as a decimal label.  For example, for
Allocator Identifier 978879 and Node Number 2, the derived DNS name is:

~~~
2.978879.ipn.arpa.
~~~

A BP Agent discovers the CL endpoint(s) for a node by querying
directly for service records beneath this derived name.  For example,
a BP Agent discovering TCPCLv4 {{TCPCL}} service for this node queries
for SRV records {{RFC2782}} at:

~~~
_dtn-bundle._tcp.2.978879.ipn.arpa.
~~~

The query name is constructed solely from the Allocator Identifier
and Node Number.  The IPN Service Number does not participate in this
lookup; all services of a node share the same discovery name.  No
preliminary PTR query is required, and this discovery procedure
requires no PTR records.

This differs from the conventional use of in-addr.arpa and ip6.arpa,
in which a PTR record maps an address to a hostname that is then used
for further lookups.  The discovery procedure in this document
publishes service records directly beneath the IPN-derived name; no
IPN-to-hostname mapping step is involved.

## Examples

The two examples in this section show alternative DNS configurations
for the same node.  In both, a BP Agent (perhaps collocated at a
ground station) has received a Bundle destined for
`ipn:978879.2.345` and uses TCPCLv4 {{TCPCL}}.  An operator publishes
either the configuration of Example A or the configuration of
Example B, not both: the SRV RRset of Example A and the CNAME of
Example B must not be published together at the same owner name
(see {{dns-constraints}}).

The records shown are zone excerpts; routine SOA and NS records are
omitted.

### Example A: Service Records in the Allocator's Zone

In this configuration, the SRV records are published directly in the
allocator's zone.

~~~ dns
; Relevant records in 978879.ipn.arpa.
$ORIGIN 978879.ipn.arpa.
_dtn-bundle._tcp.2  3600 IN SRV 0 0 4556 antenna-relay.space-agency.example.

; Relevant records in space-agency.example.
$ORIGIN space-agency.example.
antenna-relay  3600 IN A    192.0.2.1
antenna-relay  3600 IN AAAA 2001:db8::1
~~~

The BP Agent:

1. constructs the query name `_dtn-bundle._tcp.2.978879.ipn.arpa.`
   from Allocator Identifier 978879 and Node Number 2 of the
   destination `ipn:978879.2.345`;

2. queries for SRV records at `_dtn-bundle._tcp.2.978879.ipn.arpa.`,
   receiving an SRV record with port 4556 and target
   `antenna-relay.space-agency.example.`;

3. queries for A and AAAA records at
   `antenna-relay.space-agency.example.`, receiving `192.0.2.1` and
   `2001:db8::1`; and

4. attempts a TCPCLv4 connection to `[2001:db8::1]:4556` or
   `192.0.2.1:4556` and, once a session is established, transfers
   the Bundle.

### Example B: Service Lookup Redirected through a CNAME

In this configuration, the allocator's zone contains a CNAME that
redirects the service lookup to a name in the service operator's zone,
where the SRV records are published.

~~~ dns
; Relevant records in 978879.ipn.arpa.
$ORIGIN 978879.ipn.arpa.
_dtn-bundle._tcp.2  3600 IN CNAME _dtn-bundle._tcp.mars-orbiter2.space-agency.example.

; Relevant records in space-agency.example.
$ORIGIN space-agency.example.
_dtn-bundle._tcp.mars-orbiter2  3600 IN SRV 0 0 4556 antenna-relay.space-agency.example.
antenna-relay  3600 IN A    192.0.2.1
antenna-relay  3600 IN AAAA 2001:db8::1
~~~

The BP Agent starts with the same query as in Example A: it queries
for SRV records at `_dtn-bundle._tcp.2.978879.ipn.arpa.`.  Ordinary
DNS resolution follows the CNAME at that name to
`_dtn-bundle._tcp.mars-orbiter2.space-agency.example.` and returns the
SRV RRset found there, with port 4556 and target
`antenna-relay.space-agency.example.`.  There is no separate
preliminary CNAME query.  Resolution of the A and AAAA records at
`antenna-relay.space-agency.example.` and establishment of the
TCPCLv4 connection then proceed as in Example A.

This configuration divides administrative responsibility: the
allocator's zone maintains only the alias, while the service operator
maintains the SRV records in its own zone and can change them without
further changes to the allocator's zone.

### Onward Forwarding

In these examples, `antenna-relay.space-agency.example.` is configured
to accept Bundles for onward forwarding to node 2.  Establishing a
TCPCLv4 connection to that relay does not change the Bundle's
destination EID, which remains `ipn:978879.2.345`, and does not imply
that the spacecraft is currently reachable.  Forwarding the Bundle
toward node 2 is the relay's responsibility.

## DNS Constraints {#dns-constraints}

The following DNS behaviors apply to the configurations above:

* A CNAME aliases only its own owner name {{Section 3.6.2 of RFC1034}}.
  To redirect the service lookup, the CNAME is placed at the complete
  service owner name (e.g., `_dtn-bundle._tcp.2.978879.ipn.arpa.`).
  A CNAME at the bare node name (e.g., `2.978879.ipn.arpa.`) does not
  alias its descendants and so does not redirect the service lookup.

* A CNAME record and an SRV RRset cannot coexist at the same owner
  name {{Section 3.6.2 of RFC1034}}.

* The Target of the SRV record eventually obtained must have one or
  more address records and must not be an alias {{RFC2782}}.

## Other Convergence Layers

Each CL specification that uses DNS for discovery defines its own
service query names and record types (e.g., SRV or SVCB records)
beneath the IPN-derived name.  Appropriately capable BP Agents query
for those records using the same derived name.

# Operational Considerations

TODO Operational

Note that use of DNS may not be appropriate for all network
environments. Any deployments where querying DNS might not align
with reliability or performance requirements are recommended to
obtain BP FIB information by other mechanisms (out of scope of
this document).

# Security Considerations

TODO Security


# IAB Considerations

In accordance with the procedure in {{Section 3 of RFC3172}}, the IAB
is requested to approve the zone "ipn.arpa" as an infrastructure
namespace derived from IPN identifiers.  Names in this
zone are constructed from IPN Allocator Identifiers and Node Numbers
and are usable for service discovery, such as locating the CL
endpoint(s) of the BP Agent responsible for a given IPN node (see
{{concept-of-operations}}).

This follows the precedent of e164.arpa {{RFC6116}}, in which names
are derived from E.164 telephone numbers and queried directly for
service records (NAPTR).

Per {{RFC9758}}, IPN addresses have an Allocator identifier and an
allocator-specific identifier (both 32-bit unsigned integers).

# IANA Considerations

## Creation of ipn.arpa

Following approval by the IAB in accordance with
{{Section 3 of RFC3172}}, IANA is requested to add "ipn.arpa" to the
".arpa" domain.  This namespace supports DNS names derived from IPN
Allocator Identifiers and Node Numbers, as specified in this document
(see {{concept-of-operations}}).

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
