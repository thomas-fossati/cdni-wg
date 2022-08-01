---
v: 3

title: CDNI extensions for HTTPS delegation
abbrev: CDNI extensions for HTTPS delegation
docname: draft-ietf-cdni-interfaces-https-delegation-latest

category: std
consensus: true
submissiontype: IETF

ipr: trust200902
area: ART
workgroup: CDNI Working Group
keyword: [HTTPS, delegation, ACME, STAR]

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 - name: Frédéric Fieau
   org: Orange
   street: 40-48, avenue de la Republique
   city: Chatillon
   code: 92320
   country: France
   email: frederic.fieau@orange.com

 - name: Emile Stephan
   org: Orange
   street: 2, avenue Pierre Marzin
   city: Lannion
   code: 22300
   country: France
   email: emile.stephan@orange.com

 - name: Sanjay Mishra
   org: Verizon
   street: 13100 Columbia Pike
   city: Silver Spring
   code: MD 20904
   country: United States of America
   email: sanjay.mishra@verizon.com

venue:
  group: Content Delivery Networks Interconnection
  mail: cdni@ietf.org
  github: TODO

normative:
  RFC8006:
  RFC8008:
  RFC8739:
  RFC9115:

informative:
  RFC7336:
  RFC7337:

entity:
  SELF: "RFCthis"

--- abstract

This document defines a new Footprint and Capabilities metadata objects to
support HTTPS delegation between two or more interconnected CDNs.
Specifically, this document outlines CDNI Metadata interface objects for
the delegation method described in the ACME Delegation document (RFC 9115).

--- middle

# Introduction

Content delivery over HTTPS using one or more CDNs along the path requires
credential management.  This specifically applies when an entity delegates
delivery of encrypted content to another trusted entity.

{{RFC9115}} defines a mechanism where an upstream entity, that is, holder of a
X.509 certificate can give a temporary delegated authority, via issuing a
certificate to one or more downstream entities for the purposes of delivering
content on its behalf.  Furthermore, the upstream entity has the ability to
extend the duration of the certificate automatically and iteratively until it
allows the last renewal to end and therefore terminate the use of certificate
authority to the downstream entity.

More specifically, {{RFC9115}} defines a process where the upstream Content
Delivery Network (uCDN), the holder of the domain, generates on-demand a X.509
certificate for the downstream CDN (dCDN).  The certificate generation process
ensures that the certified public key corresponds to a private key controlled
by the downstream CDN.  {{RFC9115}} follows {{RFC8739}} for Short-Term,
Automatically Renewed Certificate (STAR) in the Automated Certificate
Management Environment (ACME).

This document defines CDNI Metadata to make use of HTTPS delegation between an
upstream CDN (uCDN) and a downstream CDN (dCDN) based on mechanism specified in
{{RFC9115}}.  Furthermore, it includes a proposal of IANA registry to enable
adding of delegation methods.

{{terminology}} defines terminology used in this document.  {{metadata}}
presents delegation metadata for the FCI interface.  Section 4 addresses the
metadata for handling HTTPS delegation with the Metadata Interface.  Section 5
addresses IANA registry for delegation methods.  Section 6 covers the security
considerations.

## Terminology {#terminology}

This document uses terminology from CDNI framework documents such as: CDNI
framework document {{RFC7336}}, CDNI requirements {{RFC7337}} and CDNI
interface specifications documents: CDNI Metadata interface {{RFC8006}} and
CDNI Footprint and capabilities {{RFC8008}}.

# Delegation metadata for CDNI FCI {#metadata}

The Footprint and Capabilities interface as defined in {{RFC8008}}, allows a
dCDN to send a FCI capability type object to a uCDN.  This draft adds an object
named FCI.SupportedDelegationMethods.

This object shall allow a dCDN to advertise the capabilities regarding the
supported delegation methods and their configuration.

The following is an example of the supported delegated methods capability
object for a CDN supporting STAR delegation method.

~~~
{
  "capabilities": [
    {
      "capability-type": "FCI.SupportedDelegationMethods",
      "capability-value": {
        "delegation-methods": [
          "AcmeStarDelegationDelegationMethod",
          "... Other supported delegation methods ..."
        ]
      },
      "footprints": [
        "Footprint objects"
      ]
    }
  ]
}
~~~

# Delegation metadata for CDNI

This section defines Delegation metadata using the current Metadata interface
model.  This allows bootstrapping delegation methods between a uCDN and a
delegate dCDN.

## Usage example related to an HostMatch object

This section presents the use of CDNI Delegation metadata to apply to an
HostMatch object, as defined in {{RFC8006}} and as specified in the following
sections.

The existence of delegation method in the CDNI metadata Object shall enable the
use of this method, as chosen by the delegating entity.  In the case of an
HostMatch object, the delegation method will be activated for the set of Host
defined in the HostMatch.  See {{acmedeleobj}} for more details about
delegation methods metadata specification.

The HostMatch object can reference a host metadata that points at the
delegation information.  Delegation metadata are added to a Metadata
object.

Those "delegation" metadata can apply to other MI objects such as
PathMatch object metadata.

Below shows both HostMatch and its Metadata related to a host, for
example, here is a HostMatch object referencing "video.example.com":

~~~
HostMatch:
{
  "host": "video.example.com",
  "host-metadata": {
    "type": "MI.HostMetadata",
    "href": "https://metadata.ucdn.example/host1234"
  }
}
~~~

Following the example above, the metadata is modeled for
ACMEStarDelegationMethod as follows:

~~~
{
  "generic-metadata-value": {
    "acme-delegations": [
      "https://acme.ucdn.example/acme/delegation/ogfr8EcolOT",
      "https://acme.ucdn.example/acme/delegation/wSi5Lbb61E4"
    ]
  }
}
~~~

## AcmeStarDelegationMethod object {#acmedeleobj}

This section defines the AcmeStarDelegationMethod object which describes
metadata related to the use of ACME/STAR API presented in {{RFC9115}}.

As expressed in {{RFC9115}}, when an origin has set a delegation to a specific
domain (i.e. dCDN), the dCDN should present to the end-user client, a
short-term certificate bound to the master certificate.

~~~
dCDN                  uCDN             Content Provider           CA
 |              ACME/STAR proxy        ACME/STAR client    ACME/STAR srv
 |                     |                     |                     |
 | 1. GET Metadata incl. Delegation Method object with CSR template|
 +-------------------->|                     |                     |
 | 200 OK + Metadata incl. CSR template [CDNI]                     |
 |<--------------------+                     |                     |
 | 2. Request delegation: video.dcdn.example + dCDN public key     |
 +-------------------->|                     |                     |
 |                     | 3. Request STAR Cert + dCDN public key    |
 |                     +-------------------->| 4. Request STAR cert|
 |                     |                     |    + Pubkey         |
 |                     |                     |-------------------->|
 |                     |                     | 5. STAR certificate |
 |                     | 6. STAR certificate |<--------------------|
 | 7. STAR certificate |<--------------------+                     |
 +<--------------------|                     |                     |
 |                     |                     |                     |
 | 8. Retrieve STAR certificate (credential-location-uri)          |
 +---------------------------------------------------------------->|
 |                     |                     |         9. renew +--|
 |                     |                     |            cert  |  |
 | 10. Star certificate                      |                  +->|
 |<----------------------------------------------------------------+
 |  ...                |                     |                     |
~~~
{: #fig-call-flow artwork-align="center"
   title="Example call-flow of STAR delegation in CDNI showing 2 levels of delegation"}

* Property: acme-delegations

* Description: an array of delegation objects associated with the dCDN account on the uCDN ACME server (see Section 2.3.1 of {{RFC9115}} for the details).

* Type: Objects

* Mandatory-to-Specify: Yes

# IANA Considerations

This document requests the registration of the following entries under the
"CDNI Payload Types" registry hosted by IANA regarding "CDNI delegation":

| Payload Type | Specification |
|---
| MI.AcmeStarDelegationMethod | {{&SELF}} |
| FCI.SupportedDelegationMethods| {{&SELF}} |

[^to-be-removed]

[^to-be-removed]: RFC Editor: please replace {{&SELF}} with the RFC number of this RFC and remove this note.

## CDNI MI AcmeStarDelegationMethod Payload Type

Purpose:
: The purpose of this Payload Type is to distinguish AcmeStarDelegationMethod
  MI objects (and any associated capability advertisement)

Interface:
: MI

Encoding:
: See {{acmedeleobj}}

## CDNI FCI SupportedDelegationMethods Payload Type

Purpose:
: The purpose of this Payload Type is to distinguish SupportedDelegationMethods
  FCI objects (and any associated capability advertisement)

Interface:
: FCI

Encoding:
: See {{metadata}}

# Security considerations

Delegation metadata proposed here do not alter nor change Security
Considerations as outlined in the following RFCs: An Automatic Certificate
Management Environment (ACME) Profile for Generating Delegated Certificates
{{RFC9115}}; the CDNI Metadata {{RFC8006}} and CDNI Footprint and Capabilities
{{RFC8008}}.

--- back

# Acknowledgments
{:unnumbered}

TODO
