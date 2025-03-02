---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "An ARPA Parent for IPN Addresses"
abbrev: "IPN dot ARPA"
category: std

docname: draft-todo-yourname-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: Internet
workgroup: DTN Working Group
keyword:
 - DTN
 - BP
 - IPN
 - DNS
venue:
  group: DTN
  type: Working Group
  mail: dtn@ietf.org
  arch: https://ietf.org/wg/dtn
  github: ekline/draft-dtn-ipn-arpa
  # latest: https://ietf.org/LATEST

author:
 -
    fullname: Erik Kline
    organization: Aalyria Technologies, LLC
    email: ek.ietf@gmail.com

normative:

informative:


--- abstract

This document requests a DNS parent for IPN addresses, discusses
the registration procedures and management of the DNS zone, as
well as some operational recommendations.  IPN addresses have a
DNS representation of the form 1.978879.ipn.arpa, for IPN node 1
under IPN Allocator 978879.

--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
