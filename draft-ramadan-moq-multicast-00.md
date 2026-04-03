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
   discovery and multi-path delivery across TV, mobile, and browser
   platforms.  All multicast delivery uses MMTP packets — the same
   packet format used on MoQ QUIC streams and datagrams — providing
   track routing, timestamps, sequencing, FEC metadata, and
   authentication natively.

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
       1.1.  Multicast as Supplement to MoQ Unicast
   2.  Terminology
   3.  Delivery Paths
   4.  Multicast Catalog Extension
       4.1.  Multicast Endpoint Format
       4.2.  Network Source Types
             4.2.1.  AMT (Automatic Multicast Tunneling)
             4.2.2.  ATSC 3.0 (Broadcast Television)
             4.2.3.  Multiple Network Sources
   5.  Multicast Packet Format
   6.  Multi-Path Delivery
       6.1.  Packaging Negotiation
   7.  Security Considerations
       7.1.  Multicast Security
       7.2.  Content Authentication
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
3. **MMTP wire format**: All multicast delivery uses MMTP packets —
   the same packet format used on MoQ QUIC streams and datagrams.
   MMTP provides track routing (packet_id), timestamps, sequencing
   (PSN), FEC metadata (FEC Type, OTI), random access signaling
   (RAP), and fragmentation (C flag) natively.
4. **Multi-path delivery**: Combining MoQ unicast and multicast for
   seamless failover and FEC symbol deduplication

MoQ relays MAY operate as TreeDN [RFC9706] nodes for hierarchical
distribution.

### 1.1. Multicast as Supplement to MoQ Unicast

Multicast delivery as defined in this document is a supplement to
MoQ/QUIC unicast.  Subscribers MUST establish a MoQ session first
to receive the catalog, negotiate parameters, and receive initial
media.  Multicast MAY then be used as an optimized delivery path
for ongoing media data.  This ensures that codec configuration is
available before multicast reception begins.

Exception: MMTP-packaged streams [I-D.ramadan-moq-mmt] delivered
via ATSC 3.0 broadcast or native SSM are self-describing and MAY
operate as unidirectional data streams without a MoQ session.  MMTP
carries per-packet routing (packet_id), timing (timestamp),
sequencing (Packet Sequence Number), FEC metadata (FEC Type, OTI),
and signaling (PA, MPI messages) natively.  ATSC 3.0 and ARIB
STD-B60 receivers consume MMTP over SSM as their native delivery
path.

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

**MMTP**: MMT Protocol — the packet layer of MPEG Media Transport
([I-D.bouazizi-mmtp] Section 3).  Each MMTP packet is
self-describing, carrying track routing, timestamps, sequencing,
and FEC metadata natively.

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

Multiple transports MAY be available for a given stream.  Applications
select transports based on availability and local policy.

For broadcast deployments with Single Frequency Network (SFN)
diversity, the receiver transport preference order is: SFN diversity
reception, single transmitter, native SSM, AMT tunneling, MoQ/QUIC
unicast.

## 4. Multicast Catalog Extension

For tracks available via multicast, the MoQ catalog includes a
top-level `multicast` field containing an `endpoints` array for
endpoint discovery.

### 4.1. Multicast Endpoint Format

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
      "tracks": [
        { "name": "video",        "packetId": 1 },
        { "name": "video/repair", "packetId": 2 },
        { "name": "audio",        "packetId": 3 },
        { "name": "audio/repair", "packetId": 4 }
      ],
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
        "tracks": [
          { "name": "video",        "packetId": 1 },
          { "name": "video/repair", "packetId": 2 }
        ]
      },
      {
        "sourceAddress": "10.0.0.1",
        "groupAddress": "232.1.1.2",
        "port": 5004,
        "tracks": [
          { "name": "audio", "packetId": 3 }
        ]
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

**tracks** (array, REQUIRED): Tracks available on this endpoint.
  Each element is an object with the following fields:

  - **name** (string, REQUIRED): Track name corresponding to the
    track identifier used in the MoQ catalog.
  - **packetId** (integer, REQUIRED): MMTP packet_id used for
    packet-level track routing on multicast.  Maps directly to the
    Packet ID field in the MMTP header (Section 3.1 of
    [I-D.ramadan-moq-mmt]).  Values MUST be unique within an
    (sourceAddress, groupAddress, port) tuple.

**bandwidth** (integer, RECOMMENDED): Aggregate bandwidth of this
  endpoint in bits per second.  Publishers SHOULD include bandwidth
  to enable capacity-aware join decisions.  Subscribers SHOULD check
  available network capacity before joining high-bandwidth groups.

**networkSource** (object or array, OPTIONAL): Network delivery
  configuration describing how subscribers can reach the multicast
  stream when native IP multicast routing is not available.  May
  appear on individual endpoints or at the top-level `multicast`
  object to apply to all endpoints.  See Section 4.2.

Each (sourceAddress, groupAddress, port) multicast tuple MUST be
associated with at most one MoQ namespace.  Publishers requiring
multiple independent streams MUST use distinct multicast groups or
ports.  This constraint ensures that MMTP packet_id values are
unambiguous within a multicast group.

Subscribers receiving a catalog with multicast endpoints MAY
auto-connect to the multicast group when multicast APIs are
available and multicast delivery is preferred (IWA DirectSocket,
native UDP sockets, AMT tunneling).  Multicast joins create
IGMP/MLD state on intermediate routers; subscribers SHOULD only
join when multicast provides a concrete benefit over unicast.
This enables a seamless upgrade path: subscribers first connect via
MoQ/QUIC to receive the catalog and media, then optionally switch
to multicast for lower-latency, FEC-protected delivery.

During multicast join (which may take 1-3 seconds for IGMP/MLD),
subscribers SHOULD continue receiving via MoQ/QUIC.  Once multicast
data arrives, subscribers switch to the multicast path.  This
dual-path startup avoids join latency gaps.

If multicast reception fails or degrades, subscribers fall back
to MoQ/QUIC unicast by subscribing with filter LatestGroup or
NextGroup.  No multicast-to-MoQ coordinate mapping is needed —
the relay provides the current group position.

Receivers SHOULD implement hysteresis to prevent flapping between
multicast and unicast paths.  Switch away from multicast after
sustained loss or consecutive FEC block failures; switch back only
after multicast reception is stable for a sufficient period.
Specific thresholds are implementation-defined and SHOULD be
tunable.

### 4.2. Network Source Types

The `networkSource` field describes how subscribers can reach the
multicast stream when native IP multicast routing is not available.
It appears at the `multicast` level (applying to all endpoints) or
on individual endpoints.

**type** (string, REQUIRED): Delivery technology identifier.

Defined types:

#### 4.2.1. AMT (Automatic Multicast Tunneling)

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

Note: DRIAD requires TYPE260 reverse DNS records on the source IP.
DRIAD relay addresses are typically anycast — subscribers connect
to the topologically nearest relay without additional configuration.
Publishers without reverse DNS control SHOULD provide the `relay`
field directly.

#### 4.2.2. ATSC 3.0 (Broadcast Television)

```json
{
  "networkSource": {
    "type": "atsc3",
    "frequency": 533000,
    "plpId": 0,
    "serviceId": 1,
    "slsUri": "https://example.com/atsc3/sls/service1.xml"
  }
}
```

**frequency** (integer, REQUIRED): RF center frequency in kHz.

**plpId** (integer, OPTIONAL): Physical Layer Pipe ID.
  Defaults to 0 (base PLP).

**serviceId** (integer, OPTIONAL): Service ID within the
  broadcast multiplex.

**slsUri** (string, OPTIONAL): URI for Service Layer Signaling (SLS)
  bootstrap.  Enables receivers joining via IP (not RF) to acquire
  the full ATSC 3.0 SLS including S-TSID with FEC parameters.

Receivers with ATSC 3.0 tuner hardware can receive the stream
directly over RF.  MoQ subscribers without tuner hardware ignore
this networkSource type.

#### 4.2.3. Multiple Network Sources

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
transport hierarchy defined in Section 3.

## 5. Multicast Packet Format

All multicast delivery uses MMTP packets.  Each UDP datagram carries
one MMTP packet — the same packet format used on MoQ QUIC streams
and datagrams.  MMTP provides track routing (packet_id), timestamps,
sequencing (Packet Sequence Number), FEC metadata (FEC Type, Source/
Repair FEC Payload ID, per-packet OTI), random access signaling
(RAP flag), and fragmentation (C flag) natively.

LOC video objects [I-D.ietf-moq-loc] and CMAF chunks
[I-D.ietf-moq-cmsf] are frame-sized (10-100KB+) and exceed the
UDP datagram MTU (~1300 bytes).  Per [I-D.ietf-moq-loc] Section
4.1: "When mapped to QUIC datagrams, each object must fit entirely
within a QUIC datagram."  The same constraint applies to multicast
UDP datagrams.  MMTP [I-D.bouazizi-mmtp] fragments media into
MTU-sized packets natively per Section 4.1.1.  This is not a design
choice — it is a physical constraint of datagram-based delivery.

No additional multicast framing, encapsulation, or header format is
needed.  The MMTP packet format is defined in [I-D.ramadan-moq-mmt]
Section 3.1 and [I-D.bouazizi-mmtp] Section 3.

This design means the same MMTP packet can be delivered via three
transports without modification:

| Transport | Encapsulation |
|-----------|---------------|
| Reliable QUIC stream | MMTP packet as MoQ stream object |
| QUIC datagram | MMTP packet as MoQ datagram object |
| Multicast UDP | MMTP packet as UDP datagram |

Receivers on multicast demultiplex packets using the MMTP packet_id
field, which maps to the `packetId` assigned in the multicast
catalog endpoint (Section 4.1).  FEC source and repair packets are
distinguished by the MMTP FEC Type field (1=source, 2=repair).

For ATSC 3.0 and ARIB STD-B60 broadcast receivers, MMTP over SSM
is the native delivery path.  No format conversion is needed — MoQ
relays at network edges bridge the same MMTP packets between
multicast and QUIC transports.

## 6. Multi-Path Delivery

The same media content (source and repair) can be transmitted over
both MoQ unicast (QUIC/WebTransport) and multicast (UDP/SSM/AMT)
paths simultaneously.  Because the same MMTP packets are used on all
transports, receivers can combine symbols from any path for FEC
recovery.

When symbols arrive from multiple paths simultaneously, receivers:

1. MUST deduplicate symbols using SBN+ESI as the unique key
   within a single (namespace, track_name) tuple
2. MAY combine source symbols from reliable MoQ/QUIC with repair
   symbols from lossy multicast — skipping FEC when all source
   symbols arrive reliably

Multicast-to-unicast failover: if multicast reception fails,
subscribers fall back to MoQ/QUIC unicast by subscribing with
filter LatestGroup or NextGroup per [I-D.ietf-moq-transport].
No coordinate mapping between multicast SBN and MoQ group_id is
needed — the relay provides the current position.

### 6.1. Packaging Negotiation

Packaging negotiation is implicit in MoQ: subscribers subscribe to
tracks by name, and the catalog advertises packaging per track.  A
subscriber that only supports LOC subscribes to the LOC track; a
subscriber that supports MMTP subscribes to the MMTP track.  The
`altGroup` catalog field enables publishers to offer the same
content in multiple packaging formats.
No explicit packaging capability negotiation is needed.

## 7. Security Considerations

### 7.1. Multicast Security

SSM inherently limits traffic to authorized sources via (S,G)
filtering.  Sequence numbers enable replay detection.  For AMT,
trust is delegated to the relay per [RFC7450].

### 7.2. Content Authentication

Multicast UDP lacks QUIC's integrity guarantees.  For MMTP-packaged
multicast delivery, content authentication uses the MMTP
signed_mmt_message mechanism per [I-D.bouazizi-mmtp] Section 3.1
(header extension format).  This provides per-packet authentication
using digital signatures carried in MMTP header extensions.

Key distribution for multicast authentication uses one of two
signaling paths:

1. **MMTP PA (Package Access) messages**: Certificates are carried
   in PA messages published on the MoQ signaling track.  PA
   messages are MMTP signaling packets (Packet Type = signaling)
   that carry security-related tables.

2. **MoQ catalog `auth` extension**: Certificates or certificate
   URLs are included in the catalog's `auth` field, enabling
   pre-join key discovery.

The specific key distribution mechanism is out of scope for this
document, but the signaling paths above provide the transport.
Publishers SHOULD rotate signing keys periodically and deliver new
certificates via PA messages.  Receivers that miss a PA message can
request the current certificate via the MoQ signaling track.

The authentication configuration is declared in the catalog as part
of the multicast configuration:

```json
{
  "multicast": {
    "auth": {
      "scheme": "signed_mmt_message"
    },
    "endpoints": [...]
  }
}
```

**scheme** (string, REQUIRED if auth present): Authentication
  mechanism identifier.  Defined values:
  - "signed_mmt_message": MMTP-native per-packet authentication
    per [I-D.bouazizi-mmtp] Section 3.1
  - "alta": ALTA per [I-D.krose-mboned-alta] — lightweight
    asymmetric loss-tolerant authentication, carried in MMTP
    header extensions

On MoQ unicast, end-to-end content authentication MAY be provided
by Secure Objects which provides object-level encryption and
authentication independent of multicast.

Content authentication for multicast is an active area of work.
Deployments SHOULD track developments in [I-D.krose-mboned-alta]
and related specifications.

## 8. IANA Considerations

This document has no IANA actions.  The multicast catalog extension
(Section 4) is defined by this document and does not require IANA
registration.

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

[I-D.bouazizi-mmtp]
           Bouazizi, I., "MMT Protocol (MMTP)",
           draft-bouazizi-mmtp-01 (work in progress).

### 9.2. Informative References

[I-D.ramadan-moq-mmt]
           Ramadan, O., "MPEG Media Transport (MMT) Packaging for
           Media over QUIC", draft-ramadan-moq-mmt (work in progress).

[I-D.ramadan-moq-fec]
           Ramadan, O., "Forward Error Correction for Media over QUIC",
           draft-ramadan-moq-fec (work in progress).

[I-D.ietf-moq-catalogformat]
           Nandakumar, S., et al., "Common Catalog Format for
           MoQ", draft-ietf-moq-catalogformat (work in progress).

[I-D.krose-mboned-alta]
           Krose, B., "Asymmetric Loss-Tolerant Authentication",
           draft-krose-mboned-alta (work in progress).

[I-D.ietf-moq-loc]
           Zanaty, M., et al., "Low Overhead Media Container",
           draft-ietf-moq-loc (work in progress).

[I-D.ietf-moq-cmsf]
           Law, W., "CMSF: A CMAF Compliant Implementation of
           MOQT Streaming Format", draft-ietf-moq-cmsf
           (work in progress).

[ISO.23008-1]
           ISO, "Information technology - High efficiency coding and
           media delivery in heterogeneous environments - Part 1:
           MPEG media transport (MMT)", ISO/IEC 23008-1:2023.
           (Informative.  A freely available description of the MMTP
           wire format is provided by [I-D.bouazizi-mmtp].)

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
