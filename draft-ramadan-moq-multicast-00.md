# Multicast Delivery and Endpoint Discovery for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: October 2026                                      April 2026

     Multicast Delivery and Endpoint Discovery for Media over QUIC
                    draft-ramadan-moq-multicast-00

Abstract

   This document specifies multicast delivery mechanisms and catalog-
   based endpoint discovery for Media over QUIC (MoQ).  It defines how
   MoQ sessions integrate with IP multicast (SSM, ASM), Automatic
   Multicast Tunneling (AMT), and TreeDN for scalable live streaming.
   The specification includes a multicast catalog extension for endpoint
   discovery, and delivery path selection across TV, mobile, and browser
   platforms.  This document is container-format agnostic and is
   referenced by both [I-D.ramadan-moq-mmt] and [I-D.ramadan-moq-fec].

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

   Internet-Drafts are working documents of the Internet Engineering
   Task Force (IETF).  Note that other groups may also distribute
   working documents as Internet-Drafts.  The list of current
   Internet-Drafts is at https://datatracker.ietf.org/drafts/current/.

   Internet-Drafts are draft documents valid for a maximum of six
   months and may be updated, replaced, or obsoleted by other documents
   at any time.  It is inappropriate to use Internet-Drafts as
   reference material or to cite them other than as "work in progress."

Copyright Notice

   Copyright (c) 2026 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

Table of Contents

   1.  Introduction
   2.  Terminology
   3.  Delivery Paths
   4.  Transport Hierarchy
   5.  Multicast Catalog Extension
       5.1.  Multicast Endpoint Format
       5.2.  Network Source Types
             5.2.1.  AMT (Automatic Multicast Tunneling)
             5.2.2.  ATSC 3.0 (Broadcast Television)
             5.2.3.  Multiple Network Sources
   6.  Multicast Packet Formats
   7.  Security Considerations
       7.1.  Multicast Security
   8.  IANA Considerations
   9.  References
       9.1.  Normative References
       9.2.  Informative References
   Authors' Addresses
```

## 1. Introduction

Live streaming audiences routinely reach tens of millions of
concurrent viewers.  Traditional HTTP-based CDN architectures
replicate streams at x86 server farms close to viewers, but this
approach does not scale cost-effectively for 4K/8K/360-degree
bitrates.

IP multicast provides an alternative where a single stream is
replicated at the network layer, but lacks the economic incentives
for widespread ISP deployment.  Media over QUIC (MoQ)
[I-D.ietf-moq-transport] provides a modern transport for real-time
media, but its unicast model faces the same scaling limitations as
HTTP CDNs for mass-audience events.

This document specifies how MoQ integrates with IP multicast to
combine the scalability of multicast with the reliability and
signaling of QUIC:

1. **Delivery paths**: Platform-specific multicast reception for
   TV, mobile, and browser clients
2. **Catalog extension**: A container-agnostic multicast endpoint
   discovery mechanism for MoQ catalogs [I-D.ietf-moq-catalogformat]

MoQ relays MAY operate as TreeDN [RFC9706] nodes for hierarchical
distribution.

This specification is container-format agnostic.  It works with
MMT [I-D.ramadan-moq-mmt], CMAF [I-D.ietf-moq-cmsf], LOC
[I-D.ietf-moq-loc], or any other MoQ packaging format.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY",
and "OPTIONAL" in this document are to be interpreted as described
in BCP 14 [RFC2119] [RFC8174].

**SSM**: Source-Specific Multicast [RFC4607]

**AMT**: Automatic Multicast Tunneling [RFC7450]

**DRIAD**: DNS Reverse IP AMT Discovery [RFC8777]

**TreeDN**: Tree-based Content Delivery Network [RFC9706]

**IWA**: Isolated Web App — a Chrome packaging format that grants
DirectSocket API access

## 3. Delivery Paths

MoQ content can reach receivers via multiple delivery paths depending
on platform capabilities:

| Client Type | Multicast Path |
|-------------|----------------|
| Native (TV, mobile) | SSM direct via OS multicast API |
| Native (no multicast) | AMT tunneling over UDP [RFC7450] |
| Browser (IWA) | SSM/AMT via DirectSocket [WICG-DirectSockets] |
| Browser (standard) | MoQ/WebTransport (unicast) |

Both native SSM and AMT tunneling require UDP socket access.
MoQ/WebTransport is the universal fallback and requires no
special platform capabilities.

Broadcast receivers (ATSC 3.0, ARIB STD-B60) consume MMTP streams
natively over RF tuner hardware.  MoQ integration occurs at the
gateway/head-end level via TreeDN [RFC9706] and AMT [RFC7450] /
DRIAD [RFC8777].

## 4. Transport Hierarchy

Multiple transports MAY be available for a given stream.  The catalog
signals which transports are offered via the multicast endpoint format
(Section 5) and standard MoQ subscription.  Applications select
transports based on availability and local policy.

## 5. Multicast Catalog Extension

For tracks available via multicast, the MoQ catalog includes a
top-level `multicast` field containing an `endpoints` array for
endpoint discovery.

### 5.1. Multicast Endpoint Format

The `multicast` field is a catalog extension per
[I-D.ietf-moq-catalogformat] Section 3.1.  Parsers that do not
support multicast MUST ignore it.

The `multicast` field contains an `endpoints` array listing one or
more multicast groups.  A single-endpoint deployment uses a
one-element array:

```json
{
  "multicast": {
    "endpoints": [{
      "sourceAddress": "69.25.95.10",
      "groupAddress": "232.0.10.1",
      "port": 8000,
      "tracks": ["video", "video/repair", "audio"],
      "bandwidth": 6500000
    }]
  }
}
```

Multi-group deployments (e.g., per-quality ABR tiers or separate
audio/video groups) use multiple elements:

```json
{
  "multicast": {
    "endpoints": [
      {
        "sourceAddress": "10.0.0.1",
        "groupAddress": "232.1.1.1",
        "port": 5004,
        "tracks": ["video", "video/repair"]
      },
      {
        "sourceAddress": "10.0.0.1",
        "groupAddress": "232.1.1.2",
        "port": 5004,
        "tracks": ["audio"]
      }
    ],
    "networkSource": {
      "type": "amt",
      "discovery": "driad",
      "relay": "amt.example.com"
    }
  }
}
```

Endpoint field definitions:

**protocol** (string, OPTIONAL): Transport protocol.
  - "ssm": Source-Specific Multicast (RFC 4607).  This is the default
    when `sourceAddress` is present.
  - "asm": Any-Source Multicast.

**sourceAddress** (string, OPTIONAL): SSM source IP address per
  [RFC4607].  Required for Source-Specific Multicast.  If omitted,
  implies ASM.

**groupAddress** (string, REQUIRED): Multicast group address.
  - SSM range: 232.0.0.0/8 (IPv4), ff3x::/32 (IPv6)
  - ASM range: 224.0.0.0/4 (IPv4), ff0x::/16 (IPv6)

**port** (integer, REQUIRED): UDP port number.

**tracks** (array, REQUIRED): Track names available on this endpoint.
  Track names correspond to the track identifiers used in the MoQ
  catalog renditions.

**bandwidth** (integer, OPTIONAL): Aggregate bandwidth of this endpoint in bits per second. Subscribers SHOULD check available network capacity before joining high-bandwidth multicast groups.

**networkSource** (object or array, OPTIONAL): Network delivery
  configuration describing how subscribers can reach the multicast
  stream when native IP multicast routing is not available.  May
  appear on individual endpoints or at the top-level `multicast`
  object to apply to all endpoints.  See Section 5.2.

Subscribers receiving a catalog with multicast endpoints SHOULD
auto-connect to the multicast group when multicast APIs are
available (IWA DirectSocket, native UDP sockets, AMT tunneling).
This enables a seamless upgrade path: subscribers first connect via
MoQ/QUIC to receive the catalog and media, then optionally switch
to multicast for lower-latency, FEC-protected delivery.

During multicast join (which may take 1-3 seconds for IGMP/MLD), subscribers SHOULD continue receiving via MoQ/QUIC. Once multicast data arrives, subscribers switch to the multicast path. This dual-path startup avoids join latency gaps.

### 5.2. Network Source Types

The `networkSource` field describes how subscribers can reach the
multicast stream when native IP multicast routing is not available.
It appears at the `multicast` level (applying to all endpoints) or
on individual endpoints.

**type** (string, REQUIRED): Delivery technology identifier.

Defined types:

#### 5.2.1. AMT (Automatic Multicast Tunneling)

```json
{
  "networkSource": {
    "type": "amt",
    "relay": "69.25.95.1",
    "discovery": "driad"
  }
}
```

**relay** (string, OPTIONAL): AMT relay address (IP or hostname).
  When present, subscribers SHOULD connect directly to this relay.

**discovery** (string, OPTIONAL): AMT relay discovery method.
  - "driad": DNS Reverse IP AMT Discovery per [RFC8777].
    Subscribers query the source IP's reverse DNS for AMT relay
    records.  This is the RECOMMENDED discovery method.
  - "manual": Relay address is provided in the `relay` field.

Subscribers SHOULD attempt relay discovery in this order:
1. Use `relay` directly if provided
2. Use DRIAD discovery on the SSM source IP if `discovery` is
   "driad" or omitted
3. Fall back to MoQ/QUIC unicast

Note: DRIAD requires the SSM source IP to have a reverse DNS delegation with TYPE260 records. Publishers using cloud-hosted sources that lack reverse DNS control SHOULD provide the `relay` field as a direct fallback.

#### 5.2.2. ATSC 3.0 (Broadcast Television)

```json
{
  "networkSource": {
    "type": "atsc3",
    "frequency": 533000,
    "plpId": 0,
    "serviceId": 1
  }
}
```

**frequency** (integer, REQUIRED): RF center frequency in kHz.

**plpId** (integer, OPTIONAL): Physical Layer Pipe ID.
  Defaults to 0 (base PLP).

**serviceId** (integer, OPTIONAL): Service ID within the
  broadcast multiplex.

This type indicates the multicast stream is available via an
ATSC 3.0 broadcast signal [ATSC-A331].  Receivers with ATSC 3.0
tuner hardware (USB dongles, built-in tuners, or NextGen TV
sets) can receive the stream directly over RF without Internet
connectivity.

Note: These tuning parameters are intended for receivers with ATSC 3.0 tuner hardware. MoQ subscribers without tuner hardware ignore this networkSource type and use AMT or MoQ/QUIC instead.

#### 5.2.3. Multiple Network Sources

When a stream is available via multiple delivery technologies,
`networkSource` MAY be an array:

```json
{
  "networkSource": [
    { "type": "amt", "relay": "69.25.95.1", "discovery": "driad" },
    { "type": "atsc3", "frequency": 533000, "plpId": 0 }
  ]
}
```

Subscribers select the highest-priority available source per the
transport hierarchy defined in Section 4.

## 6. Multicast Packet Formats

Each MoQ packaging type defines its own multicast UDP wire format. The catalog's `packaging` field tells receivers which parser to use: MMTP packets per [I-D.ramadan-moq-mmt], LOC objects per [I-D.ietf-moq-loc], or condensed packets per [I-D.ramadan-moq-fec] Section 7.4.

## 7. Security Considerations

### 7.1. Multicast Security

SSM inherently limits traffic to authorized sources via (S,G)
filtering.  For additional security:

- **DTLS-SRTP**: Encrypt media payloads when confidentiality is
  required over multicast
- **Source Authentication**: Validate source IP at AMT relay;
  for SSM, receivers verify (S,G) membership
- **FEC Integrity**: FEC decoder inconsistencies can detect injection
  attacks; receivers SHOULD verify decoded media structure
- **Replay Protection**: Sequence numbers enable detection of
  replayed or reordered packets
- **AMT Relay Trust**: For AMT, trust is delegated to the relay's
  authentication mechanisms per [RFC7450]
- **Content Authentication**: For content integrity across untrusted relay chains, publishers MAY include an Auth Tag MoQ extension header per [I-D.ramadan-moq-fec] on source track objects. The auth tag (e.g., ALTA) enables receivers to verify that media content has not been modified in transit, complementing SSM source address validation and QUIC hop-by-hop integrity. The authentication scheme and tag size are signaled in the catalog.

## 8. IANA Considerations

This document has no IANA actions.  The multicast catalog extension
(Section 5) is a JSON schema extension to MoQ catalogs and does not
require IANA registration.

## 9. References

### 9.1. Normative References

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119,
           DOI 10.17487/RFC2119, March 1997.

[RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
           2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
           May 2017.

[RFC4607]  Holbrook, H. and B. Cain, "Source-Specific Multicast for IP",
           RFC 4607, DOI 10.17487/RFC4607, August 2006.

[RFC7450]  Bumgardner, G., "Automatic Multicast Tunneling", RFC 7450,
           DOI 10.17487/RFC7450, February 2015.

[RFC8777]  Holland, J., "DNS Reverse IP Automatic Multicast Tunneling
           (AMT) Discovery", RFC 8777, DOI 10.17487/RFC8777, April 2020.

[RFC9706]  Holland, J., et al., "TreeDN: Tree-Based Content Delivery
           Network (CDN) for Live Streaming to Mass Audiences",
           RFC 9706, December 2024.

[I-D.ietf-moq-transport]
           Curley, L., Pugin, K., Nandakumar, S., Vasiliev, V., and
           I. Swett, "Media over QUIC Transport",
           draft-ietf-moq-transport (work in progress).

### 9.2. Informative References

[I-D.ramadan-moq-mmt]
           Ramadan, O., "MPEG Media Transport (MMT) Packaging for
           Media over QUIC", draft-ramadan-moq-mmt (work in progress).

[I-D.ramadan-moq-fec]
           Ramadan, O., "Forward Error Correction for Media over QUIC",
           draft-ramadan-moq-fec (work in progress).

[I-D.ietf-moq-cmsf]
           Law, W., "CMSF: A CMAF Compliant Implementation of
           MOQT Streaming Format", draft-ietf-moq-cmsf
           (work in progress).

[I-D.ietf-moq-loc]
           Zanaty, M., et al., "Low Overhead Media Container",
           draft-ietf-moq-loc (work in progress).

[I-D.ietf-moq-catalogformat]
           Nandakumar, S., et al., "Common Catalog Format for
           MoQ", draft-ietf-moq-catalogformat (work in progress).

[ATSC-A331]
           ATSC, "Signaling, Delivery, Synchronization, and Error
           Protection", A/331:2025, February 2025.

[WICG-DirectSockets]
           Rayskiy, A., "Direct Sockets API", W3C Community Group
           Draft Report, https://wicg.github.io/direct-sockets/

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
