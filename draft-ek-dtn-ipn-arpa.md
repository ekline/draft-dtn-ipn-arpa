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
  RFC6672:
  RFC9758:

informative:
  RFC1918:
  RFC6116:
  RFC6303:
  RFC8375:
  TCPCL: RFC9174
  BIND-ARM:
    title: "BIND 9 Administrator Reference Manual"
    author:
      - org: Internet Systems Consortium
    date: false
    target: https://bind9.readthedocs.io/en/latest/chapter3.html


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

This document uses the following terms as defined in {{RFC9758}}:

Allocator Identifier:
: A 32-bit unsigned integer identifying the Allocator that assigned a
  Node Number ({{Section 3.2 of RFC9758}}).

Node Number:
: A 32-bit unsigned integer identifying a node in the context of an
  Allocator ({{Section 3.3 of RFC9758}}).

Fully Qualified Node Number (FQNN):
: The combination of an Allocator Identifier and a Node Number, which
  uniquely identifies a node ({{Section 3.3.1 of RFC9758}}).  As in
  {{RFC9758}}, an FQNN is written as (Allocator Identifier, Node
  Number); for example, (978879,2).

Service Number:
: An unsigned integer identifying a service on a node
  ({{Section 3.5 of RFC9758}}).

ipn URI:
: A URI using the 'ipn' URI scheme.  An ipn URI comprises an Allocator
  Identifier, a Node Number, and a Service Number; that is, an FQNN
  plus a Service Number ({{Section 3 of RFC9758}}).  An ipn URI used
  as a Bundle Protocol Endpoint Identifier (EID) is termed an "ipn
  EID" ({{Section 5 of RFC9758}}).

This document defines the following terms:

DNS base name:
: The DNS name beneath ipn.arpa that represents an FQNN, constructed
  as specified in {{discovery-model}}.

CL-specific query name:
: A DNS name, defined by a CL specification relative to a DNS base
  name, at which a BP Agent queries for the records needed to reach
  the node using that CL.  For example, the TCPCLv4 query name is
  formed by prepending `_dtn-bundle._tcp` to the DNS base name.

# Concept of Operations {#concept-of-operations}

The Forwarding Information Base (FIB) of a BP Agent may be populated
from a variety of sources.  Typically, explicit (manual) configuration
is given to associate BP destination endpoints with the Convergence
Layer parameters required to establish communication with a responsible
receiving BP Agent.

A BP Agent may also dynamically create a FIB entry by querying for
DNS Resource Records as described in this section.

## Discovery Model {#discovery-model}

The ipn.arpa zone is an infrastructure namespace derived from IPN
identifiers.  Discovery is keyed on the FQNN of the destination node.

Before constructing a DNS name, a BP Agent MUST parse and interpret
the destination ipn URI or ipn EID according to {{RFC9758}}, yielding
numeric values for its Allocator Identifier, Node Number, and Service
Number.  This includes the text syntax rules of
{{Section 4 of RFC9758}} and the CBOR decoding rules of
{{Section 6.2 of RFC9758}}.  In particular, an Allocator Identifier
omitted from the text representation means zero, the Default
Allocator ({{Section 3.2.2 of RFC9758}}).  An omitted Allocator
Identifier does not cause the allocator label to be omitted from the
DNS name.

For Allocator Identifier A and Node Number N, the DNS base name MUST
be:

~~~
N.A.ipn.arpa.
~~~

Each of N and A MUST be represented as a single label containing its
unsigned decimal value, using ASCII digits without leading zeroes.
The value zero is represented by the single digit "0".  The labels
are not padded to a fixed width and are not split into individual
digits.

The Service Number does not participate in construction.  All ipn
URIs that share an FQNN map to the same DNS base name.

| ipn URI | DNS base name |
|---|---|
| `ipn:978879.2.345` | `2.978879.ipn.arpa.` |
| `ipn:978879.2.0` | `2.978879.ipn.arpa.` |
| `ipn:16384.345` | `16384.0.ipn.arpa.` |
| `ipn:0.16384.345` | `16384.0.ipn.arpa.` |
{: title="Example DNS Base Names"}

Because the DNS base name is constructed from the parsed numeric
components rather than from the text of the URI, equivalent
representations of the same FQNN produce the same DNS base name.  For
example, `ipn:16384.345` and `ipn:0.16384.345` both identify the FQNN
(0,16384).  Likewise, the two-element and three-element CBOR encodings
of an ipn EID ({{Section 6.4 of RFC9758}}) produce the same DNS base
name.

The DNS base name is distinct from the CL-specific query name at
which a BP Agent queries for records.  For example, a BP Agent
discovering TCPCLv4 {{TCPCL}} service for the FQNN (978879,2)
prepends `_dtn-bundle._tcp` to the DNS base name and queries for SRV
records {{RFC2782}} at:

~~~
_dtn-bundle._tcp.2.978879.ipn.arpa.
~~~

No preliminary PTR query is required, and this discovery procedure
requires no PTR records.

This differs from the conventional use of in-addr.arpa and ip6.arpa,
in which a PTR record maps an address to a hostname that is then used
for further lookups.  The discovery procedure in this document
publishes service records directly beneath the DNS base name; no
IPN-to-hostname mapping step is involved.

The Null ipn URI ({{Section 3.1 of RFC9758}}) and LocalNode ipn URIs
({{Section 3.4.2 of RFC9758}}) are recognized after the interpretation
described above.  For example, an ipn URI with a zero Allocator
Identifier, a zero Node Number, and a non-zero Service Number is
treated as the Null ipn URI ({{Section 3.4.1 of RFC9758}}), and
`ipn:!.7` is a LocalNode ipn URI.  Neither identifies a remote
destination for this procedure: the Null endpoint has no members, and
a bundle destined for a LocalNode ipn EID is not allowed to leave the
local node ({{Section 5.4 of RFC9758}}).  A BP Agent MUST NOT perform
DNS discovery for the Null ipn URI or for a LocalNode ipn URI.  In
addition,
{{Section 5.4 of RFC9758}} prohibits publishing LocalNode ipn EIDs in
any node identification directory, such as a DNS registration.

These cases are identified by the complete FQNN: (0,0) for the Null
ipn URI and (0,4294967295) for LocalNode ipn URIs.  This document
does not reserve Node Number 0 or Node Number 4294967295 under any
other Allocator Identifier.  For example, the FQNNs (978879,0) and
(978879,4294967295) are processed like any other FQNN.

TODO: Specify the treatment of Private Use identities
({{Section 3.4.3 of RFC9758}} and {{Section 5.5 of RFC9758}}),
including resolution within a locally scoped namespace and prevention
of inappropriate queries for such names in the public DNS.  This
policy will be addressed separately.

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
relative to the DNS base name.  Appropriately capable BP Agents query
for those records using the same DNS base name.

# Operational Considerations

TODO Operational

Note that use of DNS may not be appropriate for all network
environments. Any deployments where querying DNS might not align
with reliability or performance requirements are recommended to
obtain BP FIB information by other mechanisms (out of scope of
this document).

## Managing Allocator Ranges {#managing-allocator-ranges}

An Allocator may hold a range of Allocator Identifiers
({{Section 3.2.1 of RFC9758}}).  Numeric allocation ranges do not
automatically correspond to DNS subtrees.  With the decimal-label
layout specified in {{discovery-model}}, each Allocator Identifier A
has its own name, `A.ipn.arpa.`, directly beneath ipn.arpa.  The
identifiers in a range are siblings, not descendants of a common name.

An operator responsible for several Allocator Identifiers can serve an
individually delegated zone for each one.  Alternatively, DNAME
records {{RFC6672}} can redirect each allocator subtree into a common
operator-managed zone.

The following example uses Allocator Identifiers 974848 and 974849,
from the example range in {{Section 9.1 of RFC9758}}.  It assumes that
the organization is authorized to administer both Allocator
Identifiers.  The records are zone excerpts; routine SOA and NS
records are omitted.

~~~ dns
; Relevant records in ipn.arpa.
$ORIGIN ipn.arpa.
974848 3600 IN DNAME 974848.allocators.operator.example.
974849 3600 IN DNAME 974849.allocators.operator.example.

; Relevant records in the single zone operator.example.
$ORIGIN operator.example.
_dtn-bundle._tcp.2.974848.allocators 3600 IN SRV 0 0 4556 relay-a.operator.example.
_dtn-bundle._tcp.2.974849.allocators 3600 IN SRV 0 0 4556 relay-b.operator.example.
relay-a 3600 IN A    192.0.2.1
relay-a 3600 IN AAAA 2001:db8::1
relay-b 3600 IN A    192.0.2.2
relay-b 3600 IN AAAA 2001:db8::2
~~~

A BP Agent discovering TCPCLv4 service for `ipn:974848.2.345`
constructs its initial query from the standard DNS base name, exactly
as in {{discovery-model}}, and queries for SRV records at:

~~~
_dtn-bundle._tcp.2.974848.ipn.arpa.
~~~

The DNAME at `974848.ipn.arpa.` redirects this query to:

~~~
_dtn-bundle._tcp.2.974848.allocators.operator.example.
~~~

There the BP Agent finds an SRV record with port 4556 and target
`relay-a.operator.example.`, whose addresses are 192.0.2.1 and
2001:db8::1.  Ordinary DNS processing performs the redirection
({{Section 3 of RFC6672}}).  The BP Agent does not construct any name
in the operator's zone itself.

The example retains the Allocator Identifier in each target subtree.
As a result, node 2 under Allocator 974848
(`2.974848.allocators.operator.example.`) remains distinct from node 2
under Allocator 974849 (`2.974849.allocators.operator.example.`).
Pointing both DNAMEs at the same target would merge their
corresponding lookup names: queries for the FQNNs (974848,2) and
(974849,2) would then reach the same records.  Distinct target
subtrees need to be preserved whenever the allocator namespaces must
remain distinct.  Retaining the decimal allocator label in the target
is a convenient convention, not a DNS requirement.

One operator-managed zone can contain all of these subtrees.  One
DNAME is still required in ipn.arpa for each redirected Allocator
Identifier.  The technique reduces the number of separately
administered child zones, not the number of allocator entries in the
parent.

The following constraints apply:

* A DNAME redirects the descendants of its owner name, not the owner
  name itself ({{Section 2.3 of RFC6672}}).

* Records cannot be published beneath the DNAME owner in the source
  zone ({{Section 2.4 of RFC6672}}).  Once `974848.ipn.arpa.` owns a
  DNAME, all data for that allocator is published in the target
  subtree.

* DNAME redirection is distinct from NS delegation.  An NS delegation
  creates a zone cut and transfers authority for the child zone to
  its own servers.  A DNAME instead rewrites names beneath its owner
  into the target namespace.  A parent-side DNAME cannot coexist with
  a delegation NS RRset at the same owner name
  ({{Section 2.3 of RFC6672}}).  An allocator zone that is already
  delegated can instead publish a DNAME at its own apex, keeping its
  SOA and NS records.  That arrangement retains the separate
  child-zone administration.

See {{RFC6672}} for processing details and DNSSEC considerations
({{Section 5.3 of RFC6672}}).

The same technique can redirect service subtrees beneath individual
node names in 0.ipn.arpa (for example, with a DNAME at
`N.0.ipn.arpa.`), subject to authorization by the administrator of
that namespace.  Because 0.ipn.arpa is served only locally (see
{{creation-of-ipn-arpa}}), such DNAMEs exist only in locally served
instances of that zone.

### Provisioning Tools

Some DNS implementations provide implementation-specific facilities,
such as the `$GENERATE` directive of BIND {{BIND-ARM}}, as
conveniences for producing repetitive records.  For example, the two
parent-side DNAME records above can be expressed in BIND syntax as:

~~~ dns
$ORIGIN ipn.arpa.
$GENERATE 974848-974849 $ 3600 IN DNAME $.allocators.operator.example.
~~~

Such a directive expands into ordinary records in the zone.  It
introduces no numeric-range matching in DNS, and resolvers need no
support for it.  Similar tooling can generate NS records or other
repetitive data.  Use of these facilities is optional.

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

Per {{Section 3.3.1 of RFC9758}}, the FQNN of a node comprises a
32-bit Allocator Identifier and a 32-bit Node Number.  Those two
components determine the DNS base name (see {{discovery-model}}).

# IANA Considerations

## Creation of ipn.arpa {#creation-of-ipn-arpa}

Following approval by the IAB in accordance with
{{Section 3 of RFC3172}}, IANA is requested to add "ipn.arpa" to the
".arpa" domain.  This namespace supports DNS names derived from IPN
Allocator Identifiers and Node Numbers, as specified in this document
(see {{concept-of-operations}}).

"0.ipn.arpa" corresponds to Allocator Identifier zero, the Default
Allocator ({{Section 3.2.2 of RFC9758}}).  It is treated like the
IN-ADDR.ARPA zones for the private address space of {{RFC1918}}
({{Section 4.1 of RFC6303}}): it has only local significance.  A
"0.ipn.arpa" zone MAY exist in local DNS resolvers or servers that
serve it authoritatively as a locally served zone {{RFC6303}}, but a
populated "0.ipn.arpa" zone MUST NOT ever be published in the global
DNS.

TODO: Specify how "0.ipn.arpa" is represented in the global DNS so
that locally served answers can still be validated.  If ipn.arpa is
signed, an authenticated denial of existence for "0.ipn.arpa" would
cause validating resolvers to reject locally served data; locally
served zones need an insecure delegation or must be within an insecure
zone ({{Section 7 of RFC6303}}).  Compare the unsigned delegation of
home.arpa to black-hole servers ({{Section 7 of RFC8375}}).  Also
consider adding "0.ipn.arpa" to the IANA "Locally-Served DNS Zones"
registry ({{Section 6 of RFC6303}}).

## 'ipn' Allocator Registry List of Nameservers

IANA is requested to augment its registration procedures for entries
in the 'ipn' Scheme URI Allocator Identifiers Registry with a column
for "Authoritative Nameservers".  Requesters of an Allocator or
Allocator range MAY provide the IP addresses of DNS nameservers that
will be considered authoritative for the .ipn.arpa zone for each requested
Allocator.

TODO: Discuss the procedures for delegating the corresponding child
zones of ipn.arpa to the allocator agency.

TODO: In the forthcoming administration-policy pass, address the
authorization, creation, and maintenance of DNAME redirections in
ipn.arpa (see {{managing-allocator-ranges}}) alongside NS delegation
requests.

## Guidance to Designate Experts

TBD: things like make sure there are at least two nameserver IP addresses
and both IPv4 and IPv6 addresses are given, etc.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
