# MPEG Media Transport (MMT) Packaging for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: January 2027                                       July 2026

        MPEG Media Transport (MMT) Packaging for Media over QUIC
                      draft-ramadan-moq-mmt-02

Abstract

   This document specifies the use of MPEG Media Transport (MMT) as a
   container format for Media over QUIC (MoQ).  MMT provides a unified
   framework for streaming media over heterogeneous networks including
   broadcast (ATSC 3.0, ARIB STD-B60), multicast (SSM), and unicast
   (QUIC/WebTransport).  This specification defines the mapping of MMT
   packets to MoQ objects, interaction with Application-Layer FEC
   including multi-path FEC delivery where MMTP repair packets are
   passed through identically across ATSC 3.0 broadcast, SSM multicast,
   and MoQ CDN paths, bidirectional conversion between S-TSID and MoQ
   catalogs, and compatibility considerations for both ATSC 3.0 and
   ARIB STD-B60 systems.

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
   5.  Media Fragment Unit (MFU) Mode
       5.1.  MFU Fragmentation
   6.  FEC Integration
       6.1.  Interleaving
       6.2.  OTI Signaling
   7.  FEC_CONFIG Message
       7.1.  Message Format
       7.2.  Repair Track Discovery
       7.3.  Multicast Delivery of FEC_CONFIG
   8.  Multicast Integration
   9.  ARIB STD-B60 Compatibility
       9.1.  Clock Reference
       9.2.  8K UHDTV Support
       9.3.  Hybridcast Integration
       9.4.  Typical FEC Parameters
   10. Transport Hierarchy
   11. Catalog Signaling
       11.1. Container Values
       11.2. S-TSID to MoQ Catalog Conversion
       11.3. MoQ Catalog to S-TSID Conversion
       11.4. Multicast Catalog Extension Reference
   12. Security Considerations
       12.1. Multicast Security
   13. IANA Considerations
   14. References
   Appendix A. Bandwidth Comparison
   Appendix B. S-TSID Conversion Example
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
4. **Bidirectional signaling**: Convert between S-TSID (ATSC) and MoQ
   catalogs for seamless interoperability

### 1.1. Relationship to Other MoQ Media Formats

This specification complements existing MoQ packaging formats:

| Format | Container | Primary Use Case | FEC Support |
|--------|-----------|------------------|-------------|
| MSF (LOC) | WebCodecs chunks | Low-latency unicast | No |
| CMSF | CMAF fMP4 | Adaptive streaming | No |
| CARP | CMAF fMP4 | ABR delivery | No |
| This spec (MMT) | MMTP + MPU (ISOBMFF) | Broadcast bridge | Yes |

MMT packaging is RECOMMENDED when:
- Ingesting ATSC 3.0 or ARIB STD-B60 broadcasts
- FEC protection is required for multicast delivery
- Hybrid broadcast/unicast architectures are deployed
- Interoperability with broadcast receivers is needed

CMAF packaging (CMSF/CARP) is RECOMMENDED when:
- Content originates as DASH/HLS
- No multicast delivery is planned
- Interoperability with existing CDN infrastructure is needed

## 2. Terminology

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
RECOMMENDED over full MMTP encapsulation.  LOC eliminates ~30 bytes
of per-object MMTP overhead that is redundant over MoQ (packet_id,
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

The MPU Sequence Number serves as the single authoritative ordering
key across all subsystems.  This design ensures consistency between
media decode, FEC recovery, and MoQ transport:

```
MPU Sequence Number (mpuSeq)
  │
  ├── MoQ Group ID        (§4.1: mpuSeq → Group ID)
  │     └── Decode order   — H.264/H.265 requires frames in sequence
  │
  ├── FEC Source Block     (SBN = mpuSeq / K, per [I-D.ramadan-moq-fec])
  │     └── ESI alignment  — source symbols map to sequential mpuSeqs
  │
  └── MMTP Timestamp       (§3.1: wallclock presentation time)
        └── Presentation    — monotonic within decode order
```

Receivers MUST track the last decoded mpuSeq and:
- **Keyframes (RAP=1)**: Always decode; reset decode-order tracking
- **Delta frames (RAP=0)**: Decode only if mpuSeq > lastDecodedMpuSeq;
  drop out-of-order delta frames to prevent reference frame mismatch
- **FEC repair**: Use mpuSeq to derive SBN for block alignment

This ordering model applies identically across delivery paths:
- MoQ unicast (QUIC streams, per-group delivery)
- SSM multicast (UDP, per-packet delivery)
- ATSC 3.0 broadcast (ROUTE/MMTP, per-packet delivery)

The decoder MUST enforce mpuSeq ordering regardless of delivery path.
While MoQ per-group delivery naturally provides ordered groups,
multicast and flat-stream delivery may reorder MFUs due to:
- FEC interleaving (interleave depth > 1 causes block-level reordering)
- Network path reordering (UDP provides no ordering guarantee)
- CDN relay buffering (objects from different groups may interleave)

Receivers SHOULD NOT rely on the transport layer for decode ordering.
The decoder is the correct enforcement point because different
delivery paths (MoQ groups, multicast, broadcast) have different
ordering guarantees, but the decode requirement is the same.

Implementation note: Fragment reassembly (handling FI=1,2,3 within
a single MFU) and decode ordering (enforcing mpuSeq between MFUs)
are separate concerns.  The reassembler buffers and reorders fragments
by fragment counter; the decoder tracks lastDecodedMpuSeq and drops
out-of-order delta frames.  This separation avoids cross-layer
coupling between the container parser and the media decoder.

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

For very large frames (e.g., 8K I-frames), consider:
- Using QUIC's stream-based reliable delivery
- Increasing QUIC max datagram size
- Fragmenting at the MoQ object level with object dependencies

## 6. FEC Integration

MMT's AL-FEC framework supports multiple FEC schemes including
RaptorQ [RFC6330], LDPC [RFC5170], and Reed-Solomon [RFC5510],
with parameters signaled via MMTP signaling messages or, in
ATSC 3.0, via S-TSID (which is part of the ROUTE transport layer).
For MoQ, FEC repair uses the model defined in
[I-D.ramadan-moq-fec]:

```
Source Track: video
  └── Objects: MMTP packets with FEC Type=0 or 1 (source)

Repair Track: video/repair
  └── Objects: MMTP packets with FEC Type=2 (repair, passthrough)
```

When using MMT packaging, the repair track also uses "mmtp"
packaging.  Repair track objects carry raw MMTP repair packets
(type 0x03) without modification per [I-D.ramadan-moq-fec]
Appendix C.  MoQ relays pass MMTP repair packets through as-is —
no header stripping, no re-framing, no re-encoding.

This passthrough design enables multi-path FEC delivery: the same
MMTP repair packets are delivered identically via ATSC 3.0 RF
broadcast, SSM multicast, and MoQ CDN.  Receivers implement one
MMTP parser for both source and repair tracks, extracting SBN, ESI,
and OTI from the standard MMTP FEC Payload ID fields.  See
[I-D.ramadan-moq-fec] Appendix C for the byte-offset extraction
recipe.

### 6.1. Interleaving

MMT AL-FEC interleaves source symbols across multiple MFUs:

```
MFU:        0    1    2    3    4    5    6    7
            │    │    │    │    │    │    │    │
Block 0:    S0   S1   S2   S3   ─────────────────►  R0, R1
Block 1:                        S4   S5   S6   S7 ► R2, R3
```

Default interleave depth varies by application:
- ATSC 3.0: 30-60 frames (~1-2 seconds at 30fps)
- ARIB STD-B60: 60 frames (~2 seconds at 30fps)
- Low-latency: 4-8 frames (~130-270ms at 30fps)

### 6.2. OTI Signaling

The S-TSID table contains RaptorQ OTI (Object Transmission Info):

```
S-TSID {
  source_filter (S,G address),
  fec_oti {
    transfer_length (40 bits),
    symbol_size (16 bits),
    num_source_blocks (8 bits),
    num_sub_blocks (16 bits),
    alignment (8 bits),
  }
}
```

For MoQ, OTI is signaled via FEC_CONFIG message per
[I-D.ramadan-moq-fec] Section 4.

## 7. FEC_CONFIG Message

The FEC_CONFIG message and its wire format are defined normatively
in [I-D.ramadan-moq-fec] Section 4.1.  This document does not
redefine FEC_CONFIG but specifies MMT-specific considerations for
its use.

### 7.1. MMT-Specific FEC_CONFIG Usage

When used with MMT packaging, the FEC_CONFIG fields map as follows:

- **FEC Algorithm**: Typically 0x02 (RaptorQ) for ATSC 3.0 ingest,
  or as specified in MMTP AL-FEC signaling
- **Repair Container**: 0x01 (MMTP) for MMT packaging.  See
  [I-D.ramadan-moq-fec] Section 4.6
- **Source Symbols Per Block**: Corresponds to the number of MFUs
  (or MMTP packets) covered by one FEC block
- **Interleave Depth**: Number of MPU frames spanned by each FEC
  block, matching the original broadcast FEC interleave depth
- **OTI**: For RaptorQ, the 12-byte concatenation of Common FEC OTI
  and Scheme-Specific FEC OTI per [RFC6330]

When Repair Container is 0x01 (MMTP), per-packet OTI embedded in
each MMTP repair packet (12 bytes at byte offset 21) provides
immediate decoder configuration without FEC_CONFIG.  FEC_CONFIG
is therefore OPTIONAL for MMTP container mode; if sent, its
fields are informational and SHOULD match the per-packet OTI
values.

See [I-D.ramadan-moq-fec] for the complete message format, field
definitions, algorithm registry, and precedence rules.

### 7.2. Repair Track Discovery

When FEC is enabled, the repair track uses the naming convention
defined in [I-D.ramadan-moq-fec] Section 6.1:

```
Source Track:  [namespace, track_name]
Repair Track:  [namespace, track_name, "repair"]
```

The subscriber MUST subscribe to the repair track separately.
The repair track uses lower priority (typically 7) so repair
symbols are dropped first under congestion.

### 7.3. Multicast Delivery of FEC_CONFIG

For multicast (SSM/ASM) delivery where bidirectional signaling is
not available, FEC_CONFIG parameters are conveyed via:

1. **MMTP AL-FEC Signaling (message_id=0x0203)**: In-band delivery
   per ISO/IEC 23008-1:2023 Amendment 1:2025

2. **MoQ Catalog Extension**: Out-of-band delivery via catalog JSON:

```json
{
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "interleaveDepth": 30,
        "symbolSize": 1312,
        "repairTrack": "video/repair"
      }
    },
    {
      "name": "video/repair",
      "packaging": "mmtp"
    }
  ]
}
```

## 8. Multicast Integration

MMT content can be delivered via IP multicast (SSM, AMT) and TreeDN
for scalable distribution.  Platform-specific delivery paths, SSM
group allocation, TreeDN integration with ISP router AMT deployment,
DePIN incentives, and IWA home gateway architecture are defined in
[I-D.ramadan-moq-multicast].

When MMT is delivered over multicast, MMTP packets are transmitted
as UDP datagrams with the standard MMTP header intact.  MoQ relays
at network edges terminate the multicast path and bridge MMTP
content into the MoQ application layer via QUIC/WebTransport.

For ATSC 3.0 and ARIB STD-B60 receivers, MMTP over SSM is the
native delivery path and requires no protocol translation.

## 9. ARIB STD-B60 Compatibility

ARIB STD-B60 (Japan's MMT-based broadcasting standard) uses the
same ISO 23008-1 foundation as ATSC 3.0 with the following specific
considerations:

### 9.1. Clock Reference

ARIB STD-B60 uses UTC wallclock timestamps in NTP short format,
consistent with ISO 23008-1.  The MMTP Timestamp field carries the
UTC send time of the packet, which maps to MoQ object timestamps.

The MMTP Timestamp uses NTP short format (32-bit: 16-bit seconds +
16-bit fractional seconds relative to NTP epoch):

```
MoQ Timestamp (seconds) = MMTP Timestamp upper 16 bits
                          + (lower 16 bits / 65536)
```

Note: This differs from MPEG-2 TS, which uses a 90kHz PTS/DTS clock.

### 9.2. 8K UHDTV Support

ARIB STD-B60 supports 8K UHDTV (7680x4320) via HEVC Main 10 profile
at Level 6.1 (4:2:0, 10-bit).
For efficient 8K delivery:

1. **Tiled Delivery**: MoQ Group boundaries SHOULD align with HEVC
   CTU rows for spatial random access
2. **Parallel Decoding**: Multiple MoQ tracks MAY carry tile regions
   for parallel decode
3. **Bandwidth**: 8K @ 60fps requires ~80-100 Mbps; FEC adds 25%

Example 8K track structure:
```
namespace: "live/8k"
tracks:
  - video/tile_0_0  (top-left quadrant)
  - video/tile_0_1  (top-right quadrant)
  - video/tile_1_0  (bottom-left quadrant)
  - video/tile_1_1  (bottom-right quadrant)
  - video/repair    (FEC for all tiles)
```

### 9.3. Hybridcast Integration

ARIB defines Hybridcast for companion device synchronization
(second screen experiences).  When bridging Hybridcast services:

1. Include timeline alignment metadata in MoQ catalog
2. Preserve MMT Composition Timeline (CT) information
3. Signal synchronization points via MoQ object timestamps

Catalog extension for Hybridcast:
```json
{
  "hybridcast": {
    "timelineId": "urn:isdb:timeline:ct",
    "ptsOffset": 0,
    "syncToleranceMs": 100
  }
}
```

### 9.4. Typical FEC Parameters

ARIB STD-B60 deployments typically use more conservative FEC
parameters than ATSC 3.0:

| Parameter | ARIB STD-B60 Typical | ATSC 3.0 Typical |
|-----------|-----------------|------------------|
| K (source symbols) | 64 | 32 |
| Interleave depth | 60 frames | 30 frames |
| Symbol size | 1316 bytes | 1312 bytes |
| Overhead | 20-30% | 25% |

Publishers SHOULD preserve original FEC parameters when ingesting
ARIB STD-B60 content.

## 10. Transport Hierarchy

Clients SHOULD attempt transports in preference order.  The transport
hierarchy for native clients (TV, mobile) and browser clients is
defined in [I-D.ramadan-moq-multicast] Section 6.

For MMT-specific deployments, AL-FEC (Section 6) is essential on
SSM/AMT paths since there is no retransmission.  On MoQ/QUIC paths,
FEC reduces retransmission latency but QUIC provides a reliable
fallback.

## 11. Catalog Signaling

The MoQ catalog signals MMT packaging via the standard CMSF
`packaging` field [I-D.ietf-moq-catalogformat].

### 11.1. Packaging Registration

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
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "codec": "avc1.64001f",
      "width": 1920,
      "height": 1080,
      "framerate": 30
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

### 11.2. S-TSID to MoQ Catalog Conversion

When ingesting ATSC 3.0 content delivered via ROUTE, generate MoQ
catalog from the S-TSID signaling table (defined in ATSC A/331 for
the ROUTE transport layer).  For MMT-delivered content, equivalent
parameters are obtained from MMTP signaling messages (MPT/MPI):

```
S-TSID Input:
<S-TSID>
  <RS sIpAddr="192.168.1.100" dIpAddr="232.1.1.50" dPort="5000">
    <LS tsi="1" bw="5000000">
      <SrcFlow rt="true">
        <ContentInfo>
          <MediaInfo contentType="video" repId="1080p"/>
        </ContentInfo>
        <Payload codePoint="128" formatId="2"/>
      </SrcFlow>
      <RepairFlow>
        <FECParameters maximumDelay="1000" overhead="25"
                       fecOTI="K=32;T=1312;Z=4">
          <ProtectedObject tsi="1">
            <SourceTOI x="0" y="255"/>
          </ProtectedObject>
        </FECParameters>
      </RepairFlow>
    </LS>
  </RS>
</S-TSID>

MoQ Catalog Output:
{
  "version": 1,
  "namespace": "atsc/service_1",
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "codec": "avc1.64001f",
      "bitrate": 5000000,
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "symbolSize": 1312,
        "interleaveDepth": 4,
        "repairTrack": "video/repair"
      }
    },
    {
      "name": "video/repair",
      "packaging": "mmtp"
    }
  ],
  "multicast": {
    "endpoints": [{
      "protocol": "ssm",
      "source": "192.168.1.100",
      "group": "232.1.1.50",
      "port": 5000,
      "tsi": 1,
      "tracks": ["video", "video/repair"]
    }]
  }
}
```

The `multicast` field in the output uses the extended format defined
in [I-D.ramadan-moq-multicast] Section 7.2.  Conversion rules:
- `RS@sIpAddr` → `multicast.endpoints[].source`
- `RS@dIpAddr` → `multicast.endpoints[].group`
- `RS@dPort` → `multicast.endpoints[].port`
- `LS@tsi` → `multicast.endpoints[].tsi`
- `LS@bw` → `track.bitrate`
- `FECParameters@overhead` → `fec.p` (computed as K × overhead / 100)
- `fecOTI` K,T,Z → `fec.k`, `fec.symbolSize`, `fec.interleaveDepth`

### 11.3. MoQ Catalog to S-TSID Conversion

When generating ATSC-compatible output, convert MoQ catalog to S-TSID:

```
MoQ Catalog Input:
{
  "tracks": [{
    "name": "video",
    "bitrate": 5000000,
    "fec": {
      "algorithm": "raptorq",
      "sourceSymbols": 32,
      "repairSymbols": 8,
      "symbolSize": 1312,
      "interleaveDepth": 4,
      "repairTrack": "video/repair"
    }
  }],
  "multicast": {
    "endpoints": [{
      "source": "192.168.1.100",
      "group": "232.1.1.50",
      "port": 5000,
      "tsi": 1
    }]
  }
}

S-TSID Output:
<S-TSID xmlns="tag:atsc.org,2016:XMLSchemas/ATSC3/Delivery/S-TSID/1.0/">
  <RS sIpAddr="192.168.1.100" dIpAddr="232.1.1.50" dPort="5000">
    <LS tsi="1" bw="5000000">
      <SrcFlow rt="true" minBuffSize="5000000">
        <ContentInfo>
          <MediaInfo contentType="video"/>
        </ContentInfo>
        <Payload codePoint="128" formatId="2" srcFecPayloadId="6"/>
      </SrcFlow>
      <RepairFlow>
        <FECParameters maximumDelay="133" overhead="25"
                       fecOTI="F=32;T=1312;Z=4;N=1;Al=8">
        </FECParameters>
      </RepairFlow>
    </LS>
  </RS>
</S-TSID>
```

Conversion rules:
- `multicast.endpoints[].source` → `RS@sIpAddr`
- `fec.p / fec.k × 100` → `FECParameters@overhead`
- `fec.interleaveDepth × frameDuration` → `FECParameters@maximumDelay`

### 11.4. Multicast Endpoint Catalog Extension

The multicast catalog extension — including simple and extended
formats and format detection rules — is defined in
[I-D.ramadan-moq-multicast] Section 7.

When converting S-TSID to MoQ catalog (Section 11.2), the `multicast`
field in the output catalog MUST conform to the extended format
defined in [I-D.ramadan-moq-multicast] Section 7.2, using the
`endpoints` array to represent per-TSI multicast groups.

## 12. Security Considerations

MMT content protection uses Common Encryption (CENC) which is
preserved through MoQ transport.  The MMTP header is not encrypted,
allowing relays to inspect packet type and sequence without
accessing media content.

### 12.1. Multicast Security

Multicast-specific security considerations (source authentication,
replay protection, AMT relay trust) are defined in
[I-D.ramadan-moq-multicast] Section 8.

## 13. IANA Considerations

This document requests registration of a MoQ Streaming Format
packaging value in the registry established by
[I-D.ietf-moq-catalogformat]:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "mmtp" | MMTP-encapsulated media per ISO 23008-1 | This document |

This document also requests registration of MoQ message type
(shared with [I-D.ramadan-moq-fec]):

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

## 14. References

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

## Appendix A. Bandwidth Comparison

FEC signaling bandwidth overhead:

### A.1. Per-Session Overhead (One-Time)

| Method | Size | When Sent |
|--------|------|-----------|
| FEC_CONFIG (0x50) | ~30 bytes | After SUBSCRIBE_OK |
| MMTP AL-FEC (0x0203) | ~42 bytes | First signaling packet |
| Catalog JSON FEC | ~100 bytes | Catalog fetch |

### A.2. Per-Object Overhead (Recurring)

| Method | Size | Frequency |
|--------|------|-----------|
| LOC Extension | 8 bytes | Every object |
| MoQ Extensions (CMAF+FEC) | ~10-12 bytes | Every CMAF source object |
| MMTP Header | 12 bytes | Every packet |
| FEC_CONFIG | 0 bytes | N/A (one-time) |

Note: The MoQ Extensions row accounts for FEC PID extension (8 bytes)
+ Object Length extension (1-4 bytes varint).  LOC source objects need
zero additional extensions for FEC — SBN and ESI are derived from MoQ
group_id and object_id.

### A.3. Total Overhead Analysis

For a 30fps stream with k=32 source symbols per FEC block:

| Method | Overhead/second | Overhead/hour |
|--------|-----------------|---------------|
| FEC_CONFIG | ~0.03 KB | ~0.1 KB |
| LOC Extension | ~0.24 KB | ~0.86 MB |
| MMTP (source only) | ~0.36 KB | ~1.3 MB |
| MMTP + Repair | ~0.50 KB | ~1.8 MB |

FEC_CONFIG has the lowest per-session overhead and zero
per-object overhead, making it ideal for unicast MoQ.

MMTP AL-FEC signaling has higher initial overhead but
works without bidirectional signaling (multicast).

### A.4. Recommendations

| Use Case | Recommended Method |
|----------|-------------------|
| MoQ Unicast | FEC_CONFIG |
| SSM Multicast | MMTP AL-FEC |
| Adaptive FEC | FEC_CONFIG + dynamic updates |
| Hybrid (MoQ + SSM) | Both (MMTP in-band, FEC_CONFIG for unicast) |

## Appendix B. S-TSID Conversion Example

Complete example showing bidirectional conversion between
ATSC S-TSID and MoQ catalog for a multi-track service.

### B.1. Original ATSC S-TSID

```xml
<?xml version="1.0" encoding="UTF-8"?>
<S-TSID xmlns="tag:atsc.org,2016:XMLSchemas/ATSC3/Delivery/S-TSID/1.0/">
  <RS sIpAddr="10.0.0.1" dIpAddr="232.1.1.10" dPort="5000">
    <!-- Video 1080p -->
    <LS tsi="1" bw="8000000">
      <SrcFlow rt="true" minBuffSize="8000000">
        <ContentInfo>
          <MediaInfo contentType="video" repId="1080p" lang="en"/>
        </ContentInfo>
        <EFDT>
          <FDT-Instance Expires="4294967295">
            <File Content-Location="video/1080p/init.mp4" TOI="1"/>
          </FDT-Instance>
        </EFDT>
        <Payload codePoint="128" formatId="2" srcFecPayloadId="6"/>
      </SrcFlow>
      <RepairFlow>
        <FECParameters maximumDelay="1000" overhead="25"
                       fecOTI="F=32;T=1312;Z=30;N=1;Al=8">
          <ProtectedObject tsi="1">
            <SourceTOI x="0" y="65535"/>
          </ProtectedObject>
        </FECParameters>
      </RepairFlow>
    </LS>
    <!-- Audio Stereo -->
    <LS tsi="2" bw="128000">
      <SrcFlow rt="true">
        <ContentInfo>
          <MediaInfo contentType="audio" repId="stereo" lang="en"/>
        </ContentInfo>
        <Payload codePoint="128" formatId="2"/>
      </SrcFlow>
    </LS>
  </RS>
</S-TSID>
```

### B.2. Converted MoQ Catalog

```json
{
  "version": 1,
  "namespace": "atsc/service_broadcast",
  "generatedAt": "2026-07-15T10:30:00Z",
  "tracks": [
    {
      "name": "video/1080p",
      "packaging": "mmtp",
      "codec": "avc1.64001f",
      "width": 1920,
      "height": 1080,
      "framerate": 30,
      "bitrate": 8000000,
      "language": "en",
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "symbolSize": 1312,
        "interleaveDepth": 30,
        "repairTrack": "video/1080p/repair"
      }
    },
    {
      "name": "video/1080p/repair",
      "packaging": "mmtp",
      "priority": 7
    },
    {
      "name": "audio/stereo",
      "packaging": "mmtp",
      "codec": "mp4a.40.2",
      "sampleRate": 48000,
      "channelCount": 2,
      "bitrate": 128000,
      "language": "en"
    }
  ],
  "multicast": {
    "endpoints": [
      {
        "protocol": "ssm",
        "source": "10.0.0.1",
        "group": "232.1.1.10",
        "port": 5000,
        "tsi": 1,
        "tracks": ["video/1080p", "video/1080p/repair"]
      },
      {
        "protocol": "ssm",
        "source": "10.0.0.1",
        "group": "232.1.1.10",
        "port": 5000,
        "tsi": 2,
        "tracks": ["audio/stereo"]
      }
    ],
    "amtFallback": {
      "discovery": "driad"
    }
  }
}
```

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
