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
  RFC4035:
  RFC6672:
  RFC9364:
  RFC9758:

informative:
  RFC1918:
  RFC6116:
  RFC6303:
  RFC7116:
  RFC8126:
  RFC8375:
  TCPCL: RFC9174
  BIND-ARM:
    title: "BIND 9 Administrator Reference Manual"
    author:
      - org: Internet Systems Consortium
    date: false
    target: https://bind9.readthedocs.io/en/latest/chapter3.html
  IANA-NS-REQS:
    title: "Technical requirements for authoritative name servers"
    author:
      - org: IANA
    date: false
    target: https://www.iana.org/help/nameserver-requirements


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

Names for Allocator Identifier zero, such as `16384.0.ipn.arpa.`, and
for Experimental Use Allocator Identifiers are constructed by the same
rule.  They are usable in locally served instances, but they are not
published in the global DNS (see {{publication-restrictions}}).

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

Private Use Node Numbers ({{Section 3.4.3 of RFC9758}} and
{{Section 5.5 of RFC9758}}) exist only under Allocator Identifier zero,
so their DNS base names fall beneath 0.ipn.arpa.  Their public DNS
publication is therefore excluded, and they can be resolved only in
locally served instances (see {{allocator-zero}} and
{{publication-restrictions}}).

TODO: Address, separately from the publication policy above, the
remaining questions for locally served namespaces: preventing queries
for such names from leaking into the public DNS, the negative answers
that the public DNS returns for them, and DNSSEC validation of locally
served data (see {{local-resolution-todo}}).

## Examples {#examples}

The examples in this section describe one deployment: a fictional
space agency that offers a shared public ingress service for its
destinations.  Other operators can use different arrangements, such
as a distinct relay for each node or direct publication of a node's
own endpoints; nothing in this document requires the architecture
shown here.

The agency administers Allocator Identifier 978879.  Its example
destinations are:

| FQNN | Destination | DNS base name |
|---|---|---|
| (978879,1) | mission operations center | `1.978879.ipn.arpa.` |
| (978879,2) | mars-orbiter | `2.978879.ipn.arpa.` |
| (978879,3) | lunar-lander | `3.978879.ipn.arpa.` |
{: title="Example Destinations"}

The destination names are descriptive labels chosen independently of
the Node Numbers.

All three destinations use the same public ingress service, provided
by two relay BP Agents, `relay-a.space-agency.example.` and
`relay-b.space-agency.example.`.  The relays are separate BP nodes
with their own Node IDs, and each has onward routes to all three
destinations.  The agency also operates cislunar and deep-space
ground facilities; onward forwarding from the relays uses
agency-managed routes and contact arrangements, which are not shown.

Example A and Example B are alternative configurations of the
allocator's zone.  The agency publishes one or the other, not both: a
CNAME and an SRV RRset must not coexist at the same owner name (see
{{dns-constraints}}).  All records shown are zone excerpts; routine
zone infrastructure (such as SOA and NS records) and DNSSEC records
are omitted.

Both configurations use the following address records for the relays:

~~~ dns
$ORIGIN space-agency.example.
$TTL 3600

relay-a IN A    192.0.2.10
relay-a IN AAAA 2001:db8::10
relay-b IN A    192.0.2.11
relay-b IN AAAA 2001:db8::11
~~~
{: #relay-addresses title="Relay Address Records"}

### Example A: Service Records in the Allocator's Zone

In this configuration, the SRV records for each destination are
published directly in the allocator's zone.

~~~ dns
$ORIGIN 978879.ipn.arpa.
$TTL 3600

_dtn-bundle._tcp.1 IN SRV 10 1 4556 relay-a.space-agency.example.
_dtn-bundle._tcp.1 IN SRV 10 1 4556 relay-b.space-agency.example.

_dtn-bundle._tcp.2 IN SRV 10 1 4556 relay-a.space-agency.example.
_dtn-bundle._tcp.2 IN SRV 10 1 4556 relay-b.space-agency.example.

_dtn-bundle._tcp.3 IN SRV 10 1 4556 relay-a.space-agency.example.
_dtn-bundle._tcp.3 IN SRV 10 1 4556 relay-b.space-agency.example.
~~~

A BP Agent that has a Bundle destined for `ipn:978879.2.345`, and
that uses TCPCLv4 {{TCPCL}}:

1. constructs the query name `_dtn-bundle._tcp.2.978879.ipn.arpa.`
   from Allocator Identifier 978879 and Node Number 2 of the
   destination;

2. queries for SRV records at that name, receiving two candidate
   relay endpoints, `relay-a.space-agency.example.` and
   `relay-b.space-agency.example.`, both on port 4556 with priority 10
   and weight 1;

3. orders the candidates as specified in {{RFC2782}}: because both
   records have the same priority and weight, either relay can be
   selected first;

4. queries for A and AAAA records at the selected target; for
   `relay-a.space-agency.example.`, it receives `192.0.2.10` and
   `2001:db8::10`; and

5. attempts a TCPCLv4 connection to `[2001:db8::10]:4556` or
   `192.0.2.10:4556`.  If that relay cannot be reached, the BP Agent
   can try the other candidate, `relay-b.space-agency.example.`
   (`[2001:db8::11]:4556` or `192.0.2.11:4556`).  Once a session is
   established, it transfers the Bundle.

### Example B: Service Lookup Redirected through a CNAME

In this configuration, the allocator's zone contains, instead of
Example A's SRV records, a CNAME for each destination that redirects
the service lookup to a single shared name in the agency's zone:

~~~ dns
$ORIGIN 978879.ipn.arpa.
$TTL 3600

_dtn-bundle._tcp.1 IN CNAME (
    _dtn-bundle._tcp.ingress.space-agency.example. )
_dtn-bundle._tcp.2 IN CNAME (
    _dtn-bundle._tcp.ingress.space-agency.example. )
_dtn-bundle._tcp.3 IN CNAME (
    _dtn-bundle._tcp.ingress.space-agency.example. )
~~~

The agency publishes the shared service in its own zone:

~~~ dns
$ORIGIN space-agency.example.
$TTL 3600

_dtn-bundle._tcp.ingress IN SRV (
    10 1 4556 relay-a.space-agency.example. )
_dtn-bundle._tcp.ingress IN SRV (
    10 1 4556 relay-b.space-agency.example. )
~~~

The BP Agent starts with the same query as in Example A: it queries
for SRV records at `_dtn-bundle._tcp.2.978879.ipn.arpa.`.  Ordinary
DNS resolution follows the CNAME at that name to
`_dtn-bundle._tcp.ingress.space-agency.example.` and returns the SRV
RRset found there, which contains the same two candidate relay
endpoints as in Example A.  There is no separate preliminary CNAME
query.  Relay selection, address resolution, and establishment of the
TCPCLv4 connection then proceed as in Example A.

With this configuration, the agency maintains the common relay
selection in one SRV RRset.  The allocator's zone holds only the
aliases, and the set of relays can be changed in
`space-agency.example.` without changes to the allocator's zone.

### Shared Ingress Behavior {#shared-ingress}

The following points apply to both configurations:

* Equal SRV priorities and weights provide approximately balanced
  relay selection across connection attempts {{RFC2782}}.  They do
  not guarantee equal bundle volume, because a session can carry any
  number of Bundles, and they do not indicate current server load.

* Multiple destination FQNNs can share the same discovery result.
  Queries for (978879,1), (978879,2), and (978879,3) all yield the
  same two relays, but the destinations remain distinct, and each
  Bundle retains its destination EID; for example, a Bundle for the
  mars-orbiter is still destined for `ipn:978879.2.345` when it is
  transferred to a relay.  Each relay is a session peer with its own
  Node ID (see {{cl-peer-identity}}).

* After accepting a Bundle, a relay forwards it using the agency's
  configured onward routes and contact arrangements.  A successful
  connection to a relay does not imply that the destination
  spacecraft is currently reachable.

* The example uses a single public DNS view.  The agency's internal
  forwarding arrangements are configured in its BP Agents rather than
  in DNS, so they require no internal DNS overrides.

* The public records expose the advertised FQNNs, their association
  with the ingress service, and the relay addresses.  They do not
  describe the agency's ground facilities or contact arrangements.
  The example limits disclosure of internal topology; it does not
  provide confidentiality for the published data.

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
from the example range in {{Section 9.1 of RFC9758}}.  They are
additional illustrative allocator namespaces, and the example assumes
that the agency of {{examples}} is also authorized to administer
them.  The records are zone excerpts; routine zone infrastructure and
DNSSEC records are omitted.  The address records for the relays are
those in {{relay-addresses}}.

~~~ dns
$ORIGIN ipn.arpa.
$TTL 3600

974848 IN DNAME 974848.allocators.space-agency.example.
974849 IN DNAME 974849.allocators.space-agency.example.
~~~

~~~ dns
$ORIGIN 974848.allocators.space-agency.example.
$TTL 3600

_dtn-bundle._tcp.2 IN SRV 10 1 4556 relay-a.space-agency.example.
_dtn-bundle._tcp.2 IN SRV 10 1 4556 relay-b.space-agency.example.

$ORIGIN 974849.allocators.space-agency.example.

_dtn-bundle._tcp.2 IN SRV 10 1 4556 relay-a.space-agency.example.
_dtn-bundle._tcp.2 IN SRV 10 1 4556 relay-b.space-agency.example.
~~~

A BP Agent discovering TCPCLv4 service for `ipn:974848.2.345`
constructs its initial query from the standard DNS base name, exactly
as in {{discovery-model}}, and queries for SRV records at:

~~~
_dtn-bundle._tcp.2.974848.ipn.arpa.
~~~

The DNAME at `974848.ipn.arpa.` redirects this query to:

~~~
_dtn-bundle._tcp.2.974848.allocators.space-agency.example.
~~~

There the BP Agent finds the same relay-a/relay-b SRV pair as in
{{examples}}.  Ordinary DNS processing performs the redirection
({{Section 3 of RFC6672}}).  The BP Agent does not construct any name
in the agency's zone itself.

The example retains the Allocator Identifier in each target subtree.
As a result, node 2 under Allocator 974848
(`2.974848.allocators.space-agency.example.`) remains distinct from
node 2 under Allocator 974849
(`2.974849.allocators.space-agency.example.`), even though their
service records select the same relay hosts.  Both FQNNs are also
distinct from (978879,2), the mars-orbiter; they identify other
nodes.  Pointing both DNAMEs at the same target would merge their
corresponding lookup names: queries for the FQNNs (974848,2) and
(974849,2) would then reach the same records.  Distinct target
subtrees need to be preserved whenever the allocator namespaces must
remain distinct.  Retaining the decimal allocator label in the target
is a convenient convention, not a DNS requirement.

One agency-managed zone can contain all of these subtrees.  One
DNAME is still required in ipn.arpa for each redirected Allocator
Identifier.  The technique reduces the number of separately
administered child zones, not the number of allocator entries in the
parent.

The following constraints apply:

* A DNAME redirects the descendants of its owner name, not the owner
  name itself ({{Section 2.3 of RFC6672}}).

* Records cannot be published beneath the DNAME owner in the source
  zone ({{Section 2.4 of RFC6672}}).  Once `974848.ipn.arpa.` owns a
  DNAME, data beneath the allocator name is published in the target
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
that namespace.  Because populated subtrees beneath 0.ipn.arpa are
not published in the public DNS, such DNAMEs exist only in locally
served instances of that zone.  The same applies to DNAMEs for
Experimental Use Allocator Identifiers (see
{{publication-restrictions}}).

### Provisioning Tools

Some DNS implementations provide implementation-specific facilities,
such as the `$GENERATE` directive of BIND {{BIND-ARM}}, as
conveniences for producing repetitive records.  For example, the two
parent-side DNAME records above can be expressed in BIND syntax as:

~~~ dns
$ORIGIN ipn.arpa.
$TTL 3600

$GENERATE 974848-974849 $ DNAME $.allocators.space-agency.example.
~~~

Such a directive expands into ordinary records in the zone.  It
introduces no numeric-range matching in DNS, and resolvers need no
support for it.  Similar tooling can generate NS records or other
repetitive data.  Use of these facilities is optional.

# Security Considerations

This section describes the trust model for forwarding information
derived from DNS and the use of DNSSEC with this mechanism.

## Discovery Yields Candidates

The discovery procedure in {{concept-of-operations}} identifies
candidate convergence-layer endpoints through which a destination
node might be reached.  A candidate can be the destination node
itself or a relay that accepts Bundles for onward forwarding, as in
{{shared-ingress}}.  Successful discovery does not guarantee that
the destination is currently reachable, that a candidate will accept
a session or a Bundle, or that the Bundle will eventually be
delivered.

## Discovery, Peer Authentication, and Forwarding Policy {#forwarding-policy}

Using DNS-derived forwarding information involves two distinct
authentication questions: whether the DNS data used in discovery is
authentic (see {{dnssec}}), and whether the convergence-layer peer to
which a session is established is the entity it claims to be (see
{{cl-peer-identity}}).  Authenticating one does not authenticate the
other.

A BP Agent uses DNS-derived forwarding information according to its
configured forwarding policy.  That policy may permit automatic use of
authenticated discovery mappings.  Discovery does not override
configured forwarding policy or convergence-layer security policy;
DNS-derived candidates are subject to the same policy as forwarding
information from other sources.

## DNSSEC {#dnssec}

Forged discovery information can redirect connection attempts, and
therefore bundle transfers, to endpoints chosen by an attacker.
DNSSEC {{RFC9364}} provides data origin authentication and integrity
for DNS data.  It does not establish the identity of the live
convergence-layer peer at the resulting address and port, and it
does not guarantee that the advertised service is available.  ENUM
makes similar observations ({{Section 7.1 of RFC6116}}).

Operators publishing discovery data for this mechanism in the public
DNS SHOULD provide DNSSEC authentication of that data, including the
necessary chain of trust.  This recommendation applies to relevant
alias, service, and address records, including records published
outside ipn.arpa:

* any CNAME or DNAME records that redirect the query, such as those in
  {{dns-constraints}} and {{managing-allocator-ranges}};

* the SRV (or other CL-specific) records; and

* the address records for the SRV target.

The ipn.arpa zone itself is signed (see {{creation-of-ipn-arpa}}), and
delegated allocator zones can be linked into its chain of trust with
DS records (see {{ds-management}}).  Signing ipn.arpa alone does not
authenticate records in delegated zones or external target
namespaces; each record used in discovery has its own security
status.

Validation of these records follows standard DNSSEC processing; this
document does not define a new validation procedure.  For example, a
CNAME synthesized from a validated DNAME does not require its own
signature ({{Section 3.1 of RFC6672}} and
{{Section 5.3.1 of RFC6672}}).

### Validation by BP Agents

BP Agents using public DNS discovery SHOULD use DNSSEC validation,
either locally or through a trusted validating resolver.  When relying
on validation performed by another resolver, the resolver and the
channel used to obtain its results need to satisfy the trust
requirements of {{Section 4.9.3 of RFC4035}}.

DNSSEC distinguishes data that is secure, insecure, bogus, or
indeterminate ({{Section 4.3 of RFC4035}}).  Insecure data lies in a
part of the namespace with no chain of trust, for example a target
zone that is not signed.  Bogus data fails validation where a chain
of trust is expected, which can indicate an attack or a
misconfiguration.  An application can also receive results whose
security status is unknown to it, for example from a resolver that
does not convey validation results.  The absence of an authentication
indication does not by itself establish which of these cases applies.

This document does not impose a universal policy of accepting only
secure results, and it does not authorize bypassing validation
failures.  Detailed acceptance, failure, and fallback policies remain
open (see {{security-open-items}}).

## Convergence-Layer Peer Identity {#cl-peer-identity}

The applicable convergence-layer specification and configured
security policy govern authentication of the convergence-layer peer.
For TCPCLv4, session security,
including certificate-based identification of the peer, is specified
in {{Section 4.4 of TCPCL}}, and the handling of the peer's Node ID
is specified in {{Section 4.6 of TCPCL}}.  This document does not
modify those procedures.

The name queried in ipn.arpa identifies the destination node, not the
convergence-layer peer, and is not the name of the host to which a
connection is made.  In {{examples}}, the peer is one of the relays,
`relay-a.space-agency.example.` or `relay-b.space-agency.example.`,
each a separate BP node with its own Node ID, distinct from the
destination's.  The Bundle's destination EID remains
`ipn:978879.2.345` regardless of the session peer's identity.  A BP
Agent therefore needs to distinguish the destination EID from the
Node ID that the peer presents.  {{Section 4.6 of TCPCL}} describes
the handling of a peer Node ID that differs from the one intended,
and of a peer Node ID that is not authenticated.

## Open Items {#security-open-items}

TODO: Specify the BP Agent's detailed acceptance, failure, and
fallback policies for discovery results, including the treatment of
insecure, bogus, and indeterminate results and of failed lookups.

TODO: Address local resolution, public negative answers, and DNSSEC
treatment for 0.ipn.arpa and Experimental Use Allocator Identifiers,
which lie beneath the signed public hierarchy (see
{{local-resolution-todo}}).

TODO: Address caching of discovery results, operation during
disruption, and other remaining security considerations.


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

IANA is requested to arrange the ongoing administration of ipn.arpa,
consistent with the arrangements approved by the IAB under
{{RFC3172}}.  Administration includes processing the requests
described in {{allocator-dns-provisioning}}.  It is distinct from
operation of the authoritative DNS servers for ipn.arpa; this document
does not require that IANA itself operate those servers.  In this
document,
the "ipn.arpa operator" is the entity that performs the DNS
provisioning and operational functions for the ipn.arpa zone under
those arrangements.

The ipn.arpa operator MUST sign the public ipn.arpa zone using
DNSSEC and maintain a chain of trust through the arpa zone.  This
requirement applies to the ipn.arpa zone itself.  Recommendations
concerning discovery data in delegated allocator zones and in other
namespaces are addressed to the operators of those zones and are
given in {{dnssec}}.

The following subsections establish restrictions on publishing
discovery data in the global DNS for two sets of names beneath
ipn.arpa.  These restrictions are established by this document; they
concern only the DNS discovery mechanism specified here and do not
alter any allocation or its Bundle Protocol semantics.

### Allocator Identifier Zero {#allocator-zero}

"0.ipn.arpa" corresponds to Allocator Identifier zero, the Default
Allocator ({{Section 3.2.2 of RFC9758}}).  Public DNS discovery for
FQNNs with Allocator Identifier zero is intentionally outside the
scope of this document.  A populated "0.ipn.arpa" zone MUST NOT ever
be published in the global DNS.  A "0.ipn.arpa" zone MAY exist in
local DNS resolvers or servers that serve it authoritatively as a
locally served zone {{RFC6303}}, much as the IN-ADDR.ARPA zones for
the private address space of {{RFC1918}} are served locally
({{Section 4.1 of RFC6303}}).

This restriction covers every Node Number under Allocator Identifier
zero.  That includes the Private Use Node Numbers 1 through 16383
({{Section 3.4.3 of RFC9758}}) and the Node Numbers 2097152 through
268435455 reserved for Private or Experimental Use, a reservation
carried over from the "CBHE Node Numbers" registry
({{Section 3.2.1 of RFC7116}}; {{Section 9.2 of RFC9758}}).

The restriction does not mean that Allocator Identifier zero has only
local significance: the Default Allocator's registry
({{Section 9.2 of RFC9758}}) contains assigned Node Numbers as well as
the Private Use and Private or Experimental Use ranges.  Supporting
Allocator Identifier zero in the public DNS would require additional
administration and delegation procedures for existing Default
Allocator Node Number allocations, together with a treatment of those
ranges and of locally served DNS data.

Note to the Working Group (to be removed before publication as an
RFC): Public DNS support for Allocator Identifier zero can be
reconsidered if the working group chooses to take on that scope.

### Experimental Use Allocator Identifiers {#experimental-allocators}

{{Section 9.1 of RFC9758}} designates Allocator Identifiers 1073741824
through 2147483647 inclusive (0x40000000 through 0x7FFFFFFF) as
Experimental Use.  Experimental Use values are not assigned to any
particular party ({{Section 4.2 of RFC8126}}), so independent
experiments may reuse the same identifier.  A globally authoritative
mapping from such an identifier to discovery data would therefore be
inappropriate.

For each Experimental Use Allocator Identifier A, a populated subtree
beneath "A.ipn.arpa" MUST NOT be published in the global DNS.
Locally scoped DNS discovery for Experimental Use Allocator
Identifiers is permitted: names beneath "A.ipn.arpa" MAY be served by
local DNS resolvers or servers within the scope of an experiment.

This DNS publication restriction is established by this document.
{{RFC9758}} designates the Experimental Use range but does not itself
restrict DNS publication.

### Scope of the Publication Restrictions {#publication-restrictions}

The restrictions in {{allocator-zero}} and {{experimental-allocators}}
apply to every means of placing discovery data for the affected FQNNs
in the global DNS, including:

* discovery records, such as SRV, SVCB, or CNAME records, at or
  beneath "0.ipn.arpa" or "A.ipn.arpa" for an Experimental Use
  Allocator Identifier A;

* NS delegations of those names, or of names beneath them, to servers
  that publish such discovery data; and

* CNAME or DNAME records at or beneath those names that redirect
  discovery queries to data published elsewhere in the global DNS.

These restrictions do not preclude whatever empty or insecure global
representation of these names is chosen to support local resolution
(see {{local-resolution-todo}}).

TODO (next editing pass; a separate design question from the
publication policy above): Specify how "0.ipn.arpa" and the names of
Experimental Use Allocator Identifiers are represented in the global
DNS so that locally served answers can still be validated.
Because ipn.arpa is signed (see {{creation-of-ipn-arpa}}), an
authenticated denial of existence for these names would cause
validating resolvers to reject locally served data;
locally served zones need an insecure delegation or must be within an
insecure zone ({{Section 7 of RFC6303}}).  Compare the unsigned
delegation of home.arpa to black-hole servers
({{Section 7 of RFC8375}}).  Also consider adding "0.ipn.arpa" to the
IANA "Locally-Served DNS Zones" registry ({{Section 6 of RFC6303}}).
The Experimental Use range comprises 2^30 sibling names directly
beneath ipn.arpa, so its treatment may differ.  Related questions
include preventing queries for these names from leaking into the
public DNS and the negative answers that the public DNS returns for
them.
{: #local-resolution-todo}

## Allocator DNS Provisioning {#allocator-dns-provisioning}

IANA and the ipn.arpa operator are requested to maintain a procedure
that links requests for DNS provisioning at "A.ipn.arpa" to the entry
for Allocator Identifier A in the "'ipn' Scheme URI Allocator
Identifiers" registry ({{Section 9.1 of RFC9758}}).  The storage
format, the request interface, and the presentation of provisioning
information are left to IANA and the ipn.arpa operator.

DNS provisioning is optional.  It can be requested when an Allocator
Identifier is assigned or at any later time.  Having a DNS entry is
not a condition for the validity or use of an allocator assignment.

These procedures apply only to eligible assigned Allocator
Identifiers.  Allocator Identifier zero and Experimental Use Allocator
Identifiers are not eligible for public publication (see
{{publication-restrictions}}), and this document does not request any
public provisioning beneath their names.

### Request Authorization {#request-authorization}

Requests to create, modify, or remove DNS provisioning for an eligible
assigned Allocator Identifier MUST be authenticated, and MUST be
authorized by the Change Controller recorded for that identifier in
the registry or by a representative that the Change Controller has
authorized.  A representative can be, for example, a DNS service
provider or an organizational unit authorized to manage particular
identifiers; it does not need a separate numeric allocation.

These checks apply to NS, DNAME, and DS requests, including a request
to convert an allocator name between NS delegation and DNAME
redirection.

### Provisioning Alternatives {#provisioning-alternatives}

The parent-zone configuration at "A.ipn.arpa" is one of the following:

NS delegation:
: The requester supplies the domain names of the authoritative
  nameservers for the allocator zone, any addresses needed for glue,
  and any DS records to be published (see {{ds-management}}).

DNAME redirection:
: The requester supplies the fully qualified target domain name.  See
  {{managing-allocator-ranges}} and {{RFC6672}}.

These are alternatives.  A DNAME at the allocator name in ipn.arpa
does not coexist with an NS delegation, or with the DS records of a
delegation, at that name ({{Section 2.3 of RFC6672}}).

A single request can cover multiple Allocator Identifiers within the
requester's allocation.  It identifies each affected identifier and
the configuration that applies to it.  Batch provisioning introduces
no numeric-range syntax in DNS; each affected allocator name has its
own parent-zone records (compare {{managing-allocator-ranges}}).

### Maintenance {#provisioning-maintenance}

The same procedures support later updates and removal, including a
change of nameserver provider, a change of DNAME target, and changes
of DNSSEC keys (see {{ds-management}}).

When the registered Change Controller for an Allocator Identifier
changes, or when an assignment is withdrawn or, if applicable,
reassigned, IANA and the ipn.arpa operator review the associated DNS
provisioning and coordinate with the currently authorized party to
update or remove provisioning that no longer applies.  This document
does not define a revocation process for numeric assignments or any
fixed deadline for such changes.

Records beneath an allocator name, such as node-level discovery
records and any further delegations, are managed by the allocator's
authorized DNS administrators.  These procedures do not require IANA
or the ipn.arpa operator to provision individual Node Numbers.

### DS Records for Delegated Allocator Zones {#ds-management}

The ipn.arpa operator MUST support authenticated requests to add,
update, and remove DS records for delegated allocator zones.  Such
requests are authorized as described in {{request-authorization}}.
The operator of a delegated allocator zone is responsible for
coordinating changes to its signing keys with updates to the
corresponding DS records in ipn.arpa.

This provides support for secure delegations; it does not require
that every delegated allocator zone be signed.  This document does
not specify the operational interface for these requests or the
format in which DS data is stored.

DS procedures apply to NS delegations.  A DNAME in ipn.arpa (see
{{managing-allocator-ranges}}) is not a delegation: it is signed as
ordinary data in the ipn.arpa zone, and the namespace to which it
redirects has its own DNSSEC chain of trust.

## Request Validation and Allocation Review {#request-validation}

The allocation procedures of {{Section 9.1 of RFC9758}}, including
Expert Review, continue to apply to numeric assignments of Allocator
Identifiers.  Routine DNS provisioning changes for an existing
assignment do not require a new allocation Expert Review.

The ipn.arpa operator performs the applicable DNS technical checks on
provisioning requests under its published operational procedures.
IANA's technical requirements for delegations in the zones it manages
{{IANA-NS-REQS}} are relevant background.  DNAME provisioning is
subject to the requirements of {{RFC6672}}.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
