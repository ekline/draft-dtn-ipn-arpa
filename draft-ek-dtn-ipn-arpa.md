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
  RFC9460:
  RFC9758:
  TCPCL: RFC9174

informative:
  RFC1918:
  RFC2308:
  RFC6116:
  RFC6303:
  RFC6840:
  RFC7116:
  RFC8126:
  RFC8375:
  RFC8767:
  RFC9076:
  RFC9171:
  RFC9520:
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

This document requests the creation of ipn.arpa, an infrastructure DNS
namespace for the Bundle Protocol (BP).  It maps a Fully Qualified
Node Number to the DNS base name `N.A.ipn.arpa.`, where N is the Node
Number and A is the Allocator Identifier.  A BP Agent can use that
name to discover a convergence-layer endpoint at the destination or
at a relay.  This document specifies an SRV binding for version 4 of
the TCP Convergence-Layer Protocol (TCPCLv4).

--- middle

# Introduction

The Bundle Protocol (BP) {{RFC9171}} supports communication in networks
subject to delay and disruption.  In the ipn URI scheme {{RFC9758}},
IANA assigns Allocator Identifiers, and Allocators assign Node Numbers.
An Allocator can hold more than one identifier.  The pair of Allocator
Identifier and Node Number forms a Fully Qualified Node Number (FQNN).

A BP Agent can know an IPN destination without knowing a DNS domain
associated with its allocator.  The ipn.arpa namespace provides a
common starting point for discovery and links DNS administration to
allocator assignments.  The FQNN determines the DNS base name
({{discovery-model}}); the convergence-layer (CL) discovery binding
determines which name and record type to query.

Discovery is optional.  It supplies candidate next hops, not routes
or contact plans.  The BP Agent makes forwarding decisions under its
configured policy.  Not all FQNNs have public discovery data
({{publication-restrictions}}).


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms from {{RFC9758}}:

Allocator Identifier:
: A 32-bit unsigned integer that identifies the Allocator that
  assigned a Node Number ({{Section 3.2 of RFC9758}}).

Node Number:
: A 32-bit unsigned integer that identifies a node in the context of
  an Allocator ({{Section 3.3 of RFC9758}}).

Fully Qualified Node Number (FQNN):
: The pair (Allocator Identifier, Node Number), for example (978879,2)
  ({{Section 3.3.1 of RFC9758}}).

Service Number:
: An unsigned integer that identifies a service on a node
  ({{Section 3.5 of RFC9758}}).

ipn URI:
: A URI in the 'ipn' URI scheme.  It contains an FQNN and a Service
  Number ({{Section 3 of RFC9758}}).  An ipn URI used as a BP Endpoint
  Identifier (EID) is an "ipn EID" ({{Section 5 of RFC9758}}).

This document defines the following terms:

DNS base name:
: The DNS name beneath ipn.arpa that represents an FQNN
  ({{discovery-model}}).

Discovery binding:
: The specification of DNS discovery for one CL.  It defines the
  query names, record types, processing rules, and fallback behavior.
  This document specifies the TCPCLv4 binding ({{tcpcl-binding}}).

CL-specific query name:
: The name at or beneath the DNS base name that a BP Agent queries
  first under a discovery binding.  Alias and service targets can be
  elsewhere.

ipn.arpa operator:
: The entity that provisions and operates the ipn.arpa zone
  ({{creation-of-ipn-arpa}}).

# Concept of Operations {#concept-of-operations}

A BP Agent can use DNS discovery alongside configuration and other
sources to populate its forwarding information base.

## Discovery Model {#discovery-model}

Before constructing a DNS name, a BP Agent MUST parse and interpret
the destination ipn URI or ipn EID according to {{RFC9758}}.  In the
two-field text form, the omitted Allocator Identifier is zero; the
DNS name still includes the label `0` ({{Section 4 of RFC9758}}).
In the two-element CBOR scheme-specific part, the first integer
encodes both the Allocator Identifier and the Node Number
({{Section 6.2 of RFC9758}}).

For Allocator Identifier A and Node Number N, the DNS base name MUST
be:

~~~
N.A.ipn.arpa.
~~~

Each of N and A MUST be represented as a single label containing its
unsigned decimal value, using ASCII digits without leading zeroes.
Zero is the single digit `0`.  Neither label is padded or split.

The Service Number is not part of the DNS base name.  URIs with the
same FQNN give the same name, regardless of the Service Number or
the encoding of the FQNN ({{Section 6.4 of RFC9758}}):

| ipn URI | DNS base name |
|---|---|
| `ipn:978879.2.345` | `2.978879.ipn.arpa.` |
| `ipn:978879.2.0` | `2.978879.ipn.arpa.` |
| `ipn:16384.345` | `16384.0.ipn.arpa.` |
| `ipn:0.16384.345` | `16384.0.ipn.arpa.` |
{: title="Example DNS Base Names"}

The last two rows are usable in locally served instances, not for
public discovery.  The same local scope applies to discovery for
Experimental Use Allocator Identifiers ({{publication-restrictions}},
{{locally-served}}).

A BP Agent MUST NOT perform DNS discovery for the Null ipn URI or a
LocalNode ipn URI.  Neither identifies a remote destination.  The BP
Agent tests the interpreted pair (A,N), not the Node Number alone:

* (0,0): `ipn:0.0.0` is the Null ipn URI ({{Section 3.1 of RFC9758}}).
  A URI with this FQNN and a nonzero Service Number is invalid;
  {{Section 3.4.1 of RFC9758}} requires processors to treat it as Null.

* (0,4294967295): a LocalNode ipn URI identifies a resource on the
  local node ({{Section 3.4.2 of RFC9758}}).  {{Section 5.4 of RFC9758}}
  prohibits publication of LocalNode ipn EIDs in node identification
  directories, including DNS.

Neither special case applies under a nonzero Allocator Identifier.

## TCPCLv4 Discovery Binding {#tcpcl-binding}

This document specifies TCPCLv4 {{TCPCL}} discovery with SRV records
{{RFC2782}} at the following CL-specific query name:

~~~
_dtn-bundle._tcp.N.A.ipn.arpa.
~~~

The BP Agent constructs this name directly from the FQNN.  It needs no
preliminary PTR lookup as used in in-addr.arpa or ip6.arpa.

The registered service name for TCP port 4556 is `dtn-bundle`
({{Section 8.1 of TCPCL}}).  The BP Agent uses priority and weight to
select an SRV record under {{RFC2782}}, then uses its Target and Port.
The Port need not be 4556.  A Target of `.` indicates that the
specified service is unavailable at that domain.

Without a usable SRV record, this binding supplies no new candidate.
After an unsuccessful SRV lookup, the BP Agent MUST NOT fall back to
resolving the DNS base name for addresses and connecting to TCP port
4556 as part of this binding.  This exception to the suggested usage
in {{RFC2782}} does not affect other bindings, independently obtained
forwarding information, or retained results ({{other-cls}},
{{discovery-results}}, {{caching}}).

## Examples {#examples}

In this deployment, a fictional space agency offers one public ingress
service for three destinations.  Its Allocator Identifier, 978879,
and the identifiers in {{managing-allocator-ranges}} come from the
Example Range in {{Section 9.1 of RFC9758}}.  The zone excerpts omit
routine SOA, NS, and DNSSEC records.

| FQNN | Destination | DNS base name |
|---|---|---|
| (978879,1) | mission operations center | `1.978879.ipn.arpa.` |
| (978879,2) | mars-orbiter | `2.978879.ipn.arpa.` |
| (978879,3) | lunar-lander | `3.978879.ipn.arpa.` |
{: title="Example Destinations"}

The agency chose the destination names independently of the Node
Numbers.  Its two ingress relays are separate BP nodes with their own
Node IDs.  Both have onward routes to all three destinations through
the agency's cislunar and deep-space ground facilities.

Examples A and B are alternative configurations of the allocator's
zone ({{dns-constraints}}).  Both use these relay addresses:

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

The allocator's zone contains the SRV records for each destination:

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

For a bundle destined for `ipn:978879.2.345`, the BP Agent queries
for SRV at `_dtn-bundle._tcp.2.978879.ipn.arpa.` and orders the two
relay candidates according to {{RFC2782}}.  If it selects
`relay-a.space-agency.example.`, it resolves that target and attempts
a TCPCLv4 connection to `[2001:db8::10]:4556` or `192.0.2.10:4556`.
If it cannot reach that relay, it can try the other.  The BP Agent
transfers the bundle over a session that its policy accepts
({{forwarding-policy}}).

### Example B: Service Lookup Redirected through a CNAME

Instead of publishing an SRV RRset at each service name, the agency
makes each name an alias for the same SRV owner:

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

The agency publishes the shared SRV RRset in its own zone:

~~~ dns
$ORIGIN space-agency.example.
$TTL 3600

_dtn-bundle._tcp.ingress IN SRV (
    10 1 4556 relay-a.space-agency.example. )
_dtn-bundle._tcp.ingress IN SRV (
    10 1 4556 relay-b.space-agency.example. )
~~~

The BP Agent sends the same query as in Example A.  DNS resolution
follows the CNAME to the shared SRV RRset.  The agency can now change
the relay selection for all three destinations in one place.

### Shared Ingress Behavior {#shared-ingress}

Equal SRV priorities and weights give roughly balanced relay selection
across connection attempts ({{RFC2782}}).  They neither guarantee equal
traffic volume nor indicate current relay load.

The three destinations share relay endpoints, not identities.  Each
bundle keeps its destination EID.  After accepting it, the relay uses
the agency's routes and contact arrangements.  A connection to the
relay does not imply that a spacecraft is currently reachable.

The agency uses one public DNS view and needs no internal DNS
overrides.  Its records expose the advertised FQNNs, their association
with the ingress service, and the relay addresses, but not the internal
routes or contacts ({{privacy}}).

## DNS Constraints {#dns-constraints}

A CNAME redirects only its owner name ({{Section 3.6.2 of RFC1034}}).
In Example B the CNAME is at the full service name.  A CNAME at
`2.978879.ipn.arpa.` would not redirect the SRV lookup.  A CNAME
record and an SRV RRset cannot coexist at the same owner name.

The SRV lookup can follow a CNAME or DNAME.  In contrast, an SRV
Target other than `.` must have address records at that name, without
CNAME or DNAME redirection ({{RFC2782}}; {{Section 5.1 of RFC6672}}).

## Other Convergence Layers {#other-cls}

Other CLs can define bindings that use the same DNS base name.  For
SVCB, {{RFC9460}} and the applicable binding govern AliasMode,
ServiceMode, parameter compatibility, endpoint selection, and fallback.
SRV rules do not carry over: in ServiceMode, an SVCB TargetName of
`.` denotes the record's owner name in the response, not service
unavailability ({{Section 2.5.2 of RFC9460}}).

A BP Agent need not query for every CL it supports.  This document
specifies no order in which to try them.

# Operational Considerations

## Discovery Results and Failures {#discovery-results}

Discovery succeeds when a BP Agent obtains usable connection
information.  A valid DNS answer need not contain a usable candidate:
the discovery binding and the record-type specification determine
compatibility and candidate selection ({{tcpcl-binding}},
{{other-cls}}).  A candidate is an endpoint at the destination or at a
relay.  Discovery does not guarantee current reachability, acceptance
of a session or bundle, or eventual delivery.

Negative answers concern the name at the end of any alias chain
({{Section 2 of RFC2308}}):

NXDOMAIN:
: That name does not exist.

NODATA:
: That name exists, but has no RRset of the requested type.  The
  response code is NOERROR; the answer can still contain aliases.

A SERVFAIL response, a timeout, or another resolution failure gives no
answer about service availability.  None of these outcomes makes the
FQNN invalid.  Absence of one CL's records does not rule out another
CL's service.

An unsuccessful lookup does not invalidate independently obtained
forwarding information or require deletion of a bundle.  The BP
Agent's existing policies govern retries and route precedence.

## Caching and Operation During Disruption {#caching}

Resolvers cache positive answers ({{RFC1034}}), negative answers
({{RFC2308}}), and resolution failures ({{RFC9520}}).  A negative
answer to a refresh is a completed lookup, not a resolution failure.

A BP Agent need not repeat discovery for each bundle.  For a retained
candidate, the first TTL to expire among the RRsets used to obtain it
limits its freshness.  These can include alias, service, and address
RRsets.  Copying the candidate into a forwarding table does not
refresh it.  TTL expiry does not require closing an established CL
session or deleting a bundle; refreshed data informs later connections.

If the BP Agent cannot refresh an expired result, explicit operational
policy governs its continued use.  Endpoints, aliases, or administrative
control may have changed.  This differs from a resolver serving stale
DNS data under {{RFC8767}}.

TTL expiry also differs from signature expiry.  DNSSEC validation
checks the RRSIG validity period ({{Section 5.3.1 of RFC4035}}); the
signature expiration time limits the TTL of authenticated data
({{Section 5.3.3 of RFC4035}}).  Retaining a candidate does not extend
that validity period ({{bpa-validation}}).

## Locally Served Namespaces {#locally-served}

Local discovery under `0.ipn.arpa.` or an Experimental Use allocator
subtree ({{publication-restrictions}}) requires explicit DNS
configuration.  A resolver can serve the local zone itself or send
queries to a configured server for that subtree.  This document
requests neither automatic resolver behavior nor additions to the
IANA "Locally-Served DNS Zones" registry {{RFC6303}}.

The signed public hierarchy can authenticate the absence of a name in
public DNS, but not a conflicting local answer.  A validator can
authenticate a signed local zone with a configured local trust anchor
({{Section 7 of RFC6303}}; for nested anchors, see
{{Section 5.10 of RFC6840}}).  The trust configuration belongs where
validation takes place: at the trusted resolver ({{bpa-validation}})
or at a BP Agent that validates independently.

A deployment can accept unsigned local data under explicit policy;
that is not successful DNSSEC validation.  Administrators should scope
local resolution and trust settings to the relevant subtrees, not to
all of ipn.arpa.  Queries should stay within that scope even when local
resolution fails.  Public fallback cannot supply the local data and
can disclose the queried identifiers ({{RFC6303}}; {{privacy}}).

## Managing Allocator Ranges {#managing-allocator-ranges}

An Allocator can hold a range of Allocator Identifiers
({{Section 3.2.1 of RFC9758}}).  Each identifier has a separate name
under `ipn.arpa.`; DNS has no numeric-range syntax.  An agency that
wants to keep discovery records in its existing zone can request one
parent-side DNAME per identifier ({{provisioning-alternatives}}).

For example, the agency in {{examples}} also holds identifiers 974848
and 974849.  The `ipn.arpa.` operator publishes:

~~~ dns
$ORIGIN ipn.arpa.
$TTL 3600
974848 IN DNAME 974848.space-agency.example.
974849 IN DNAME 974849.space-agency.example.
~~~

The agency publishes both sets of records in its existing
`space-agency.example.` zone.  Neither DNAME target needs to be a
zone cut.  The relay names below are relative to the agency's zone
({{relay-addresses}}).

~~~ dns
$ORIGIN space-agency.example.
$TTL 3600
; ipn:974848.2.345
_dtn-bundle._tcp.2.974848 IN SRV 10 1 4556 relay-a
_dtn-bundle._tcp.2.974848 IN SRV 10 1 4556 relay-b
; ipn:974849.2.345
_dtn-bundle._tcp.2.974849 IN SRV 10 1 4556 relay-a
_dtn-bundle._tcp.2.974849 IN SRV 10 1 4556 relay-b
~~~

For `ipn:974848.2.345`, the BP Agent queries for SRV at
`_dtn-bundle._tcp.2.974848.ipn.arpa.` ({{tcpcl-binding}}).  DNS
substitutes `974848.space-agency.example.` for the matching suffix and
continues the lookup at
`_dtn-bundle._tcp.2.974848.space-agency.example.`.  A DNAME redirects
descendants of its owner; it does not redirect the owner itself
({{RFC6672}}).

Distinct target names keep node 2 in each allocator space distinct,
even though both use the same relays.  Using the allocator number as a
label is a convenience.  If both DNAMEs had the same target, both
queries would reach the same SRV owner.

Tools such as BIND's `$GENERATE` directive {{BIND-ARM}} can abbreviate
repetitive zone-file entries.  They produce ordinary DNS records.

# Security Considerations

## Authentication and Forwarding Policy {#forwarding-policy}

DNSSEC authenticates DNS data ({{dnssec}}), not the live session peer.
The CL specification and configured security policy govern peer
authentication ({{cl-peer-identity}}).

A BP Agent uses DNS-derived forwarding information according to its
configured forwarding policy.  That policy may permit automatic use
of authenticated discovery mappings, subject to the security
requirements of the applicable CL.

## DNSSEC {#dnssec}

Forged discovery information can redirect connection attempts and
bundle transfers to an endpoint an attacker selects.  DNSSEC
{{RFC9364}} provides data origin authentication and integrity, not a
guarantee of service availability (see also {{Section 7.1 of RFC6116}}
on ENUM).

Operators publishing discovery data for this mechanism in the public
DNS SHOULD provide DNSSEC authentication of that data, including the
necessary chain of trust.  This includes every RRset used for discovery,
even outside ipn.arpa: aliases and redirections (CNAME, DNAME, or SVCB
AliasMode), service and connection parameters (such as SRV or SVCB
ServiceMode), and addresses.

Signing ipn.arpa ({{creation-of-ipn-arpa}}) does not authenticate data
in delegated zones or external target namespaces.  Likewise, validating
a service RRset does not authenticate address RRsets supplied with it.
A CNAME synthesized from a validated DNAME needs no signature of its
own.  The validator checks the substitution
({{Section 5.3.1 of RFC6672}}).

### Validation by BP Agents {#bpa-validation}

BP Agents using public DNS discovery SHOULD use DNSSEC validation,
either locally or through a trusted validating resolver.  When a BP
Agent relies on validation by another resolver, the resolver and the
channel to it need to satisfy the trust requirements of
{{Section 4.9.3 of RFC4035}}.

{{Section 4.3 of RFC4035}} distinguishes secure, insecure, bogus, and
indeterminate RRsets.  Insecure means the validator has established
that no chain of trust reaches the RRset from any configured trust
anchor, for example through proof of an unsigned delegation.  An
insecure RRset can still have signatures.  Missing signatures where
a secure chain requires them can make the result bogus.  If the
validator cannot determine whether to expect signatures, the status
is indeterminate.

A BP Agent may not know the resolver's validation result.  A missing
authentication indication does not establish that an RRset is insecure.
The BP Agent's configured policy governs acceptance of insecure results
and results whose status it does not know.

A known validation failure is a failure of that discovery path
({{discovery-results}}).  It should not trigger automatic acceptance
of the same data through an unvalidated path.  The BP Agent can still
retrieve DNSSEC data for independent validation.  For local trust
configuration, see {{locally-served}}.

## Convergence-Layer Peer Identity {#cl-peer-identity}

For TCPCLv4, {{Section 4.4 of TCPCL}} specifies session security.  In
Example A ({{examples}}), three identifiers have different roles:

* `_dtn-bundle._tcp.2.978879.ipn.arpa.` is the SRV query name.  It
  comes from the destination FQNN.

* If the BP Agent selects `relay-a.space-agency.example.`, it resolves
  that SRV Target to establish the TCP connection.  When policy calls
  for network-level authentication, that host name is the DNS-ID
  reference identifier ({{Section 4.4.4.2 of TCPCL}}).  If the TLS
  client sends the server_name extension, it uses the relay host name,
  not the query name ({{Section 4.4.3 of TCPCL}}).

* The relay can supply its own Node ID in SESS_INIT.  The DNS-ID check
  does not authenticate that Node ID, and discovery supplies no
  expected peer Node ID.  {{Section 4.6 of TCPCL}} specifies how to
  handle absent, unexpected, or unauthenticated peer Node IDs.

The bundle's destination EID remains `ipn:978879.2.345`.

## Privacy {#privacy}

Public records can associate destination identifiers with services,
relays, and addresses ({{shared-ingress}}).  Queries can reveal to
resolvers and other observers which destinations a BP Agent tries to
reach and when.  DNSSEC does not provide confidentiality.  {{RFC9076}}
discusses resolver selection and encrypted DNS transports;
{{locally-served}} covers containment of local queries.

## Resource Consumption {#resource-consumption}

Traffic for many distinct FQNNs can consume resolver, memory, and
connection resources.  Implementations SHOULD bound outstanding
discovery work, retained discovery state, and connection attempts,
for example with resolver, queue, and connection limits.


# IAB Considerations

This document requests IAB approval for ipn.arpa as an infrastructure
namespace under {{Section 3 of RFC3172}}.  The two 32-bit components
of an FQNN, the Allocator Identifier and Node Number, determine the
DNS base name for discovery ({{discovery-model}}).  The precedent is
e164.arpa: an ENUM client derives a name from an E.164 telephone
number and queries it for NAPTR records {{RFC6116}}.

# IANA Considerations

## Creation of ipn.arpa {#creation-of-ipn-arpa}

Following IAB approval under {{Section 3 of RFC3172}}, IANA is
requested to add ipn.arpa to the arpa domain for names derived from
FQNNs ({{discovery-model}}).

IANA is requested to arrange ongoing administration, including request
processing ({{allocator-dns-provisioning}}), as approved by the IAB.
The ipn.arpa operator provisions and operates the zone; IANA need not
operate the authoritative servers itself.

The ipn.arpa operator MUST sign the public ipn.arpa zone using DNSSEC
and maintain a chain of trust through the arpa zone.  For the
recommendation to operators of delegated zones and other namespaces,
see {{dnssec}}.

## Publication Restrictions {#publication-restrictions}

This document excludes two sets of names from public DNS discovery.
The restrictions do not change any allocation or its Bundle Protocol
semantics under {{RFC9758}}.  At or beneath each restricted name,
they cover discovery records, NS delegations to servers that publish
such records, and CNAME or DNAME redirections that expose discovery
data elsewhere in the global DNS.

The public ipn.arpa zone omits these allocator names and their
descendants.  Queries for them receive ordinary DNS negative answers
with normal DNSSEC processing.  For home.arpa, an insecure
public delegation accommodates validators without local trust
configuration ({{Section 6.2 of RFC8375}}; {{Section 7 of RFC8375}});
this document requests no such delegation.  See {{locally-served}}
for local resolution and trust configuration.

### Allocator Identifier Zero {#allocator-zero}

Allocator Identifier zero denotes the Default Allocator
({{Section 3.2.2 of RFC9758}}).  Public DNS discovery for its FQNNs is
outside this document's scope.  A populated `0.ipn.arpa.` zone
MUST NOT be published in the public ipn.arpa hierarchy.  Local
resolvers or servers MAY serve it authoritatively, as with the reverse
zones for {{RFC1918}} private address space
({{Section 4.1 of RFC6303}}).

The restriction covers assigned Node Numbers as well as Private Use
Node Numbers 1 through 16383 ({{Section 3.4.3 of RFC9758}}) and Node
Numbers 2097152 through 268435455 reserved for Private or Experimental
Use ({{Section 3.2.1 of RFC7116}}; {{Section 9.2 of RFC9758}}).
Public support would require administration and delegation procedures
for existing assignments and a way to handle private ranges and
locally served data.

Note to the Working Group (to be removed before publication as an
RFC): The working group can reconsider public DNS support for
Allocator Identifier zero if it chooses to take on that scope.

### Experimental Use Allocator Identifiers {#experimental-allocators}

{{Section 9.1 of RFC9758}} designates Allocator Identifiers 1073741824
through 2147483647 (0x40000000 through 0x7FFFFFFF) as Experimental
Use.  Independent experiments can reuse these identifiers
({{Section 4.2 of RFC8126}}), so no single public mapping can
represent them all.

For each Experimental Use Allocator Identifier A, a populated subtree
beneath `A.ipn.arpa.` MUST NOT be published in the public ipn.arpa
hierarchy.  Local DNS resolvers or servers MAY serve names beneath
`A.ipn.arpa.` within the scope of an experiment.

## Allocator DNS Provisioning {#allocator-dns-provisioning}

IANA and the ipn.arpa operator are requested to maintain a procedure
that links each provisioning request for `A.ipn.arpa.` to the entry
for Allocator Identifier A in the "'ipn' Scheme URI Allocator
Identifiers" registry
({{Section 9.1 of RFC9758}}).  They determine the request interface,
storage format, and presentation of provisioning information.

A requester can seek DNS provisioning at assignment or later.  An
allocator assignment remains valid and usable without a DNS entry.
Only Allocator Identifiers assigned for operational use are eligible
for public provisioning.  This excludes Allocator Identifier zero
({{allocator-zero}}), Experimental Use Allocator Identifiers
({{experimental-allocators}}), unassigned or reserved identifiers,
and the Example Range, 974848 through 978943
({{Section 9.1 of RFC9758}}).

The allocator's authorized DNS administrators manage node-level records
and further delegations.  Neither IANA nor the ipn.arpa operator needs
to provision individual Node Numbers.

### Request Authorization {#request-authorization}

Requests to create, update, or remove NS, DNAME, or DS records for an
eligible Allocator Identifier are subject to the same administrative
authority and requester-verification procedures that IANA applies to
changes to the corresponding entry in the "'ipn' Scheme URI Allocator
Identifiers" registry.  DNS provisioning changes do not themselves
require a new review of the numeric allocation.  The same procedures
apply to a conversion between NS delegation and DNAME redirection.

### Provisioning Alternatives {#provisioning-alternatives}

A requester chooses one parent-zone configuration for `A.ipn.arpa.`:

NS delegation:
: The requester supplies the domain names of the authoritative
  nameservers, any addresses needed for glue, and any DS records
  ({{ds-management}}).

DNAME redirection:
: The requester supplies the fully qualified target domain name
  ({{managing-allocator-ranges}}).

A parent-side DNAME cannot coexist with a delegation NS RRset or its
DS records at the same name ({{Section 2.3 of RFC6672}}).

One request can cover several identifiers in the requester's
allocation.  It specifies the configuration for each identifier;
each allocator name has its own parent-zone records.

### DS Records for Delegated Allocator Zones {#ds-management}

The ipn.arpa operator MUST support authenticated requests to add,
update, and remove DS records for delegated allocator zones.
{{request-authorization}} applies to these requests.  The child-zone
operator coordinates signing-key changes that affect a parent DS
record with updates to that record.  Support for DS does not require
every delegated zone to be signed.

DS records apply to NS delegations, not parent-side DNAMEs.  The
ipn.arpa operator signs a DNAME as data in ipn.arpa.  Authentication
of data at its target depends on the target namespace's own chain of
trust.

### Request Validation and Allocation Review {#request-validation}

{{Section 9.1 of RFC9758}} governs numeric assignments, including
Expert Review.  The ipn.arpa operator checks DNS provisioning requests
under its published technical procedures.  IANA's requirements for
delegations {{IANA-NS-REQS}} provide background for NS requests;
{{RFC6672}} specifies the DNS requirements for DNAME redirection.
