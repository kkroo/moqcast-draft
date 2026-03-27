# MPEG Media Transport (MMT) Packaging for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: January 2027                                       July 2026

        MPEG Media Transport (MMT) Packaging for Media over QUIC
                      draft-ramadan-moq-mmt-00

Abstract

   This document specifies the use of MPEG Media Transport (MMT) as a
   container format for Media over QUIC (MoQ).  It defines the mapping
   of MMTP packets to MoQ objects, Application-Layer FEC integration
   with multi-path repair passthrough across ATSC 3.0, SSM multicast,
   and MoQ CDN paths, bidirectional conversion between S-TSID and MoQ
   catalogs, and compatibility with ATSC 3.0 and ARIB STD-B60 systems.

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

Table of Contents

   1.  Introduction
       1.1.  Relationship to Other MoQ Media Formats
   2.  Terminology
   3.  MMTP Header Format
   4.  MoQ Object Mapping
       4.1.  Track Structure
       4.2.  Object Payload
       4.3.  Group Boundaries
       4.4.  Unified Ordering Model
   5.  MFU Mode
       5.1.  MFU Fragmentation
   6.  FEC Integration
       6.1.  Interleaving
       6.2.  OTI Signaling
   7.  FEC_CONFIG Usage
       7.1.  MMT-Specific Fields
       7.2.  Repair Track Discovery
       7.3.  Multicast FEC Signaling
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
   13. IANA Considerations
   14. References
   Appendix A. Bandwidth Comparison
   Appendix B. S-TSID Conversion Example
   Authors' Addresses
```

## 1. Introduction

MPEG Media Transport (MMT) [ISO.23008-1] delivers multimedia over
heterogeneous networks.  It is supported in ATSC 3.0 (Americas,
South Korea) alongside ROUTE/DASH, and is the basis for ARIB STD-B60
(Japan).  MMT provides Media Processing Units (MPU) as the media
container, Application-Layer FEC (AL-FEC), cross-layer signaling,
and hybrid broadcast/broadband delivery.

This document defines how MMT-encapsulated media is transported over
MoQ [I-D.ietf-moq-transport], enabling:

1. Broadcast-to-unicast bridging without transcoding
2. Unified FEC via MoQ repair tracks per [I-D.ramadan-moq-fec]
3. Multicast promotion per [I-D.ramadan-moq-multicast]
4. Bidirectional S-TSID / MoQ catalog conversion

### 1.1. Relationship to Other MoQ Media Formats

| Format | Container | Primary Use | FEC |
|--------|-----------|-------------|-----|
| MSF (LOC) | WebCodecs chunks | Low-latency unicast | No |
| CMSF | CMAF fMP4 | Adaptive streaming | No |
| CARP | CMAF fMP4 | ABR delivery | No |
| This spec | MMTP + MPU (ISOBMFF) | Broadcast bridge | Yes |

MMT packaging is RECOMMENDED when ingesting ATSC 3.0 or ARIB STD-B60
broadcasts, when FEC protection is needed for multicast, or when
interoperability with broadcast receivers is required.

CMAF packaging (CMSF/CARP) is RECOMMENDED when content originates as
DASH/HLS without multicast delivery requirements.

## 2. Terminology

**MMTP**: MMT Protocol -- the packet layer of MMT (ISO 23008-1 Clause 8)

**MPU**: Media Processing Unit -- a self-contained media segment,
typically aligned with a GOP

**MFU**: Media Fragment Unit -- a single coded media frame within an MPU

**AL-FEC**: Application-Layer Forward Error Correction

**S-TSID**: Service-based Transport Session Instance Description --
ATSC 3.0 signaling table with transport and FEC parameters

**SLT**: Service Layer Table -- ATSC 3.0 bootstrap signaling

**MPT**: MMT Package Table -- signaling table with MMT asset info

**MPI**: MMT Presentation Information -- presentation timing table

## 3. MMTP Header Format

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
- **Timestamp**: UTC wallclock in NTP short format; maps to MoQ
  object timestamp
- **Packet Sequence Number**: Maps to MoQ object ID within group
- **FEC Type**: 0=no AL-FEC, 1=source, 2=repair, 3=reserved
- **RAP Flag**: 1 indicates Random Access Point

## 4. MoQ Object Mapping

### 4.1. Track Structure

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

Publishers MAY strip the MMTP header if subscribers negotiate raw
ISOBMFF delivery via the catalog `container` field.  Stripping loses
FEC Type, RAP flag, and timestamp metadata; receivers MUST then rely
on catalog and MoQ object headers for this information.

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

The MPU Sequence Number is the single authoritative ordering key
across media decode, FEC recovery, and MoQ transport:

```
MPU Sequence Number (mpuSeq)
  |
  +-- MoQ Group ID        (mpuSeq -> Group ID)
  |     \-- Decode order   -- H.264/H.265 requires frames in sequence
  |
  +-- FEC Source Block     (SBN = mpuSeq / K, per [I-D.ramadan-moq-fec])
  |     \-- ESI alignment  -- source symbols map to sequential mpuSeqs
  |
  \-- MMTP Timestamp       (wallclock presentation time)
        \-- Presentation    -- monotonic within decode order
```

Receivers MUST track the last decoded mpuSeq and:

- **Keyframes (RAP=1)**: Always decode; reset decode-order tracking
- **Delta frames (RAP=0)**: Decode only if mpuSeq > lastDecodedMpuSeq;
  drop out-of-order delta frames to prevent reference frame mismatch
- **FEC repair**: Use mpuSeq to derive SBN for block alignment

This ordering model applies identically across MoQ unicast (QUIC
streams), SSM multicast (UDP), and ATSC 3.0 broadcast (ROUTE/MMTP).
The decoder MUST enforce mpuSeq ordering regardless of delivery path.

Receivers SHOULD NOT rely on transport-layer ordering.  Multicast
and broadcast paths may reorder MFUs due to FEC interleaving,
network path reordering, or relay buffering.

Fragment reassembly (FI=1,2,3 within a single MFU) and decode
ordering (mpuSeq between MFUs) are separate concerns.  The
reassembler buffers fragments by fragment counter; the decoder
tracks lastDecodedMpuSeq and drops out-of-order delta frames.

## 5. MFU Mode

For ultra-low-latency delivery, MMT supports MFU mode where each
video frame is a separate MoQ object:

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

MFU mode enables per-frame FEC protection, frame-level
prioritization (IDR vs P/B), and lower end-to-end latency.

### 5.1. MFU Fragmentation

When MFU size exceeds typical MTU (1200-1400 bytes for QUIC),
publishers SHOULD:

1. Fragment MFU across multiple MMTP packets
2. Set Packet Counter Flag (C=1) for reassembly tracking
3. Publish all fragments as a single MoQ object
4. Include the complete MFU in the object payload

MMTP fragmentation is transparent to MoQ; each MoQ object
represents a complete, potentially multi-packet MFU.  Receivers
reassemble using Packet Counter and Packet Sequence Number before
media processing.

## 6. FEC Integration

MMT AL-FEC supports RaptorQ [RFC6330], LDPC [RFC5170], and
Reed-Solomon [RFC5510], with parameters signaled via MMTP signaling
messages or S-TSID (ATSC 3.0).  For MoQ, FEC repair uses the model
in [I-D.ramadan-moq-fec]:

```
Source Track: video
  \-- Objects: MMTP packets with FEC Type=0 or 1 (source)

Repair Track: video/repair
  \-- Objects: MMTP packets with FEC Type=2 (repair, passthrough)
```

Repair track objects carry raw MMTP repair packets without
modification, using Repair Container 0x01 ("mmtp") per
[I-D.ramadan-moq-fec] Section 4.6.  MoQ relays pass MMTP repair
packets through as-is.

This passthrough design enables multi-path FEC: the same MMTP
repair packets are delivered identically via ATSC 3.0 RF broadcast,
SSM multicast, and MoQ CDN.  Receivers implement one FEC repair
parser for all paths.  See [I-D.ramadan-moq-fec] Appendix C for
the byte-offset extraction recipe.

### 6.1. Interleaving

MMT AL-FEC interleaves source symbols across multiple MFUs:

```
MFU:        0    1    2    3    4    5    6    7
            |    |    |    |    |    |    |    |
Block 0:    S0   S1   S2   S3                      -> R0, R1
Block 1:                        S4   S5   S6   S7  -> R2, R3
```

Typical interleave depths:

- ATSC 3.0: 30-60 frames (~1-2s at 30fps)
- ARIB STD-B60: 60 frames (~2s at 30fps)
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

For MoQ, OTI is signaled via FEC_CONFIG per [I-D.ramadan-moq-fec]
Section 4.

## 7. FEC_CONFIG Usage

The FEC_CONFIG message format is defined normatively in
[I-D.ramadan-moq-fec] Section 4.1.  This section specifies
MMT-specific considerations.

### 7.1. MMT-Specific Fields

When used with MMT packaging, FEC_CONFIG fields map as follows:

- **FEC Algorithm**: Typically 0x02 (RaptorQ) for ATSC 3.0 ingest
- **Repair Container**: 0x01 (MMTP) per [I-D.ramadan-moq-fec]
  Section 4.6
- **Source Symbols Per Block**: Number of MFUs (or MMTP packets)
  per FEC block
- **Interleave Depth**: Number of MPU frames per FEC block,
  matching the original broadcast FEC interleave depth
- **OTI**: For RaptorQ, the 12-byte concatenation of Common FEC OTI
  and Scheme-Specific FEC OTI per [RFC6330]

When Repair Container is 0x01 (MMTP), per-packet OTI embedded in
each MMTP repair packet (12 bytes at byte offset 21) provides
immediate decoder configuration.  FEC_CONFIG is therefore OPTIONAL
for MMTP container mode; if sent, its fields SHOULD match the
per-packet OTI values.

### 7.2. Repair Track Discovery

When FEC is enabled, repair tracks use the naming convention in
[I-D.ramadan-moq-fec] Section 6.1:

```
Source Track:  [namespace, track_name]
Repair Track:  [namespace, track_name, "repair"]
```

The subscriber MUST subscribe to the repair track separately.
The repair track uses lower priority (typically 7) so repair
symbols are dropped first under congestion.

### 7.3. Multicast FEC Signaling

For multicast delivery without bidirectional signaling, FEC_CONFIG
parameters are conveyed via:

1. **MMTP AL-FEC Signaling (message_id=0x0203)**: In-band delivery
   per ISO/IEC 23008-1:2023 Amendment 1:2025

2. **MoQ Catalog Extension**: Out-of-band via catalog JSON:

```json
{
  "tracks": [{
    "name": "video",
    "fec": {
      "algorithm": "raptorq",
      "sourceSymbols": 32,
      "repairSymbols": 8,
      "interleaveDepth": 30,
      "symbolSize": 1312,
      "repairContainer": "mmtp",
      "repairTrack": "video/repair"
    }
  }]
}
```

## 8. Multicast Integration

MMT content can be delivered via IP multicast (SSM, AMT) and TreeDN.
Platform-specific delivery paths, SSM group allocation, TreeDN
integration, and home gateway architecture are defined in
[I-D.ramadan-moq-multicast].

Over multicast, MMTP packets are transmitted as UDP datagrams with
the standard MMTP header intact.  MoQ relays at network edges bridge
MMTP content into MoQ via QUIC/WebTransport.

For ATSC 3.0 and ARIB STD-B60 receivers, MMTP over SSM is the
native delivery path and requires no protocol translation.

## 9. ARIB STD-B60 Compatibility

ARIB STD-B60 (Japan) uses the same ISO 23008-1 foundation as
ATSC 3.0 with the following considerations.

### 9.1. Clock Reference

ARIB STD-B60 uses UTC wallclock timestamps in NTP short format
(32-bit: 16-bit seconds + 16-bit fractional seconds):

```
MoQ Timestamp (seconds) = MMTP Timestamp upper 16 bits
                          + (lower 16 bits / 65536)
```

This differs from MPEG-2 TS, which uses a 90kHz PTS/DTS clock.

### 9.2. 8K UHDTV Support

ARIB STD-B60 supports 8K UHDTV (7680x4320) via HEVC Main 10
profile at Level 6.1.  For efficient 8K delivery:

1. MoQ Group boundaries SHOULD align with HEVC CTU rows for
   spatial random access
2. Multiple MoQ tracks MAY carry tile regions for parallel decode
3. 8K at 60fps requires ~80-100 Mbps; FEC adds ~25%

Example 8K track structure:

```
namespace: "live/8k"
tracks:
  - video/tile_0_0  (top-left)
  - video/tile_0_1  (top-right)
  - video/tile_1_0  (bottom-left)
  - video/tile_1_1  (bottom-right)
  - video/repair    (FEC for all tiles)
```

### 9.3. Hybridcast Integration

ARIB Hybridcast enables companion device synchronization.  When
bridging Hybridcast services, publishers SHOULD:

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

| Parameter | ARIB STD-B60 | ATSC 3.0 |
|-----------|-------------|----------|
| K (source symbols) | 64 | 32 |
| Interleave depth | 60 frames | 30 frames |
| Symbol size | 1316 bytes | 1312 bytes |
| Overhead | 20-30% | 25% |

Publishers SHOULD preserve original FEC parameters when ingesting
ARIB STD-B60 content.

## 10. Transport Hierarchy

Clients SHOULD attempt transports in preference order.  The
transport hierarchy is defined in [I-D.ramadan-moq-multicast]
Section 6.

For MMT deployments, AL-FEC (Section 6) is essential on SSM/AMT
paths since there is no retransmission.  On MoQ/QUIC paths, FEC
reduces retransmission latency but QUIC provides reliable fallback.

## 11. Catalog Signaling

The MoQ catalog indicates MMT container format and multicast
endpoints.

### 11.1. Container Values

| Value | Description |
|-------|-------------|
| "isobmff" | Raw ISOBMFF/MPU (MMTP header stripped) |
| "mmtp" | MMTP-encapsulated MPU (ISOBMFF) |
| "mfu" | MMTP MFU mode (per-frame objects) |

Example:

```json
{
  "tracks": [{
    "name": "video",
    "container": "mmtp",
    "codec": "avc1.64001f",
    "width": 1920,
    "height": 1080,
    "framerate": 30
  }]
}
```

When FEC is enabled for 'isobmff' or 'mfu' containers over MoQ
unicast, source track objects SHOULD carry the FEC Payload ID and
Object Length extension headers defined in [I-D.ramadan-moq-fec].

### 11.2. S-TSID to MoQ Catalog Conversion

When ingesting ATSC 3.0 content, generate MoQ catalog from the
S-TSID signaling table (ATSC A/331).  For MMT-delivered content,
equivalent parameters come from MMTP signaling messages (MPT/MPI):

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
  "tracks": [{
    "name": "video",
    "container": "mmtp",
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
  }],
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

The `multicast` field uses the extended format in
[I-D.ramadan-moq-multicast] Section 7.2.  Conversion rules:

- `RS@sIpAddr` -> `multicast.endpoints[].source`
- `RS@dIpAddr` -> `multicast.endpoints[].group`
- `RS@dPort` -> `multicast.endpoints[].port`
- `LS@tsi` -> `multicast.endpoints[].tsi`
- `LS@bw` -> `track.bitrate`
- `FECParameters@overhead` -> `fec.p` (K * overhead / 100)
- `fecOTI` K,T,Z -> `fec.k`, `fec.symbolSize`, `fec.interleaveDepth`

### 11.3. MoQ Catalog to S-TSID Conversion

When generating ATSC-compatible output, convert MoQ catalog to
S-TSID:

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

- `multicast.endpoints[].source` -> `RS@sIpAddr`
- `fec.p / fec.k * 100` -> `FECParameters@overhead`
- `fec.interleaveDepth * frameDuration` -> `FECParameters@maximumDelay`

### 11.4. Multicast Endpoint Catalog Extension

The multicast catalog extension is defined in
[I-D.ramadan-moq-multicast] Section 7.

When converting S-TSID to MoQ catalog (Section 11.2), the
`multicast` field MUST conform to the extended format in
[I-D.ramadan-moq-multicast] Section 7.2.

## 12. Security Considerations

MMT content protection uses Common Encryption (CENC), which is
preserved through MoQ transport.  The MMTP header is not encrypted,
allowing relays to inspect packet type and sequence without
accessing media content.

Multicast-specific security (source authentication, replay
protection, AMT relay trust) is defined in
[I-D.ramadan-moq-multicast] Section 8.

## 13. IANA Considerations

This document requests registration of container format identifiers
in the "MoQ Container Formats" registry:

| Value | Description | Reference |
|-------|-------------|-----------|
| "mmtp" | MMTP-encapsulated MPU (ISOBMFF) | This document |
| "mfu" | MMTP MFU mode | This document |

This document also requests registration of MoQ message type
(shared with [I-D.ramadan-moq-fec]):

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

## 14. References

### Normative References

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
    Jennings, C., et al., "Low Overhead Media Container",
    draft-ietf-moq-loc (work in progress).

[I-D.ramadan-moq-fec]
    Ramadan, O., "Forward Error Correction for Media over QUIC",
    draft-ramadan-moq-fec (work in progress).

### Informative References

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

LOC source objects need zero additional FEC extensions -- SBN and
ESI are derived from MoQ group_id and object_id.

### A.3. Total Overhead Analysis

For a 30fps stream with K=32 source symbols per FEC block:

| Method | Overhead/second | Overhead/hour |
|--------|-----------------|---------------|
| FEC_CONFIG | ~0.03 KB | ~0.1 KB |
| LOC Extension | ~0.24 KB | ~0.86 MB |
| MMTP (source only) | ~0.36 KB | ~1.3 MB |
| MMTP + Repair | ~0.50 KB | ~1.8 MB |

### A.4. Recommendations

| Use Case | Recommended Method |
|----------|-------------------|
| MoQ Unicast | FEC_CONFIG |
| SSM Multicast | MMTP AL-FEC |
| Adaptive FEC | FEC_CONFIG + dynamic updates |
| Hybrid (MoQ + SSM) | Both (MMTP in-band + FEC_CONFIG for unicast) |

## Appendix B. S-TSID Conversion Example

Complete bidirectional conversion between ATSC S-TSID and MoQ
catalog for a multi-track service.

### B.1. Original ATSC S-TSID

```xml
<?xml version="1.0" encoding="UTF-8"?>
<S-TSID xmlns="tag:atsc.org,2016:XMLSchemas/ATSC3/Delivery/S-TSID/1.0/">
  <RS sIpAddr="10.0.0.1" dIpAddr="232.1.1.10" dPort="5000">
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
  "tracks": [
    {
      "name": "video/1080p",
      "container": "mmtp",
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
      "container": "fec-repair",
      "priority": 7
    },
    {
      "name": "audio/stereo",
      "container": "mmtp",
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
