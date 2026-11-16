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
namespace for the IPN node identifiers of the Bundle Protocol (BP).
Each Fully Qualified Node Number has a DNS base name of the form
N.A.ipn.arpa, where A is the Allocator Identifier and N is the Node
Number.  A BP Agent can query beneath that name for the records that a
convergence-layer discovery binding specifies, such as SRV or SVCB
records.  These records identify the convergence-layer endpoints,
including relays, that are advertised for reaching a destination.

--- middle

# Introduction

The Bundle Protocol (BP) {{RFC9171}} supports communication in networks
that are subject to delay and disruption.  The ipn URI scheme
{{RFC9758}} identifies BP nodes and services with numbers.  An
Allocator assigns Node Numbers, and IANA assigns an Allocator
Identifier to each Allocator.  The Allocator Identifier and the Node
Number together form a Fully Qualified Node Number (FQNN).

A BP Agent can know an IPN destination without knowing a DNS domain
associated with its allocator.  The ipn.arpa namespace provides a
common starting point for discovery and links DNS administration to
allocator assignments.  Each FQNN has one DNS base name,
`N.A.ipn.arpa.` ({{discovery-model}}).  A BP Agent can query beneath
that name for the records that a convergence-layer (CL) discovery
binding defines.  These records advertise CL endpoints of the
destination node or of a relay that accepts Bundles for onward
forwarding.

DNS discovery is optional, and it supplies only candidate next hops
({{discovery-results}}).  It is not a BP routing protocol, and it does
not do contact planning.  The BP Agent and its configured policy
select paths and make forwarding decisions.

Public provisioning is limited to eligible Allocator Identifiers
({{allocator-dns-provisioning}}).  Names for Allocator Identifier zero
and Experimental Use Allocator Identifiers are served only by
explicitly configured local DNS ({{publication-restrictions}},
{{locally-served}}).


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
: The combination of an Allocator Identifier and a Node Number, which
  identifies one node ({{Section 3.3.1 of RFC9758}}).  This document
  writes an FQNN as (Allocator Identifier, Node Number), for example
  (978879,2).

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
: The DNS name, relative to a DNS base name, at which a discovery
  binding queries for records.

ipn.arpa operator:
: The entity that does the DNS provisioning and operational functions
  for the ipn.arpa zone ({{creation-of-ipn-arpa}}).

# Concept of Operations {#concept-of-operations}

A BP Agent populates its Forwarding Information Base (FIB) from
configuration and from other sources.  It can also create FIB entries
from DNS records, as this section describes.

## Discovery Model {#discovery-model}

Discovery uses the FQNN of the destination node.

Before constructing a DNS name, a BP Agent MUST parse and interpret
the destination ipn URI or ipn EID according to {{RFC9758}}.  This
includes the text syntax of {{Section 4 of RFC9758}} and the CBOR
decoding of {{Section 6.2 of RFC9758}}.  The result is a numeric
Allocator Identifier, Node Number, and Service Number.  An Allocator
Identifier that the text form omits has the value zero
({{Section 3.2.2 of RFC9758}}); the DNS name still contains the
allocator label.

For Allocator Identifier A and Node Number N, the DNS base name MUST
be:

~~~
N.A.ipn.arpa.
~~~

Each of N and A MUST be represented as a single label containing its
unsigned decimal value, using ASCII digits without leading zeroes.
The value zero is the single digit "0".  The labels are not padded and
are not split into individual digits.

The Service Number is not part of the name, so all ipn URIs with the
same FQNN map to the same DNS base name:

| ipn URI | DNS base name |
|---|---|
| `ipn:978879.2.345` | `2.978879.ipn.arpa.` |
| `ipn:978879.2.0` | `2.978879.ipn.arpa.` |
| `ipn:16384.345` | `16384.0.ipn.arpa.` |
| `ipn:0.16384.345` | `16384.0.ipn.arpa.` |
{: title="Example DNS Base Names"}

The name comes from the parsed numbers, not from the text of the URI.
Equivalent representations of an FQNN therefore give the same name.
This includes `ipn:16384.345` and `ipn:0.16384.345`, and the
two-element and three-element CBOR encodings of an ipn EID
({{Section 6.4 of RFC9758}}).

The same rule constructs names for Allocator Identifier zero,
including Private Use Node Numbers, and for Experimental Use Allocator
Identifiers.  These names are not in the public hierarchy
({{publication-restrictions}}), but local DNS can serve them
({{locally-served}}).

The Null ipn URI ({{Section 3.1 of RFC9758}}) and LocalNode ipn URIs
({{Section 3.4.2 of RFC9758}}) do not identify a remote destination.
A BP Agent MUST NOT perform DNS discovery for the Null ipn URI or for
a LocalNode ipn URI.  The BP Agent identifies these cases from the
complete FQNN after interpretation: (0,0) is the Null ipn URI, and
(0,4294967295) is a LocalNode ipn URI.  For example, an ipn URI with a
zero Allocator Identifier, a zero Node Number, and a non-zero Service
Number is the Null ipn URI ({{Section 3.4.1 of RFC9758}}).  Node
Numbers 0 and 4294967295 have no special meaning under other Allocator
Identifiers; (978879,0) is an ordinary FQNN.
{{Section 5.4 of RFC9758}} also prohibits the publication of LocalNode
ipn EIDs in any node identification directory, such as DNS.

## TCPCLv4 Discovery Binding {#tcpcl-binding}

This document specifies TCPCLv4 {{TCPCL}} discovery with SRV records
{{RFC2782}} at the following CL-specific query name:

~~~
_dtn-bundle._tcp.N.A.ipn.arpa.
~~~

For example, the query name for the FQNN (978879,2) is
`_dtn-bundle._tcp.2.978879.ipn.arpa.`.  {{TCPCL}} registers the
"dtn-bundle" service name and TCP port 4556, but it does not define
this discovery procedure.

The BP Agent constructs the query name directly.  Unlike conventional
reverse DNS in in-addr.arpa and ip6.arpa, this procedure needs no
preliminary PTR lookup.

SRV processing follows {{RFC2782}}, including priority, weight, port,
and target processing.  The following rules apply to this binding:

* A new TCPCLv4 discovery result requires a usable SRV record.  If
  the lookup obtains no usable SRV record, discovery supplies no new
  candidate.

* The address-record fallback in the usage rules of {{RFC2782}} does
  not apply.  After an unsuccessful SRV lookup, a BP Agent MUST NOT
  resolve the DNS base name for address records and connect to TCP
  port 4556 as part of this binding.

* An SRV Target of "." indicates that the service is not available at
  that domain ({{RFC2782}}).

These rules apply only to this binding; other bindings define their
own rules ({{other-cls}}).  The rules do not affect independently
obtained forwarding information or retained discovery results
({{discovery-results}}, {{caching}}).

## Examples {#examples}

The examples describe one deployment: a fictional space agency that
offers a shared public ingress service.  Other deployments can use a
separate relay for each node, or publish the endpoints of each node.
This document does not require the architecture shown here.

The agency administers Allocator Identifier 978879.  This identifier,
and the identifiers in {{managing-allocator-ranges}}, are from the
range that {{Section 9.1 of RFC9758}} reserves for examples.  The
excerpts do not request public provisioning of these identifiers.
They omit routine zone infrastructure, such as SOA and NS records, and
DNSSEC records.

| FQNN | Destination | DNS base name |
|---|---|---|
| (978879,1) | mission operations center | `1.978879.ipn.arpa.` |
| (978879,2) | mars-orbiter | `2.978879.ipn.arpa.` |
| (978879,3) | lunar-lander | `3.978879.ipn.arpa.` |
{: title="Example Destinations"}

The destination names are descriptive labels.  They do not depend on
the Node Numbers.

Two relay BP Agents, `relay-a.space-agency.example.` and
`relay-b.space-agency.example.`, provide the ingress service for all
three destinations.  Each relay is a separate BP node with its own
Node ID, and each relay has onward routes to all three destinations.
Onward forwarding uses the cislunar and deep-space ground facilities,
routes, and contact arrangements of the agency.  These are not shown.

Example A and Example B are alternative configurations of the
allocator's zone.  The agency publishes only one of them, because a
CNAME and an SRV RRset cannot coexist at the same owner name
({{dns-constraints}}).  Both configurations use these address records:

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

A BP Agent that uses TCPCLv4 to reach `ipn:978879.2.345`:

1. constructs the query name `_dtn-bundle._tcp.2.978879.ipn.arpa.`;

2. receives two SRV records, with the targets
   `relay-a.space-agency.example.` and
   `relay-b.space-agency.example.`, each with priority 10, weight 1,
   and port 4556;

3. orders the candidates as specified in {{RFC2782}}; because the
   priorities and weights are equal, either relay can be first;

4. resolves the selected target; for `relay-a.space-agency.example.`,
   it receives `192.0.2.10` and `2001:db8::10`; and

5. attempts a TCPCLv4 connection to `[2001:db8::10]:4556` or
   `192.0.2.10:4556`.  If the attempt fails, it can try
   `relay-b.space-agency.example.` (`[2001:db8::11]:4556` or
   `192.0.2.11:4556`).  If it establishes a session with a peer that
   its policy accepts ({{forwarding-policy}}), it transfers the
   Bundle.

### Example B: Service Lookup Redirected through a CNAME

Instead of SRV records, the allocator's zone contains a CNAME for each
destination.  Each CNAME points to one shared name in the agency's
zone:

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

The BP Agent sends the same query as in Example A.  Ordinary DNS
resolution follows the CNAME and returns the shared SRV RRset, which
contains the same two relays.  The remaining steps are the same as in
Example A.

The agency maintains the relay selection in one SRV RRset.  It can
change the relays in `space-agency.example.` without changes to the
allocator's zone.

### Shared Ingress Behavior {#shared-ingress}

The following points apply to both configurations:

* Equal SRV priorities and weights express equal preference
  ({{RFC2782}}).  They do not give equal bundle volume, because one
  session can carry many Bundles.  They do not indicate server load.

* Several FQNNs can share one discovery result, but each Bundle keeps
  its destination EID.  For example, a Bundle for the mars-orbiter is
  destined for `ipn:978879.2.345` when a relay receives it.  Each
  relay presents its own Node ID ({{cl-peer-identity}}).

* The example uses one public DNS view.  The agency configures its
  internal forwarding in its BP Agents, not in DNS, so it needs no
  internal DNS overrides.

* The public records show the advertised FQNNs, the ingress service,
  and the relay addresses.  They do not show the ground facilities or
  contact arrangements ({{privacy}}).

## DNS Constraints {#dns-constraints}

The following DNS rules apply to the configurations above:

* A CNAME aliases only its own owner name
  ({{Section 3.6.2 of RFC1034}}).  To redirect a service lookup, the
  CNAME is at the complete service owner name, such as
  `_dtn-bundle._tcp.2.978879.ipn.arpa.`.  A CNAME at
  `2.978879.ipn.arpa.` does not redirect the lookup.

* A CNAME record and an SRV RRset cannot coexist at the same owner
  name ({{Section 3.6.2 of RFC1034}}).

* An SRV Target other than "." must have address records and must not
  be an alias ({{RFC2782}}; {{Section 5.1 of RFC6672}}).

## Other Convergence Layers {#other-cls}

The discovery binding for another CL defines its query names, record
types, processing rules, and fallback behavior relative to the DNS
base name.  If a binding uses SVCB records, {{RFC9460}} and the
binding govern AliasMode, ServiceMode, parameter compatibility,
endpoint selection, and fallback.  SRV rules do not apply to other
record types.  For example, a ServiceMode SVCB TargetName of "."
means the owner name of the record ({{Section 2.5.2 of RFC9460}}),
not that the service is unavailable.

This document does not require a BP Agent to query for every CL that
it supports, and it does not specify an order in which to try CLs.

# Operational Considerations

If DNS does not meet the reliability or performance requirements of a
deployment, particularly during disruption, the deployment can obtain
forwarding information by other means.

## Discovery Results and Failures {#discovery-results}

Discovery succeeds when a BP Agent obtains usable connection
information under the applicable discovery binding ({{tcpcl-binding}},
{{other-cls}}).  A syntactically valid DNS answer does not always
contain usable connection information.  The binding and the
record-type specification define candidate selection and
compatibility checks.

A discovery result identifies candidate CL endpoints.  A candidate can
be the destination node or a relay that accepts Bundles for onward
forwarding.  A result does not show that the destination is reachable
now, that a candidate will accept a session or a Bundle, or that the
Bundle will be delivered.

DNS terminology {{RFC2308}} distinguishes these unsuccessful
outcomes:

NXDOMAIN:
: The name at which resolution stopped, possibly the target of an
  alias, does not exist.

NODATA:
: The name exists, but it has no records of the requested type.

SERVFAIL, timeout, and other resolution failures:
: The lookup did not complete.  This does not show that the service
  is absent.

These outcomes apply only to the DNS discovery attempt.  They do not
show that the FQNN is invalid or that other means cannot reach it.
The absence of discovery records for one CL does not show the absence
of service for another CL.  {{tcpcl-binding}} specifies the TCPCLv4
result when no usable SRV record exists.

An unsuccessful discovery attempt does not invalidate forwarding
information from other sources, such as configured routes or other
routing mechanisms.  It does not require deletion of a Bundle.  The
existing policies of the BP Agent govern retries and route precedence.

## Caching and Operation During Disruption {#caching}

Positive and negative DNS caching follow the existing DNS
specifications, including {{RFC2308}}.

A retained discovery result depends on the remaining TTLs of the
records that produced it: aliases, SRV or SVCB records, and address
records.  These TTLs can be different.  Copying a result into
forwarding information should not silently make it permanent
configuration.  A BP Agent does not need to repeat the lookup before
each Bundle transfer.  TTL expiry does not require it to close an
established CL session or to delete a Bundle; refreshed information
affects later connection attempts.

If refresh is not possible, explicit operational policy governs any
continued use of expired discovery information.  That policy considers
possible changes to endpoints, aliases, or administrative control.
{{RFC8767}} describes how resolvers can serve stale DNS data; it does
not specify how a BP Agent retains forwarding information.

TTL expiry is different from signature expiry.  Validation checks the
RRSIG validity period ({{Section 5.3.1 of RFC4035}}), and the
signature expiration time limits the TTL of authenticated data
({{Section 5.3.3 of RFC4035}}).  An earlier successful validation
therefore does not stay valid indefinitely.  {{bpa-validation}}
describes the handling of validation failures.

## Locally Served Namespaces {#locally-served}

A deployment that uses a local instance of 0.ipn.arpa or of an
Experimental Use allocator subtree ({{publication-restrictions}})
configures it explicitly with ordinary DNS administration.  The
administrator loads the zones, selects authoritative servers, and
configures resolvers to forward queries for those names.  This
document does not request automatic resolver behavior or additions to
the IANA "Locally-Served DNS Zones" registry {{RFC6303}}.

The signed public ipn.arpa namespace does not give a chain of trust to
independently administered local data.  If the public namespace does
not contain a name, normal DNSSEC processing can authenticate the
nonexistence of that name.  A validator that uses only the public
chain therefore cannot authenticate a conflicting local answer.  A
validator can authenticate a signed local zone with a configured local
trust anchor ({{Section 7 of RFC6303}}).  {{Section 5.10 of RFC6840}}
describes how a validator selects among nested trust anchors.  The
trust anchor is necessary wherever validation occurs: in a trusted
validating resolver ({{bpa-validation}}) or in a BP Agent that
validates independently.

A deployment can also accept unsigned local data under explicit local
policy.  This acceptance is a policy decision, not a successful DNSSEC
validation.  Local resolution and trust settings are best limited to
the relevant subtrees, without disabling validation for all of
ipn.arpa.

Administrators should configure resolution so that queries for these
names stay within the intended administrative or experimental scope,
also when local resolution fails.  Public resolution cannot supply the
local data, and queries that reach the public DNS can disclose the
queried identifiers ({{privacy}}).  {{RFC6303}} describes the reasons
to contain such queries.  This guidance does not require resolvers in
general to recognize these namespaces.

## Managing Allocator Ranges {#managing-allocator-ranges}

An Allocator can hold a range of Allocator Identifiers
({{Section 3.2.1 of RFC9758}}).  A numeric range does not map to one
DNS subtree: each Allocator Identifier A has its own name,
`A.ipn.arpa.`, directly beneath ipn.arpa.  An operator that administers
several identifiers can delegate a zone for each identifier.
Alternatively, it can use DNAME records {{RFC6672}} to redirect each
allocator subtree into one zone that it manages.

In the following example, the agency of {{examples}} also administers
Allocator Identifiers 974848 and 974849.  The relay address records
are those in {{relay-addresses}}.

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

A BP Agent that discovers TCPCLv4 service for `ipn:974848.2.345`
constructs the usual query name ({{tcpcl-binding}}):

~~~
_dtn-bundle._tcp.2.974848.ipn.arpa.
~~~

The DNAME at `974848.ipn.arpa.` redirects the query to:

~~~
_dtn-bundle._tcp.2.974848.allocators.space-agency.example.
~~~

There, the BP Agent finds the same relay SRV records as in
{{examples}}.  Ordinary DNS processing does the redirection
({{Section 3 of RFC6672}}).  The BP Agent does not construct names in
the agency's zone.

Each target subtree keeps its Allocator Identifier label.  Thus
(974848,2) and (974849,2) stay distinct, although their SRV records
select the same relays.  Both FQNNs are also distinct from (978879,2),
the mars-orbiter.  If both DNAMEs pointed to the same target, queries
for (974848,2) and (974849,2) would reach the same records.  Distinct
target subtrees are therefore necessary when the allocator namespaces
must stay distinct.  The decimal allocator label in the target is a
convenient convention, not a DNS requirement.

One zone can contain all of these subtrees.  Each redirected Allocator
Identifier still needs its own DNAME in ipn.arpa.  The technique
reduces the number of child zones to administer, not the number of
allocator entries in the parent.

The following DNAME constraints apply:

* A DNAME redirects the descendants of its owner name, not the owner
  name itself ({{Section 2.3 of RFC6672}}).

* The source zone cannot contain records beneath the DNAME owner
  ({{Section 2.4 of RFC6672}}).  The data for names beneath
  `974848.ipn.arpa.` is in the target subtree.

* A DNAME rewrites names beneath its owner, but an NS delegation
  creates a zone cut.  A parent-side DNAME cannot coexist with a
  delegation NS RRset at the same owner name
  ({{Section 2.3 of RFC6672}}).  A delegated allocator zone can
  instead publish a DNAME at its own apex, with its SOA and NS records.
  This arrangement keeps separate child-zone administration.

{{Section 5.3 of RFC6672}} describes DNSSEC considerations for DNAME.
Locally served instances ({{locally-served}}) can use DNAMEs in the
same way, for example at `N.0.ipn.arpa.`, if the administrator of that
namespace authorizes them.

Implementation-specific tools, such as the `$GENERATE` directive of
BIND {{BIND-ARM}}, can produce repetitive records.  For example, this
BIND input produces the two DNAME records above:

~~~ dns
$ORIGIN ipn.arpa.
$TTL 3600

$GENERATE 974848-974849 $ DNAME $.allocators.space-agency.example.
~~~

The directive expands into ordinary records.  It adds no numeric-range
matching to DNS, and resolvers need no support for it.  The use of
such tools is optional.

# Security Considerations

## Authentication and Forwarding Policy {#forwarding-policy}

The use of DNS-derived forwarding information involves two separate
authentication questions.  DNSSEC authenticates the DNS data
({{dnssec}}).  The CL authenticates the session peer
({{cl-peer-identity}}).  The authentication of one does not
authenticate the other.

The configured forwarding and security policies of the BP Agent govern
the use of DNS-derived forwarding information.  These policies can
distinguish sources of forwarding information, for example configured
routes and discovery results, or authenticated and unauthenticated
results.  A BP Agent's configured forwarding policy may permit
automatic use of authenticated discovery mappings, subject to the
security requirements of the applicable CL.

## DNSSEC {#dnssec}

Forged discovery information can redirect connection attempts, and
thus bundle transfers, to endpoints that an attacker selects.  DNSSEC
{{RFC9364}} gives data origin authentication and integrity for DNS
data.  It does not show that the advertised service is available;
ENUM makes a similar observation ({{Section 7.1 of RFC6116}}).

Operators publishing discovery data for this mechanism in the public
DNS SHOULD provide DNSSEC authentication of that data, including the
necessary chain of trust.  This recommendation applies to all DNS
records relied upon in discovery, including records published outside
ipn.arpa:

* aliases and redirections, such as CNAME, DNAME, and SVCB AliasMode
  records;

* service records and other records that carry connection parameters,
  such as SRV and SVCB ServiceMode records; and

* address records for the resulting targets.

The ipn.arpa operator signs ipn.arpa ({{creation-of-ipn-arpa}}), and
DS records can link delegated allocator zones into that chain of trust
({{ds-management}}).  The signature on ipn.arpa does not authenticate
records in delegated zones or in external target namespaces.  Each
record has its own security status.

Validation follows standard DNSSEC processing.  For example, a CNAME
synthesized from a validated DNAME needs no signature of its own
({{Section 3.1 of RFC6672}}; {{Section 5.3.1 of RFC6672}}).

### Validation by BP Agents {#bpa-validation}

BP Agents using public DNS discovery SHOULD use DNSSEC validation,
either locally or through a trusted validating resolver.  When a BP
Agent relies on validation by another resolver, the resolver and the
channel to it need to satisfy the trust requirements of
{{Section 4.9.3 of RFC4035}}.

{{Section 4.3 of RFC4035}} defines the security states secure,
insecure, bogus, and indeterminate.  An insecure result means that the
validator has established that no chain of trust exists from a
configured trust anchor to the RRset.  For example, normal DNSSEC
processing proves that the delegation to the target zone is unsigned.
An insecure result does not mean that the validator failed to build a
chain.  Missing signatures where a secure chain requires them can
produce a bogus result.  A zone can contain signatures and still be
insecure if no chain of trust reaches it.

An application can also receive a result whose security status is
unknown to it, for example from a resolver that does not report
validation results.  The absence of an authentication indication does
not establish that a result is insecure.

This document does not require a policy of accepting only secure
results.  The configured policy of the BP Agent governs the acceptance
of insecure results and of results with unknown status.

A known validation failure is a failure of the affected discovery path
({{discovery-results}}).  It should not lead to automatic acceptance
of the same data without validation, for example through a query on a
non-validating path.  A BP Agent can still retrieve DNSSEC data to
validate it independently.  {{locally-served}} describes validation
for locally served namespaces.

## Convergence-Layer Peer Identity {#cl-peer-identity}

The applicable CL specification and the configured security policy
govern authentication of the CL peer.  For TCPCLv4,
{{Section 4.4 of TCPCL}} specifies session security, including
certificate-based peer identification, and {{Section 4.6 of TCPCL}}
specifies the handling of the peer Node ID.  This document does not
change these procedures.

A TCPCLv4 discovery result involves three different identifiers.  In
Example A ({{examples}}):

* The discovery name, `_dtn-bundle._tcp.2.978879.ipn.arpa.`, comes
  from the destination FQNN.

* The selected SRV Target, `relay-a.space-agency.example.`, is the DNS
  name that the BP Agent resolves to make the TCP connection.  If the
  configured policy invokes network-level authentication
  ({{Section 4.4.4.2 of TCPCL}}), this name is the DNS-ID reference
  identifier.  It is also the HostName in the TLS server_name
  extension ({{Section 4.4.3 of TCPCL}}).

* The peer Node ID is the Node ID of the relay, which the relay sends
  in its SESS_INIT message ({{Section 4.6 of TCPCL}}).

Network-level authentication of `relay-a.space-agency.example.`
authenticates the host name only.  It does not authenticate the Node
ID of the relay.  The destination EID of the Bundle stays
`ipn:978879.2.345`.  Discovery does not supply an expected peer Node
ID.  {{Section 4.6 of TCPCL}} describes the handling of a peer Node ID
that is different from the intended Node ID or that is not
authenticated.

## Privacy {#privacy}

Public discovery records can reveal destination identifiers, their
association with services and relays, and connection endpoints.
Queries for these records can reveal to resolvers and other observers
which destinations a BP Agent tries to reach, and when.  DNSSEC does
not provide confidentiality.  {{RFC9076}} describes DNS privacy threats
and mitigations, such as resolver selection and encrypted DNS
transports.  Operators can limit what published records disclose
({{shared-ingress}}), and they can keep queries for locally served
names within their intended scope ({{locally-served}}).

## Resource Consumption {#resource-consumption}

Bundles addressed to many distinct FQNNs, whether malicious or not,
can cause many DNS queries, much retained discovery state, and many
connection attempts.  Implementations SHOULD bound outstanding
discovery work, retained discovery state, and connection attempts,
using existing mechanisms such as resolver limits, queueing, and
connection management.


# IAB Considerations

In accordance with {{Section 3 of RFC3172}}, the IAB is requested to
approve the zone "ipn.arpa" as an infrastructure namespace derived
from IPN identifiers.  Names in this zone are constructed from FQNNs
({{discovery-model}}).  BP Agents use them to discover CL endpoints for
reaching IPN nodes.

This follows the precedent of e164.arpa {{RFC6116}}, in which names
are derived from E.164 telephone numbers and queried directly for
service records (NAPTR).

# IANA Considerations

## Creation of ipn.arpa {#creation-of-ipn-arpa}

Following approval by the IAB in accordance with
{{Section 3 of RFC3172}}, IANA is requested to add "ipn.arpa" to the
".arpa" domain.  {{discovery-model}} specifies the names in ipn.arpa.

IANA is requested to arrange the ongoing administration of ipn.arpa,
consistent with the arrangements that the IAB approves under
{{RFC3172}}.  Administration includes the processing of the requests
in {{allocator-dns-provisioning}}.  Administration is separate from the
operation of the authoritative DNS servers for ipn.arpa, and this
document does not require IANA itself to operate those servers.  The
ipn.arpa operator does the DNS provisioning and operational functions
under these arrangements.

The ipn.arpa operator MUST sign the public ipn.arpa zone using DNSSEC
and maintain a chain of trust through the arpa zone.  This requirement
applies only to the ipn.arpa zone.  {{dnssec}} gives recommendations
for the operators of delegated allocator zones and other namespaces.

## Publication Restrictions {#publication-restrictions}

This document restricts the publication of discovery data for two sets
of names in the public ipn.arpa hierarchy.  These restrictions come
from this document, not from {{RFC9758}}.  They apply only to DNS
discovery, and they do not change any allocation or its Bundle
Protocol semantics.

The restrictions apply to every means of placing discovery data for
the affected FQNNs in the public ipn.arpa hierarchy:

* discovery records, such as SRV, SVCB, or CNAME records, at or
  beneath the restricted names;

* NS delegations of the restricted names, or of names beneath them,
  to servers that publish such discovery data; and

* CNAME or DNAME records at or beneath the restricted names that
  redirect discovery queries to data published elsewhere in the global
  DNS.

Names in these subtrees are absent from the public zone.  They receive
ordinary negative answers with normal DNSSEC processing.  This document
requests no special public delegation for them.  For comparison,
home.arpa has an insecure public delegation to accommodate validators
without local trust configuration ({{Section 6.2 of RFC8375}};
{{Section 7 of RFC8375}}).  This document does not request that
arrangement.  {{locally-served}} gives operational guidance for local
instances.

### Allocator Identifier Zero {#allocator-zero}

"0.ipn.arpa" corresponds to Allocator Identifier zero, the Default
Allocator ({{Section 3.2.2 of RFC9758}}).  Public DNS discovery for
FQNNs with Allocator Identifier zero is intentionally outside the
scope of this document.  A populated "0.ipn.arpa" zone MUST NOT be
published in the public ipn.arpa hierarchy specified by this document.
A "0.ipn.arpa" zone MAY exist in local DNS resolvers or servers that
serve it authoritatively as a locally served zone {{RFC6303}}, much as
the IN-ADDR.ARPA zones for the private address space of {{RFC1918}}
are served locally ({{Section 4.1 of RFC6303}}).

The restriction covers every Node Number under Allocator Identifier
zero.  This includes the Private Use Node Numbers 1 through 16383
({{Section 3.4.3 of RFC9758}}) and the Node Numbers 2097152 through
268435455 reserved for Private or Experimental Use
({{Section 3.2.1 of RFC7116}}; {{Section 9.2 of RFC9758}}).

The restriction does not mean that Allocator Identifier zero has only
local significance.  The registry of the Default Allocator
({{Section 9.2 of RFC9758}}) also contains assigned Node Numbers.
Public support would need administration and delegation procedures for
these allocations, and a treatment of the private ranges and of
locally served DNS data.

Note to the Working Group (to be removed before publication as an
RFC): The working group can reconsider public DNS support for
Allocator Identifier zero if it chooses to take on that scope.

### Experimental Use Allocator Identifiers {#experimental-allocators}

{{Section 9.1 of RFC9758}} designates Allocator Identifiers 1073741824
through 2147483647 (0x40000000 through 0x7FFFFFFF) as Experimental
Use.  Experimental Use values are not assigned to any party
({{Section 4.2 of RFC8126}}), so independent experiments can use the
same identifier.  A globally authoritative mapping to discovery data
is therefore not appropriate.

For each Experimental Use Allocator Identifier A, a populated subtree
beneath "A.ipn.arpa" MUST NOT be published in the public ipn.arpa
hierarchy.  Names beneath "A.ipn.arpa" MAY be served by local DNS
resolvers or servers within the scope of an experiment.

## Allocator DNS Provisioning {#allocator-dns-provisioning}

IANA and the ipn.arpa operator are requested to maintain a procedure
that links requests for DNS provisioning at "A.ipn.arpa" to the entry
for Allocator Identifier A in the "'ipn' Scheme URI Allocator
Identifiers" registry ({{Section 9.1 of RFC9758}}).  IANA and the
ipn.arpa operator determine the storage format, the request interface,
and the presentation of provisioning information.

DNS provisioning is optional.  It can be requested when an Allocator
Identifier is assigned or at any later time.  A DNS entry is not a
condition for the validity or use of an allocator assignment.

Eligible Allocator Identifiers are those assigned for operational use
in the registry, subject to the exclusions in this document.  This
document does not request public provisioning for:

* Allocator Identifier zero ({{allocator-zero}});

* Experimental Use Allocator Identifiers
  ({{experimental-allocators}});

* unassigned or reserved Allocator Identifiers; or

* Allocator Identifiers 974848 through 978943, which are reserved for
  documentation examples ({{Section 9.1 of RFC9758}}).

The authorized DNS administrators of the allocator manage the records
beneath the allocator name, such as node-level discovery records and
further delegations.  These procedures do not require IANA or the
ipn.arpa operator to provision individual Node Numbers.

### Request Authorization {#request-authorization}

Requests to create, update, or remove NS, DNAME, or DS records for an
eligible Allocator Identifier are subject to the same administrative
authority and requester-verification procedures that IANA applies to
changes to the corresponding entry in the "'ipn' Scheme URI Allocator
Identifiers" registry.  DNS provisioning changes do not themselves
require a new review of the numeric allocation.  The same procedures
apply to a conversion between NS delegation and DNAME redirection.

### Provisioning Alternatives {#provisioning-alternatives}

The parent-zone configuration at "A.ipn.arpa" is one of the following:

NS delegation:
: The requester supplies the domain names of the authoritative
  nameservers, any addresses needed for glue, and any DS records
  ({{ds-management}}).

DNAME redirection:
: The requester supplies the fully qualified target domain name
  ({{managing-allocator-ranges}}).

A DNAME at the allocator name does not coexist with an NS delegation,
or with the DS records of a delegation, at that name
({{Section 2.3 of RFC6672}}).

One request can cover several Allocator Identifiers within the
allocation of the requester.  The request identifies each Allocator
Identifier and its configuration.  Each allocator name still has its
own parent-zone records; DNS has no numeric-range syntax.

### DS Records for Delegated Allocator Zones {#ds-management}

The ipn.arpa operator MUST support authenticated requests to add,
update, and remove DS records for delegated allocator zones.
{{request-authorization}} applies to these requests.  The operator of
a delegated allocator zone coordinates changes to its signing keys
with updates to the DS records in ipn.arpa.

This support enables secure delegations.  It does not require that
every delegated allocator zone be signed.

DS records apply only to NS delegations.  A DNAME in ipn.arpa is not a
delegation.  The ipn.arpa operator signs it as ordinary data in the
ipn.arpa zone, and the target namespace has its own DNSSEC chain of
trust.

### Request Validation and Allocation Review {#request-validation}

The allocation procedures of {{Section 9.1 of RFC9758}}, including
Expert Review, continue to apply to numeric assignments of Allocator
Identifiers ({{request-authorization}}).

The ipn.arpa operator does the applicable DNS technical checks on
provisioning requests under its published operational procedures.
The technical requirements of IANA for delegations in the zones that
it manages {{IANA-NS-REQS}} are relevant background.  DNAME
provisioning is subject to the requirements of {{RFC6672}}.
