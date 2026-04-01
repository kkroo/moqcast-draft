# MPEG Media Transport (MMT) Packaging for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: October 2026                                      April 2026

        MPEG Media Transport (MMT) Packaging for Media over QUIC
                      draft-ramadan-moq-mmt-00

Abstract

   This document specifies the use of MPEG Media Transport (MMT) as a
   container format for Media over QUIC (MoQ).  MMT provides a unified
   framework for streaming media over heterogeneous networks including
   broadcast (ATSC 3.0, ARIB STD-B60), multicast (SSM), and unicast
   (QUIC/WebTransport).  This specification defines the mapping of MMT
   packets to MoQ objects, interaction with Application-Layer FEC
   including multi-path FEC delivery where MMTP repair packets are
   passed through identically across ATSC 3.0 broadcast, SSM multicast,
   and MoQ CDN paths, and compatibility considerations for both
   ATSC 3.0 and ARIB STD-B60 systems.

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

Table of Contents

   1.  Introduction
       1.1.  Relationship to Other MoQ Media Formats
   2.  Terminology
   3.  MMT Overview
       3.1.  MMTP Header Format
   4.  MoQ Object Mapping
       4.1.  Track Structure
       4.2.  Object Payload
       4.3.  Group Boundaries
       4.4.  Unified Ordering Model
       4.5.  Signaling Messages
   5.  Media Fragment Unit (MFU) Mode
       5.1.  MFU Fragmentation
   6.  FEC Integration
   7.  Multicast Integration
   8.  ARIB STD-B60 Compatibility
   9.  Catalog Signaling
       9.1.  Packaging Registration
   10. Security Considerations
       10.1. Multicast Security
   11. IANA Considerations
   12. References
   Authors' Addresses
```

## 1. Introduction

MPEG Media Transport (MMT) [ISO.23008-1] is a transport protocol
designed for delivery of multimedia content over heterogeneous
networks.  MMT is supported as an alternative transport in ATSC 3.0
broadcast television (Americas, South Korea) alongside the primary
ROUTE/DASH transport, and is the basis for ARIB STD-B60 (Japan).
MMT provides native support for:

- Media Processing Units (MPU, branded "mpuf" in ISOBMFF) as the
  media container, with structural similarities to fMP4
- Application-Layer FEC (AL-FEC) supporting multiple schemes
  including RaptorQ, LDPC, and Reed-Solomon
- Cross-layer signaling for adaptive streaming
- Hybrid delivery combining broadcast and broadband

This document defines how MMT-encapsulated media can be transported
over MoQ [I-D.ietf-moq-transport], enabling:

1. **Broadcast-to-Unicast bridging**: Content from ATSC 3.0 or ARIB STD-B60
   broadcasts can be relayed to MoQ subscribers without transcoding
2. **Unified FEC**: MMT's AL-FEC integrates with MoQ FEC repair tracks
   per [I-D.ramadan-moq-fec]
3. **Multicast promotion**: MoQ clients can receive SSM multicast
   directly when available, per [I-D.ramadan-moq-multicast]

NOTE: This specification targets gateway/ingest scenarios where
ATSC 3.0 or ARIB STD-B60 broadcast content enters the MoQ ecosystem.
For pure MoQ unicast delivery without broadcast interoperability
requirements, LOC [I-D.ietf-moq-loc] is RECOMMENDED (see Section 4.2).

### 1.1. Relationship to Other MoQ Media Formats

This specification complements existing MoQ packaging formats:

| Format | Container | Primary Use Case | FEC Support |
|--------|-----------|------------------|-------------|
| MSF (LOC) | WebCodecs chunks | Low-latency unicast | Via [I-D.ramadan-moq-fec] |
| CMSF | CMAF fMP4 | Adaptive streaming | Via [I-D.ramadan-moq-fec] |
| CARP | CMAF fMP4 | ABR delivery | Via [I-D.ramadan-moq-fec] |
| This spec (MMT) | MMTP + MPU (ISOBMFF) | Broadcast bridge | Native + [I-D.ramadan-moq-fec] |

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

**MMTP**: MMT Protocol - the packet layer of MMT (ISO 23008-1 Clause 8)

**MPU**: Media Processing Unit - a self-contained media segment in MMT,
typically aligned with a Group of Pictures (GOP)

**MFU**: Media Fragment Unit - a single coded media frame within an MPU

**AL-FEC**: Application-Layer Forward Error Correction

**S-TSID**: Service-based Transport Session Instance Description -
ATSC 3.0 signaling table containing transport and FEC parameters

**SLT**: Service Layer Table - ATSC 3.0 bootstrap signaling

**MPT**: MMT Package Table - signaling table containing MMT asset info

**MPI**: MMT Presentation Information - presentation timing table

## 3. MMT Overview

MMT uses a layered architecture:

```
┌─────────────────────────────────────────────┐
│              Application Layer              │
├─────────────────────────────────────────────┤
│  MPU (Media Processing Unit)                │
│  ┌─────────┬─────────┬─────────┬─────────┐ │
│  │  MFU 0  │  MFU 1  │  MFU 2  │   ...   │ │
│  │ (IDR)   │ (P)     │ (P)     │         │ │
│  └─────────┴─────────┴─────────┴─────────┘ │
├─────────────────────────────────────────────┤
│  MMTP (MMT Protocol)                        │
│  ┌─────────────────────────────────────────┐│
│  │ MMTP Header │ Payload (MPU fragment)   ││
│  │ 12 bytes    │ Variable                 ││
│  └─────────────────────────────────────────┘│
├─────────────────────────────────────────────┤
│  Transport: UDP/IP (broadcast/multicast)    │
│             QUIC/WebTransport (unicast)     │
└─────────────────────────────────────────────┘
```

### 3.1. MMTP Header Format

```
MMTP Header (12 bytes minimum) {
  Version (2),
  Packet Counter Flag (C) (1),
  FEC Type (F) (2),
  Reserved (1),
  Extension Flag (X) (1),
  RAP Flag (R) (1),
  Packet Type (6),
  Packet ID (16),
  Timestamp (32),
  Packet Sequence Number (32),
  [Packet Counter (32)],       // Present if C=1
  [Header Extension (..)]      // Present if X=1
}
```

Key fields for MoQ mapping:
- **Packet ID**: Maps to MoQ track within namespace
- **Timestamp**: UTC wallclock time in NTP short format, maps to MoQ object timestamp
- **Packet Sequence Number**: Maps to MoQ object ID within group
- **FEC Type**: 0=no AL-FEC, 1=AL-FEC source packet,
  2=AL-FEC repair packet, 3=reserved
- **RAP Flag**: 1 indicates Random Access Point

## 4. MoQ Object Mapping

### 4.1. Track Structure

An MMT stream maps to MoQ tracks as follows:

| MMT Component | MoQ Mapping |
|---------------|-------------|
| Asset (stream) | Namespace |
| Packet ID (video) | Track "video" |
| Packet ID (audio) | Track "audio" |
| MPU sequence | Group ID |
| MFU sequence | Object ID |
| AL-FEC repair | Track "video/repair" |

For ATSC 3.0 ingest, the gateway preserves the original packet_id assignments from the MPT (MMT Package Table). For MoQ-originated content using MMTP packaging, packet_id 0 is reserved for signaling messages (Section 4.5), with media tracks assigned sequentially starting from 1 (video=1, audio=2, etc.). The mapping is signaled in the catalog.

### 4.2. Object Payload

Each MoQ object carries one MMTP packet:

```
MoQ Object Payload {
  MMTP Header (12+ bytes),
  MPU Fragment (MPU metadata or MFU payload),
}
```

Publishers MAY strip the MMTP header if subscribers negotiate
raw ISOBMFF delivery via catalog `container` field.  When MMTP headers
are stripped, the object payload contains only the MPU/ISOBMFF fragment.
Note that stripping MMTP headers loses FEC Type, RAP flag, and
timestamp metadata; receivers MUST rely on catalog and MoQ object
headers for this information.

For MoQ unicast delivery without ATSC 3.0/ARIB interoperability
requirements, LOC [I-D.ietf-moq-loc] with MoQ extension headers is
RECOMMENDED over full MMTP encapsulation.  LOC eliminates 33 bytes
of per-object MMTP overhead (12-byte header + 9-byte FEC Payload
ID + 12-byte OTI for repair packets, per [I-D.ramadan-moq-fec]
Appendix C) that is redundant over MoQ (packet_id,
timestamp, sequence_number, MPU fields, MFU DU header are all
provided by MoQ transport framing and LOC extensions).  MMTP
encapsulation (container value 'mmtp') is RECOMMENDED when
multicast/broadcast interoperability is required, as each UDP packet
needs self-describing per-packet metadata.

### 4.3. Group Boundaries

Group boundaries align with MPU boundaries:
- Group N contains all MFUs of MPU N
- Object 0 of each group contains the MPU metadata (mmpu/moov boxes)
- Subsequent objects contain MFU payloads
- The first object of each group SHOULD have RAP Flag = 1

### 4.4. Unified Ordering Model

The MPU Sequence Number (mpuSeq) maps to MoQ Group ID (Section 4.1)
and serves as the single authoritative ordering key across media
decode, FEC recovery, and MoQ transport.

Receivers MUST track the last decoded mpuSeq and:
- **Keyframes (RAP=1)**: Always decode; reset decode-order tracking
- **Delta frames (RAP=0)**: Decode only if mpuSeq > lastDecodedMpuSeq;
  drop out-of-order delta frames to prevent reference frame mismatch
- **FEC repair**: Use mpuSeq to derive SBN for block alignment

This ordering model applies identically across MoQ unicast, SSM
multicast, and ATSC 3.0 broadcast delivery paths.

### 4.5. Signaling Messages

ATSC 3.0 MMTP streams carry signaling messages (PA, MPI, clock reference) in addition to media payloads. When ingesting ATSC 3.0 content:

- **PA (Package Access) messages**: Publishers SHOULD publish PA messages as MoQ objects on a dedicated signaling track (e.g., "signaling") to enable subscriber-side asset discovery.
- **MPI (MMT Presentation Information)**: Publishers MAY include MPI data in the catalog or as signaling track objects.
- **Clock references**: Mapped to MoQ object timestamps per Section 8.

Signaling messages that are only relevant to the broadcast transport layer (e.g., IP-level configuration) SHOULD NOT be forwarded to MoQ subscribers.

## 5. Media Fragment Unit (MFU) Mode

For ultra-low-latency applications, MMT supports MFU mode where
each video frame is delivered as a separate unit:

```
Standard MPU Mode:
  Object 0: [MMTP][MPU: mmpu+moov+moof+mdat containing all frames]

MFU Mode:
  Object 0: [MMTP][MPU metadata: mmpu+moov+moof header]
  Object 1: [MMTP][MFU: IDR frame NALUs]
  Object 2: [MMTP][MFU: P frame NALUs]
  Object 3: [MMTP][MFU: P frame NALUs]
  ...
```

MFU mode enables:
- Per-frame FEC protection
- Frame-level prioritization (IDR vs P/B)
- Lower end-to-end latency

### 5.1. MFU Fragmentation

When MFU size exceeds typical MTU (1200-1400 bytes for QUIC),
publishers SHOULD:

1. Fragment MFU across multiple MMTP packets
2. Set Packet Counter Flag (C=1) for reassembly tracking
3. Publish all fragments as a single MoQ object (not multiple objects)
4. Include the complete MFU in the object payload

Receivers reassemble MFUs using the Packet Counter and Packet Sequence
Number before media processing.  The MMTP fragmentation is transparent
to MoQ; each MoQ object represents a complete, potentially multi-packet
MFU.

## 6. FEC Integration

When using MMT packaging, both source and repair tracks use "mmtp"
packaging.  Repair track objects carry raw MMTP repair packets
(FEC Type=2) without modification per [I-D.ramadan-moq-fec]
Appendix C.  MoQ relays pass MMTP repair packets through as-is.

This passthrough design enables multi-path FEC delivery: identical
MMTP repair packets across ATSC 3.0 RF, SSM multicast, and MoQ CDN.
FEC parameters and OTI are signaled via the catalog per
[I-D.ramadan-moq-fec] Section 4.

## 7. Multicast Integration

MMT content can be delivered via IP multicast (SSM, AMT) and TreeDN
for scalable distribution.  Delivery paths, SSM group allocation,
and transport hierarchy are defined in [I-D.ramadan-moq-multicast].

When MMT is delivered over multicast, MMTP packets are transmitted
as UDP datagrams with the standard MMTP header intact.  MoQ relays
at network edges terminate the multicast path and bridge MMTP
content into the MoQ application layer via QUIC/WebTransport.

For ATSC 3.0 and ARIB STD-B60 receivers, MMTP over SSM is the
native delivery path and requires no protocol translation.

## 8. ARIB STD-B60 Compatibility

ARIB STD-B60 (Japan) uses the same ISO 23008-1 foundation as
ATSC 3.0.  MMTP Timestamps use NTP short format (32-bit: 16-bit
seconds + 16-bit fractional), mapping directly to MoQ object
timestamps.

## 9. Catalog Signaling

The MoQ catalog signals MMT packaging via the `packaging` field
per [I-D.ietf-moq-catalogformat].

### 9.1. Packaging Registration

This document registers "mmtp" as a MoQ Streaming Format packaging
value:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "mmtp" | MMTP-encapsulated media per ISO 23008-1 | This document |

MMTP packaging indicates that each MoQ object payload contains one
MMTP packet (source or repair).  The MMTP header provides packet
type, timestamp, sequence number, and FEC metadata natively.

Example:
```json
{
  "version": 1,
  "streamingFormat": 1,
  "streamingFormatVersion": "0.2",
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "selectionParams": {
        "codec": "avc1.64001f",
        "width": 1920,
        "height": 1080,
        "framerate": 30
      }
    },
    {
      "name": "video/repair",
      "packaging": "mmtp"
    }
  ]
}
```

When a publisher offers the same content via multiple packaging
types (mmtp, loc, cmaf), each is a separate track in the catalog.
The subscriber chooses which packaging to subscribe to based on
its capabilities.

The multicast catalog extension is defined in
[I-D.ramadan-moq-multicast] Section 4.

## 10. Security Considerations

MMT content protection uses Common Encryption (CENC) which is
preserved through MoQ transport.  The MMTP header is not encrypted,
allowing relays to inspect packet type and sequence without
accessing media content.

### 10.1. Multicast Security

Multicast-specific security considerations (source authentication,
replay protection, AMT relay trust) are defined in
[I-D.ramadan-moq-multicast] Section 6.

## 11. IANA Considerations

This document requests registration of a MoQ Streaming Format
packaging value in the registry established by
[I-D.ietf-moq-catalogformat]:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "mmtp" | MMTP-encapsulated media per ISO 23008-1 | This document |

## 12. References

[RFC2119]
    Bradner, S., "Key words for use in RFCs to Indicate
    Requirement Levels", BCP 14, RFC 2119, March 1997.

[RFC8174]
    Leiba, B., "Ambiguity of Uppercase vs Lowercase in
    RFC 2119 Key Words", BCP 14, RFC 8174, May 2017.

[ISO.23008-1]
    ISO, "Information technology - High efficiency coding and
    media delivery in heterogeneous environments - Part 1:
    MPEG media transport (MMT)", ISO/IEC 23008-1:2023.

[RFC6330]
    Luby, M., et al., "RaptorQ Forward Error Correction Scheme
    for Object Delivery", RFC 6330, August 2011.

[I-D.ietf-moq-transport]
    Curley, L., et al., "Media over QUIC Transport",
    draft-ietf-moq-transport (work in progress).

[I-D.ietf-moq-loc]
    Jennings, C., et al., "Low Overhead Container for Media
    over QUIC", draft-ietf-moq-loc (work in progress).

[I-D.ramadan-moq-fec]
    Ramadan, O., "Forward Error Correction for Media over QUIC",
    draft-ramadan-moq-fec (work in progress).

[RFC5170]
    Roca, V., et al., "Low Density Parity Check (LDPC) Staircase
    and Triangle Forward Error Correction (FEC) Schemes",
    RFC 5170, June 2008.

[RFC5510]
    Lacan, J., et al., "Reed-Solomon Forward Error Correction
    (FEC) Schemes", RFC 5510, April 2009.

[ATSC-A331]
    ATSC, "Signaling, Delivery, Synchronization, and Error
    Protection", A/331:2025, February 2025.

[ARIB-B60]
    ARIB, "MMT-Based Media Transport Scheme in Digital
    Broadcasting Systems", ARIB STD-B60, Version 1.14.

[I-D.ramadan-moq-multicast]
    Ramadan, O., "Multicast Delivery and Endpoint Discovery
    for Media over QUIC", draft-ramadan-moq-multicast
    (work in progress).

[I-D.ietf-moq-catalogformat]
    Curley, L., et al., "Common Media Server Format for
    Media over QUIC", draft-ietf-moq-catalogformat
    (work in progress).

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
