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
   Multicast Tunneling (AMT), and TreeDN for scalable live media
   delivery.  The specification includes a multicast catalog extension
   for endpoint discovery, delivery path selection across TV, mobile,
   and browser platforms, and a transport priority hierarchy.  This
   document is container-format agnostic and is referenced by both
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
       5.2.  IWA Home Gateway Architecture
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
   8.  Security Considerations
       8.1.  Multicast Security
   9.  IANA Considerations
   10. References
       10.1. Normative References
       10.2. Informative References
   Appendix A.  Catalog Examples
       A.1.  Simple Catalog (Single Multicast Endpoint)
       A.2.  Extended Catalog (Multiple Endpoints)
   Authors' Addresses
```

## 1. Introduction

Live streaming at scale strains unicast CDN architectures, particularly
at 4K/8K bitrates.  IP multicast replicates streams at the network
layer but lacks widespread ISP deployment.  Media over QUIC (MoQ)
[I-D.ietf-moq-transport] provides modern real-time media transport
over QUIC, yet its unicast model shares the same scaling limits.

This document specifies how MoQ integrates with IP multicast:

1. **Delivery paths**: Platform-specific multicast reception for
   TV, mobile, and browser clients
2. **TreeDN integration**: Hierarchical multicast distribution
   trees with AMT relay bridging per [RFC9706]
3. **Catalog extension**: A container-agnostic multicast endpoint
   discovery mechanism for MoQ catalogs

This specification is container-format agnostic and works with
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

**IWA**: Isolated Web App -- a Chrome packaging format granting
DirectSocket API access

## 3. Delivery Paths

MoQ content reaches receivers via multiple delivery paths depending
on platform capabilities:

| Client Type | Multicast Path | Requirements |
|-------------|----------------|--------------|
| TV (ATSC 3.0 / ARIB) | SSM direct (tuner) | ATSC 3.0 or ARIB receiver hardware |
| Android (native) | SSM via MulticastSocket | MulticastLock permission |
| Android (5G Broadcast) | 3GPP MBS [3GPP-MBS] | MBS-capable modem + middleware |
| iOS (native) | SSM via NWConnectionGroup | com.apple.developer.networking.multicast entitlement |
| Browser (IWA) | SSM/AMT via DirectSocket | IWA installation (enterprise policy, side-load, or allowlisting) |
| Browser (standard) | MoQ/WebTransport | None (production-ready) |

Both native SSM and AMT tunneling require UDP socket access.

**TV (ATSC 3.0 / ARIB)**: Broadcast receivers consume ROUTE/ALC or
MMTP natively over RF tuner hardware with FEC decoding and S-TSID
parsing per [ATSC-A331].  MoQ integration occurs at the gateway level
via TreeDN [RFC9706] and AMT [RFC7450] / DRIAD [RFC8777].

**Android**: MulticastSocket with MulticastLock for Wi-Fi multicast.
For 5G Broadcast, the device modem provides carrier-grade multicast
via MBS middleware [5GMAG-LIBFLUTE].

**iOS**: NWConnectionGroup (iOS 14+) with Apple-granted entitlement.

**Browser (IWA)**: The DirectSocket API [WICG-DirectSockets] provides
UDPSocket with joinGroup()/leaveGroup() for SSM and ASM reception.
IWAs require enterprise Chrome policy, Google allowlisting, or manual
user installation, limiting consumer reach but serving enterprise,
kiosk, and power-user deployments.

**Browser (standard)**: MoQ over WebTransport is the production path
for standard browsers.  WebCodecs provides sub-100ms decode latency;
MSE provides fallback.  No special permissions required.

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

MoQ relays MAY operate as TreeDN nodes, forwarding packets
over both unicast (QUIC) and multicast (SSM/AMT) paths simultaneously.

When operating as a TreeDN node:
1. Relay receives media over SSM from upstream
2. Relay serves downstream via QUIC (MoQ) and/or SSM (multicast)
3. FEC repair is preserved or augmented per local conditions

### 5.1. ISP Router AMT Deployment

ISP edge routers can serve as AMT relays, replacing x86 CDN servers
with inline packet replication on existing routing silicon
[JUNIPER-TREEDN].  AMT relay capability is built into commodity
routing chipsets and requires no additional hardware.

This architecture offloads unicast replication to in-network routers
performing either:

- Native SSM replication where PIM-SSM is enabled
- AMT unicast replication per [RFC7450] where multicast is unavailable

The MoQ relay at the edge terminates the multicast/AMT path and serves
clients over QUIC/WebTransport.

### 5.2. IWA Home Gateway Architecture

An IWA node on a wired device (Chromebox, desktop, mini-PC) can serve
as a local multicast-to-MoQ gateway.  It receives SSM or AMT multicast
via DirectSocket over wired Ethernet, then redistributes content over
WebTransport to wireless clients on the local network:

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

This provides: (1) wired reception avoids Wi-Fi jitter, (2) a single
multicast stream serves the entire household, (3) wireless clients
use standard MoQ/WebTransport with no IWA or special permissions, and
(4) the gateway can apply local FEC repair.

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

NOTE: For general consumer web applications, MoQ/WebTransport
(Priority 3) is effectively the highest-priority available transport.
IWA DirectSocket (Priorities 1-2) is limited to enterprise-managed
Chrome and users who side-load the IWA [WICG-DirectSockets].

## 7. Multicast Catalog Extension

For tracks available via multicast, the MoQ catalog includes a
top-level `multicast` field for endpoint discovery.  Two formats
are defined: a simple flat format for single-endpoint SSM
deployments, and an extended format for multi-endpoint scenarios.

### 7.1. Simple Multicast Format

The simple format is a flat object at the catalog root level.
This is the RECOMMENDED format for deployments where all tracks
share a single SSM multicast group:

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

**sourceAddress** (string, OPTIONAL): SSM source IP per [RFC4607].
  Required for SSM.  If omitted, implies ASM.

**networkSource** (object, OPTIONAL): Network delivery configuration.
  See Section 7.4.

Subscribers receiving a catalog with the simple multicast format
SHOULD auto-connect to the multicast group when multicast APIs are
available.

### 7.2. Extended Multicast Format

For deployments with multiple multicast groups (per-quality ABR
tiers, separate audio/video groups, or multiple TSI values), the
extended format uses an `endpoints` array:

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

Extended endpoint fields:

**protocol** (string, REQUIRED): "ssm" (RFC 4607) or "asm".

**sourceAddress** (string, REQUIRED for SSM): SSM source IP.

**groupAddress** (string, REQUIRED): Multicast group address.

**port** (integer, REQUIRED): UDP port number.

**tsi** (integer, OPTIONAL): Transport Session ID for correlation
  with broadcast signaling (e.g., S-TSID, MMTP).

**tracks** (array, REQUIRED): Track names on this endpoint.

**networkSource** (object, OPTIONAL): Network delivery configuration.
  See Section 7.4.

### 7.3. Format Detection

Subscribers MUST detect the multicast format by checking:

1. If `multicast.groupAddress` exists -> simple format (Section 7.1)
2. If `multicast.endpoints` exists -> extended format (Section 7.2)
3. Otherwise -> no multicast available

Publishers MUST NOT include both `groupAddress` and `endpoints` in the
same catalog.

### 7.4. Network Source Types

The `networkSource` field describes how subscribers reach the
multicast stream when native IP multicast routing is unavailable.
It appears at the `multicast` level in both formats.

**type** (string, REQUIRED): Delivery technology identifier.

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

**discovery** (string, OPTIONAL): Relay discovery method.
  - "driad": DNS Reverse IP AMT Discovery per [RFC8777] (RECOMMENDED).
  - "manual": Relay address in `relay` field.

Subscribers SHOULD attempt relay discovery in this order:
1. Use `relay` directly if provided
2. Use DRIAD on the SSM source IP if `discovery` is "driad" or omitted
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

**plpId** (integer, OPTIONAL): Physical Layer Pipe ID (default 0).

**serviceId** (integer, OPTIONAL): Service ID within the multiplex.

Receivers with ATSC 3.0 tuner hardware can receive the stream
directly over RF per [ATSC-A331].

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

Android devices with MBS-capable modems receive via MBS
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

Subscribers select the highest-priority available source per
Section 6.

## 8. Security Considerations

### 8.1. Multicast Security

SSM limits traffic to authorized sources via (S,G) filtering.
Additional mechanisms:

- **DTLS-SRTP**: Encrypt media payloads when confidentiality is
  required over multicast
- **Source Authentication**: Validate source IP at AMT relay;
  for SSM, receivers verify (S,G) membership
- **FEC Integrity**: FEC decoder inconsistencies can detect injection
  attacks; receivers SHOULD verify decoded media structure
- **Replay Protection**: Sequence numbers enable detection of
  replayed or reordered packets
- **AMT Relay Trust**: Trust delegated to relay authentication
  per [RFC7450]
- **Content Authentication**: Publishers MAY include an Auth Tag
  MoQ extension header per [I-D.ramadan-moq-fec] on source track
  objects.  The auth tag (e.g., ALTA) enables receivers to verify
  content integrity across untrusted relay chains, complementing
  SSM source address validation and QUIC hop-by-hop integrity.
  The authentication scheme and tag size are signaled in the catalog.

## 9. IANA Considerations

This document has no IANA actions.  The multicast catalog extension
(Section 7) is a JSON schema extension to MoQ catalogs and does not
require IANA registration.

## 10. References

### 10.1. Normative References

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

### 10.2. Informative References

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

Subscribers auto-discover the multicast endpoint from the catalog and
join the SSM group when multicast APIs are available.  The
`networkSource` field directs subscribers to the AMT relay when native
multicast is unavailable.

### A.2.  Extended Catalog (Multiple Endpoints)

Catalog with multiple multicast groups for per-quality ABR tiers:

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

Subscribers MUST check for simple format fields (`groupAddress`,
`port`) first, then the `endpoints` array.

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
