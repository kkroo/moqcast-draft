# Multicast Delivery and Endpoint Discovery for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: January 2027                                       July 2026

     Multicast Delivery and Endpoint Discovery for Media over QUIC
                    draft-ramadan-moq-multicast-00

Abstract

   This document specifies multicast delivery mechanisms and catalog-
   based endpoint discovery for Media over QUIC (MoQ).  It defines how
   MoQ sessions integrate with IP multicast (SSM, ASM), Automatic
   Multicast Tunneling (AMT), and TreeDN for scalable live streaming.
   The specification includes a multicast catalog extension with simple
   and extended formats for endpoint discovery, delivery path selection
   across TV, mobile, and browser platforms, and incentive models for
   incremental multicast infrastructure deployment.  This document is
   container-format agnostic and is referenced by both
   [I-D.ramadan-moq-mmt] and [I-D.ramadan-moq-fec].

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
   4.  SSM Group Allocation
   5.  TreeDN Integration
       5.1.  ISP Router AMT Deployment
       5.2.  DePIN Incentives for Multicast Infrastructure
       5.3.  IWA Home Gateway Architecture
   6.  Transport Hierarchy
   7.  Multicast Catalog Extension
       7.1.  Simple Multicast Format
       7.2.  Extended Multicast Format
       7.3.  Format Detection
       7.4.  Network Source Types
             7.4.1.  AMT (Automatic Multicast Tunneling)
             7.4.2.  ATSC 3.0 (Broadcast Television)
             7.4.3.  DVB (Digital Video Broadcasting)
             7.4.4.  5G Broadcast (3GPP MBS)
             7.4.5.  Multiple Network Sources
   8.  Multicast Packet Formats
       8.1.  MMTP Multicast
       8.2.  LOC Multicast
       8.3.  Condensed Multicast
   9.  Security Considerations
       9.1.  Multicast Security
   10. IANA Considerations
   11. References
       11.1. Normative References
       11.2. Informative References
   Appendix A.  Catalog Examples
       A.1.  Simple Catalog (Single Multicast Endpoint)
       A.2.  Extended Catalog (Multiple Endpoints)
   Authors' Addresses
```

## 1. Introduction

Live streaming audiences routinely reach tens of millions of
concurrent viewers, with events like Thursday Night Football
consuming roughly a quarter of all Internet traffic on broadcast
nights [JUNIPER-TREEDN].  Traditional HTTP-based CDN architectures
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
2. **TreeDN integration**: Hierarchical multicast distribution
   trees with AMT relay bridging per [RFC9706]
3. **Catalog extension**: A container-agnostic multicast endpoint
   discovery mechanism for MoQ catalogs
4. **Incentive architecture**: DePIN token models for incremental
   multicast infrastructure deployment

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

**DePIN**: Decentralized Physical Infrastructure Network — a token-
incentive model for infrastructure deployment

**MBone**: Multicast Backbone — the original Internet multicast
infrastructure, decommissioned circa 2000

## 3. Delivery Paths

MoQ content can reach receivers via multiple delivery paths depending
on platform capabilities:

| Client Type | Multicast Path | Requirements |
|-------------|----------------|--------------|
| TV (ATSC 3.0 / ARIB) | SSM direct (tuner) | ATSC 3.0 or ARIB receiver hardware |
| Android (native) | SSM via MulticastSocket | MulticastLock + CHANGE_WIFI_MULTICAST_STATE |
| Android (5G Broadcast) | 3GPP MBS [3GPP-MBS] | Qualcomm LTE Broadcast SDK or 5G-MAG middleware |
| iOS (native) | SSM via NWConnectionGroup | com.apple.developer.networking.multicast entitlement |
| Browser (IWA enterprise) | SSM/AMT via DirectSocket | Admin-managed Chrome policy |
| Browser (IWA DePIN) | SSM/AMT via DirectSocket | User-installed IWA (token-incentivized) |
| Browser (IWA power user) | SSM/AMT via DirectSocket | Side-loaded IWA (chrome://flags) |
| Browser (standard) | MoQ/WebTransport | None (production-ready path) |

Both native SSM and AMT tunneling require UDP socket access.

**TV (ATSC 3.0 / ARIB)**: Broadcast TV receivers consume ROUTE/ALC
or MMTP streams natively over RF tuner hardware.  These devices
implement FEC decoding and S-TSID parsing per [ATSC-A331].  MoQ
integration occurs at the gateway/head-end level via TreeDN [RFC9706]
and AMT [RFC7450] / DRIAD [RFC8777].

**Android**: Android provides MulticastSocket with MulticastLock for
Wi-Fi multicast reception.  For 5G Broadcast, the device modem
provides carrier-grade multicast via the Qualcomm LTE Broadcast SDK
or 5G-MAG MBMS middleware [5GMAG-LIBFLUTE].

**iOS**: iOS supports multicast via NWConnectionGroup (iOS 14+)
with an Apple-granted entitlement.

**Browser (IWA)**: The IWA (Isolated Web App) DirectSocket API
[WICG-DirectSockets] provides UDPSocket with joinGroup()/leaveGroup()
for SSM and ASM multicast reception.  IWAs are NOT distributable via
the Chrome Web Store and require either enterprise Chrome admin policy,
Google allowlisting, or manual user installation (chrome://flags or
side-loading).  This limits mainstream consumer reach but serves three
viable audiences: (1) enterprise/kiosk/signage deployments with managed
Chrome, (2) DePIN (Decentralized Physical Infrastructure) networks where
token incentives motivate users to install IWAs for higher-quality
multicast reception, and (3) technical power users who opt in to IWA
installation for direct multicast access.

**Browser (standard)**: MoQ over WebTransport is the production-ready
path for standard web browsers.  WebCodecs provides sub-100ms
decode latency on Chrome/Edge; MSE provides Safari/Firefox fallback.
This path requires no special permissions or entitlements.

## 4. SSM Group Allocation

For multi-bitrate ABR streams, each quality level maps to a
separate SSM group:

| Track | SSM Group | Port | Bitrate |
|-------|-----------|------|---------|
| video_4k | 232.1.1.1 | 5004 | 15 Mbps |
| video_1080p | 232.1.1.2 | 5004 | 8 Mbps |
| video_720p | 232.1.1.3 | 5004 | 4 Mbps |
| video_480p | 232.1.1.4 | 5004 | 1.5 Mbps |
| video_4k/repair | 232.1.3.1 | 5005 | ~2 Mbps |
| video_1080p/repair | 232.1.3.2 | 5005 | ~1 Mbps |

ABR quality switches require SSM group changes (IGMP/MLD leave/join).

## 5. TreeDN Integration

For large-scale MoQ deployments, TreeDN [RFC9706] provides:

- Hierarchical multicast distribution trees
- AMT relay discovery via DRIAD [RFC8777]
- CDN interconnection semantics
- Replication-as-a-Service (RaaS) replacing x86 CDN servers

MoQ relays MAY operate as TreeDN nodes, forwarding packets
over both unicast (QUIC) and multicast (SSM/AMT) paths simultaneously.

When operating as a TreeDN node:
1. Relay receives media over SSM from upstream
2. Relay serves downstream via QUIC (MoQ) and/or SSM (multicast)
3. FEC repair is preserved or augmented per local conditions

### 5.1. ISP Router AMT Deployment

Thousands of ISP routers already deployed at network edges can serve
as AMT relays with minimal configuration, replacing racks of x86 CDN
servers with inline packet replication on existing routing silicon
[JUNIPER-TREEDN].  For example, AMT relay capability is built into
commodity routing chipsets and requires no additional licenses or
service blades — a single appliance performs the replication function
that previously demanded dedicated CDN server complexes.

This architecture offloads HTTP-based unicast replication from
origin and CDN servers to in-network routers that perform either:

- Native SSM multicast replication where the ISP has enabled PIM-SSM
- AMT unicast replication where multicast is not available, with the
  router acting as the AMT relay endpoint per [RFC7450]

In both cases, the MoQ relay at the edge terminates the multicast/AMT
path and serves clients over QUIC/WebTransport, bridging the ISP
multicast domain into the MoQ application layer.

### 5.2. DePIN Incentives for Multicast Infrastructure

Decentralized Physical Infrastructure Networks (DePIN) provide a
token-incentive model for operators and end users to grow multicast
infrastructure, effectively rebuilding the multicast backbone (MBone)
that was decommissioned in the early 2000s.

Mobile and TV platforms represent the largest opportunity for
token-incentivized multicast participation.  Android devices with
MulticastSocket, iOS devices with NWConnectionGroup, 5G Broadcast
receivers, and ATSC 3.0 tuners all have native multicast or
broadcast reception capabilities that can be rewarded:

- Mobile/TV users earn tokens for receiving content via multicast
  or broadcast instead of unicast, reducing origin and CDN load
- 5G Broadcast and ATSC 3.0 receivers contribute to network
  offload simply by tuning in, with tokens rewarding participation
- ISP operators earn tokens for enabling AMT relay on edge routers,
  where a single appliance replaces racks of CDN servers
  [JUNIPER-TREEDN]
- IWA browser nodes on wired desktops or Chromeboxes earn tokens
  for receiving multicast via DirectSocket and serving as local
  WebTransport gateways to wireless clients on the same network
  (see Section 5.3)

Content providers pay fewer tokens for multicast delivery than
unicast CDN, reflecting the order-of-magnitude cost reduction
demonstrated in TreeDN deployments [JUNIPER-TREEDN].

This creates a market-driven incentive to grow multicast
reachability incrementally, without requiring coordinated global
deployment.  Each additional AMT relay or multicast-capable
endpoint expands the delivery tree, reducing aggregate bandwidth
and cost for all participants.

### 5.3. IWA Home Gateway Architecture

An IWA node running on a wired-connected device (Chromebox, desktop,
or mini-PC) can serve as a local multicast anchor point.  The device
receives SSM or AMT multicast via DirectSocket over its wired
Ethernet connection, which provides the stable, high-throughput link
needed for sustained high-bitrate reception (4K, 8K, multi-view).

The IWA node then operates as a local MoQ relay, redistributing
content over WebTransport to wireless clients on the local network:

```
Internet (SSM/AMT)
       |
       v
 [ISP Router w/ AMT Relay]
       |  wired Ethernet
       v
 [Chromebox / Desktop IWA]  <-- DirectSocket multicast rx
       |  WebTransport (local)
       v
 [Phone] [Tablet] [Smart TV] [Laptop]
```

This architecture provides several advantages:

1. Wired reception avoids Wi-Fi jitter and supports higher bitrates
   than wireless multicast (which most consumer APs do not relay)
2. A single multicast stream serves the entire household regardless
   of the number of local viewers
3. Wireless clients use standard MoQ/WebTransport with no IWA
   requirement, no special permissions, and full browser compatibility
4. The IWA gateway can apply local FEC repair, reducing retransmission
   round trips to the origin

## 6. Transport Hierarchy

MoQ clients select the highest-priority available transport:

**Native clients (TV, mobile SDK)**:

| Priority | Transport | Availability |
|----------|-----------|-------------|
| 1 | Native SSM (IGMP/MLD) | Multicast-enabled networks |
| 2 | AMT over UDP (RFC 7450) | Any network with UDP |
| 3 | MoQ over QUIC | Universal |
| 4 | HTTP DASH/HLS | Universal fallback |

**Browser clients**:

| Priority | Transport | Availability |
|----------|-----------|-------------|
| 1 | SSM via IWA DirectSocket | IWA-enabled Chrome |
| 2 | AMT via IWA DirectSocket | IWA-enabled Chrome |
| 3 | MoQ over WebTransport | Chrome 97+, Firefox 115+ |
| 4 | HTTP DASH via MSE/fetch | Universal fallback |

For each transport tier, if FEC repair tracks are available,
subscribers SHOULD subscribe to repair tracks at the same priority.
FEC reduces retransmission latency but QUIC provides a reliable
fallback.

NOTE: For general consumer web applications, MoQ/WebTransport
(Priority 3) is effectively the highest-priority available transport.
IWA DirectSocket (Priorities 1-2) is available to enterprise-managed
Chrome, DePIN participants with token-incentivized IWA installation,
and technical users who side-load the IWA [WICG-DirectSockets].

## 7. Multicast Catalog Extension

For tracks available via multicast, the MoQ catalog includes a
top-level `multicast` field for endpoint discovery.  Two formats
are defined: a simple flat format for single-endpoint SSM
deployments, and an extended format for multi-endpoint or
multi-group scenarios.

### 7.1. Simple Multicast Format

The simple format is a flat object at the catalog root level with
three fields.  This is the RECOMMENDED format for deployments where
all tracks share a single SSM multicast group:

```json
{
  "multicast": {
    "groupAddress": "232.0.10.1",
    "port": 8000,
    "sourceAddress": "69.25.95.10"
  }
}
```

**groupAddress** (string, REQUIRED): Multicast group address.
  - SSM range: 232.0.0.0/8 (IPv4), ff3x::/32 (IPv6)
  - ASM range: 224.0.0.0/4 (IPv4), ff0x::/16 (IPv6)

**port** (integer, REQUIRED): UDP port number.

**sourceAddress** (string, OPTIONAL): SSM source IP address per [RFC4607].
  Required for Source-Specific Multicast.  If omitted, implies ASM.

**networkSource** (object, OPTIONAL): Network delivery configuration
  describing how subscribers can reach the multicast stream.  This
  enables endpoints that lack native multicast routing to receive
  streams via alternative delivery technologies.  See Section 7.4.

Subscribers receiving a catalog with the simple multicast format
SHOULD auto-connect to the multicast group when multicast APIs are
available (IWA DirectSocket, native UDP sockets, AMT tunneling).
This enables a seamless upgrade path: subscribers first connect via
MoQ/QUIC to receive the catalog and media, then optionally switch
to multicast for lower-latency, FEC-protected delivery.

### 7.2. Extended Multicast Format

For deployments with multiple multicast groups (e.g., per-quality
ABR tiers, separate audio/video groups, or ATSC 3.0 ingest with
multiple TSI values), the extended format uses an `endpoints` array:

```json
{
  "multicast": {
    "endpoints": [{
      "protocol": "ssm",
      "sourceAddress": "192.168.1.100",
      "groupAddress": "232.1.1.50",
      "port": 5000,
      "tsi": 1,
      "tracks": ["video", "video/repair", "audio"]
    }],
    "networkSource": {
      "type": "amt",
      "discovery": "driad",
      "relay": "amt.example.com"
    }
  }
}
```

Extended endpoint field definitions:

**protocol** (string, REQUIRED): Transport protocol.
  - "ssm": Source-Specific Multicast (RFC 4607)
  - "asm": Any-Source Multicast

**sourceAddress** (string, REQUIRED for SSM): SSM source IP address.

**groupAddress** (string, REQUIRED): Multicast group address.

**port** (integer, REQUIRED): UDP port number.

**tsi** (integer, OPTIONAL): Transport Session ID for correlation
  with broadcast signaling (e.g., S-TSID, MMTP).

**tracks** (array, REQUIRED): Track names available on this endpoint.

**networkSource** (object, OPTIONAL): Network delivery configuration
  (same as simple format).  See Section 7.4.

### 7.3. Format Detection

Subscribers MUST detect the multicast format by checking:

1. If `multicast.groupAddress` exists → simple format (Section 7.1)
2. If `multicast.endpoints` exists → extended format (Section 7.2)
3. Otherwise → no multicast available

Publishers MUST NOT include both `groupAddress` and `endpoints` in the
same catalog.

### 7.4. Network Source Types

The `networkSource` field describes how subscribers can reach the
multicast stream when native IP multicast routing is not available.
It appears at the `multicast` level in both simple and extended
formats.

**type** (string, REQUIRED): Delivery technology identifier.

Defined types:

#### 7.4.1. AMT (Automatic Multicast Tunneling)

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

#### 7.4.2. ATSC 3.0 (Broadcast Television)

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

#### 7.4.3. DVB (Digital Video Broadcasting)

```json
{
  "networkSource": {
    "type": "dvb",
    "frequency": 474000,
    "symbolRate": 6900,
    "modulation": "QAM256",
    "serviceId": 1
  }
}
```

**frequency** (integer, REQUIRED): RF center frequency in kHz.

**symbolRate** (integer, OPTIONAL): Symbol rate in kBaud.

**modulation** (string, OPTIONAL): Modulation scheme
  (e.g., "QAM256", "QAM64", "QPSK").

**serviceId** (integer, OPTIONAL): DVB service ID.

#### 7.4.4. 5G Broadcast (3GPP MBS)

```json
{
  "networkSource": {
    "type": "5g-broadcast",
    "tmgi": "00000A-00001",
    "serviceArea": "us-west-1"
  }
}
```

**tmgi** (string, REQUIRED): Temporary Mobile Group Identity per
  [3GPP-MBS].

**serviceArea** (string, OPTIONAL): Service area identifier.

This type indicates availability via 5G Multicast-Broadcast
Services.  Android devices with MBS-capable modems can receive
the stream via the Qualcomm LTE Broadcast SDK or 5G-MAG
middleware [5GMAG-LIBFLUTE].

#### 7.4.5. Multiple Network Sources

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
transport hierarchy defined in Section 6.

## 8. Multicast Packet Formats

Each MoQ packaging type defines its own multicast UDP wire format.
The catalog's `packaging` field tells receivers which parser to use.

### 8.1. MMTP Multicast

MMTP packets (source and repair) are transmitted as-is in UDP
datagrams per [I-D.ramadan-moq-mmt].  The MMTP header provides
packet_id for track routing, FEC type for source/repair
discrimination, and NTP timestamps.  No additional framing needed.

### 8.2. LOC Multicast

LOC objects are transmitted with LOC headers per [I-D.ietf-moq-loc].
FEC Payload ID (SBN/ESI) is carried via LOC extension headers.
Auth tags use the LOC Auth Tag extension.

### 8.3. Condensed Multicast

The condensed format provides lightweight multicast framing for
CMAF content.  Each UDP datagram carries one condensed packet:

```
Condensed Multicast Packet {
  Track ID (16),              // 2 bytes — audio/video routing
  Repair (8),                 // 1 byte — 0=source, 1=repair
  Source Block Number (24),   // 3 bytes — SBN
  Encoding Symbol ID (8),    // 1 byte — ESI (max 255/block)
  Auth Length (8),            // 1 byte — 0=no auth
  Payload (..),               // source chunk or repair symbol
  [Auth Tag (Auth Length)],   // variable-length auth tag
}
```

**Track ID** (16 bits): Identifies the media track within the SSM
  group (e.g., video=1, audio=2).  Assigned by the publisher and
  signaled in the catalog.  Analogous to MMTP packet_id.

**Repair** (8 bits): 0 for source, 1 for repair.  Needed because
  K varies per block (e.g., blocks spanning keyframes), making
  ESI >= K unreliable for source/repair discrimination.

**SBN** (24 bits): Source Block Number.  Wraps at ~16M blocks
  (~25 days at 30fps with interleave depth 4).

**ESI** (8 bits): Encoding Symbol ID.  Max 255 symbols per block.

**Auth Length** (8 bits): Length of auth tag.  0 = no auth.
  Supports fixed-size (HMAC) and variable-size (ALTA) schemes.

**Auth Tag**: Present when Auth Length > 0.  Scheme-specific format.

Fixed overhead: 8 bytes.  Compared to MMTP (33 bytes for repair
with FEC Payload ID + OTI) this is 4x more compact.

On the MoQ unicast path, condensed packaging uses the headerless
format defined in [I-D.ramadan-moq-fec] Section 7.1 with SBN/ESI
derived from MoQ transport framing.  The multicast wire format in
this section applies only to IP multicast UDP delivery.

## 9. Security Considerations

### 9.1. Multicast Security

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

## 10. IANA Considerations

This document has no IANA actions.  The multicast catalog extension
(Section 7) is a JSON schema extension to MoQ catalogs and does not
require IANA registration.

## 11. References

### 11.1. Normative References

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

### 11.2. Informative References

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

[ATSC-A331]
           ATSC, "Signaling, Delivery, Synchronization, and Error
           Protection", A/331:2025, February 2025.

[WICG-DirectSockets]
           Rayskiy, A., "Direct Sockets API", W3C Community Group
           Draft Report, https://wicg.github.io/direct-sockets/

[5GMAG-LIBFLUTE]
           5G-MAG, "rt-libflute: FLUTE/ALC sender and receiver",
           https://github.com/5G-MAG/rt-libflute

[3GPP-MBS]
           3GPP, "5G Multicast-Broadcast Services; Stage 1",
           TS 22.246, Release 17.

[JUNIPER-TREEDN]
           Giuliano, L., "TreeDN - The Fix for Catastrophically
           Successful Live Streaming Events", Juniper TechPost,
           August 2025, https://community.juniper.net/blogs/lenny/
           2025/08/21/introduction-to-TreeDN

## Appendix A.  Catalog Examples

### A.1.  Simple Catalog (Single Multicast Endpoint)

Catalog with FEC and multicast for a single SSM group:

```json
{
  "video": {
    "renditions": {
      "video": {
        "codec": "avc1.64001f",
        "codedWidth": 1920,
        "codedHeight": 1080,
        "framerate": 30,
        "bitrate": 5000000,
        "container": "mmtp",
        "repairTrack": "repair",
        "fecSourceSymbols": 32,
        "fecRepairSymbols": 8,
        "fecSymbolSize": 1312,
        "fecInterleaveDepth": 4
      }
    }
  },
  "audio": {
    "renditions": {
      "audio": {
        "codec": "mp4a.40.2",
        "sampleRate": 48000,
        "numberOfChannels": 2,
        "bitrate": 128000,
        "container": "mmtp"
      }
    }
  },
  "multicast": {
    "groupAddress": "232.0.10.1",
    "port": 8000,
    "sourceAddress": "69.25.95.10",
    "networkSource": {
      "type": "amt",
      "relay": "69.25.95.1",
      "discovery": "driad"
    }
  }
}
```

Subscribers auto-discover the multicast endpoint from the catalog
and join the SSM group directly when multicast APIs are available.
When native multicast is unavailable, the `networkSource` field
directs subscribers to the AMT relay for tunneled delivery.

### A.2.  Extended Catalog (Multiple Endpoints)

For deployments with multiple multicast groups (e.g., per-quality ABR
tiers or separate audio/video groups):

```json
{
  "video": {
    "renditions": {
      "video": {
        "codec": "avc1.64001f",
        "codedWidth": 1920,
        "codedHeight": 1080,
        "framerate": 30,
        "bitrate": 8000000,
        "container": "mmtp",
        "repairTrack": "repair",
        "fecSourceSymbols": 32,
        "fecRepairSymbols": 8,
        "fecSymbolSize": 1312,
        "fecInterleaveDepth": 30
      }
    }
  },
  "audio": {
    "renditions": {
      "audio": {
        "codec": "mp4a.40.2",
        "sampleRate": 48000,
        "numberOfChannels": 2,
        "bitrate": 128000,
        "container": "mmtp"
      }
    }
  },
  "multicast": {
    "endpoints": [
      {
        "protocol": "ssm",
        "sourceAddress": "10.0.0.1",
        "groupAddress": "232.1.1.10",
        "port": 5000,
        "tsi": 1,
        "tracks": ["video", "repair"]
      },
      {
        "protocol": "ssm",
        "sourceAddress": "10.0.0.1",
        "groupAddress": "232.1.1.10",
        "port": 5000,
        "tsi": 2,
        "tracks": ["audio"]
      }
    ],
    "networkSource": {
      "type": "amt",
      "discovery": "driad"
    }
  }
}
```

Subscribers MUST check for the simple format fields (`groupAddress`, `port`)
first.  If not present, check for the `endpoints` array.

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
