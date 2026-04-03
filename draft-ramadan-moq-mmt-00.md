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
   (QUIC/WebTransport).  MMTP packets are MTU-sized (~1300 bytes) and
   work identically on reliable QUIC streams, QUIC datagrams, and
   multicast UDP — same packet, three transports.  This specification
   defines the mapping of MMT packets to MoQ objects as a passthrough
   format preserving the original MMTP wire format, field mappings
   between MoQ/LOC and MMTP, interaction with Application-Layer FEC
   including multi-path FEC delivery where MMTP repair packets are
   passed through identically across ATSC 3.0 broadcast, SSM,
   and MoQ CDN paths, relay transcoding between MMTP and LOC formats,
   and compatibility considerations for both ATSC 3.0 and ARIB STD-B60
   systems.

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
       4.4.  Ordering Model
       4.5.  Signaling Messages
   5.  Relay Transcoding
       5.1.  MMTP to LOC Unwrapping
       5.2.  CMAF and MPU ISOBMFF Compatibility
   6.  FEC Integration
       6.1.  MMTP as Universal FEC Container
       6.2.  Per-Group Encoding
       6.3.  Source Symbol Model
   7.  Field Mappings
       7.1.  MoQ Transport ↔ MMTP
       7.2.  MMTP → LOC Unwrapping
       7.3.  FEC Field Mapping
   8.  Multicast Integration
   9.  ARIB STD-B60 Compatibility
   10. Catalog Signaling
       10.1. Packaging Registration
       10.2. Backwards Compatibility
   11. Security Considerations
       11.1. Multicast Security
   12. IANA Considerations
   13. References
       13.1. Normative References
       13.2. Informative References
   Appendix A.  Implementation Guidance: Frame Ordering
   Authors' Addresses
```

## 1. Introduction

MPEG Media Transport (MMT) [I-D.bouazizi-mmtp] is a transport
protocol designed for delivery of multimedia content over
heterogeneous networks.  MMT is supported as an alternative transport in ATSC 3.0
broadcast television (Americas, South Korea) alongside the primary
ROUTE/DASH transport, and is the basis for ARIB STD-B60 (Japan).
MMT provides native support for:

- Media Processing Units (MPU, branded "mpuf" in ISOBMFF) as the
  media container, with structural similarities to fMP4
- Application-Layer FEC (AL-FEC) supporting multiple schemes
  including RaptorQ, LDPC, and Reed-Solomon
- Cross-layer signaling for adaptive streaming
- Hybrid delivery combining broadcast and broadband

MMTP packets are MTU-sized (~1300 bytes), fitting in UDP datagrams,
QUIC datagrams, and QUIC stream objects.  This means MMTP works
identically on all three transports — same packet, same FEC metadata,
same receiver logic.  MMTP is the FEC and datagram/multicast
container for MoQ: LOC video objects and CMAF chunks are frame-sized
(10-100KB+) and exceed the QUIC datagram MTU — MMTP fragmentation
is the necessary packetization layer for datagram and multicast
delivery.  MMTP handles video frame fragmentation, FEC metadata,
track routing, timestamps, and sequencing natively.

This document defines how MMT-encapsulated media can be transported
over MoQ [I-D.ietf-moq-transport], enabling:

1. **Broadcast-to-Unicast bridging**: Content from ATSC 3.0 or ARIB STD-B60
   broadcasts can be relayed to MoQ subscribers without transcoding
2. **Unified FEC**: MMTP is the FEC container for all transports —
   the same repair packets work on QUIC streams, QUIC datagrams,
   and multicast UDP
3. **Datagram delivery**: MMTP packet size (~T bytes) fits in QUIC
   datagrams, enabling unreliable FEC-protected delivery via the
   Datagram forwarding preference
4. **Multicast delivery**: MMTP over SSM is self-describing
   and operates as a unidirectional stream without MoQ signaling
5. **Single encoder pipeline**: Broadcasters can use one MMTP output
   for ATSC 3.0 RF, MoQ CDN, and multicast delivery simultaneously

### 1.1. Relationship to Other MoQ Media Formats

This specification complements existing MoQ packaging formats:

| Format | Container | Transport | FEC Support |
|--------|-----------|-----------|-------------|
| MSF (LOC) | WebCodecs chunks | Reliable QUIC stream only | N/A — QUIC retransmits |
| CMSF | CMAF fMP4 | Reliable QUIC stream only | N/A — QUIC retransmits |
| This spec (MMT) | MMTP + MPU (ISOBMFF) | All (streams, datagrams, multicast) | Native AL-FEC |

**LOC** and **CMAF** objects are frame-sized and require reliable
QUIC stream delivery (retransmission for loss recovery).  **MMTP**
fragments media into MTU-sized packets, enabling FEC-protected
delivery on QUIC datagrams and multicast UDP where retransmission
is not available.  This is a protocol constraint (MTU), not a
quality judgment.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

**MMTP**: MMT Protocol - the packet layer of MMT ([I-D.bouazizi-mmtp] Section 3)

**MPU**: Media Processing Unit - a self-contained media segment in MMT,
typically aligned with a Group of Pictures (GOP)

**MFU**: Media Fragment Unit - a single coded media frame within an MPU

**AL-FEC**: Application-Layer Forward Error Correction

**S-TSID**: Service-based Transport Session Instance Description -
ATSC 3.0 signaling table containing transport and FEC parameters

**SLT**: Service Layer Table - ATSC 3.0 bootstrap signaling

**MPT**: MMT Package Table - signaling table containing MMT asset info

**MPI**: MMT Presentation Information - presentation timing table

**ssbg_mode0**: Single Symbol per Block Group mode — one MMTP packet
equals one source symbol

## 3. MMT Overview

MMT uses a layered architecture:

```
+---------------------------------------------+
|              Application Layer              |
+---------------------------------------------+
|  MPU (Media Processing Unit)                |
|  +----------+----------+----------+-------+ |
|  |  MFU 0   |  MFU 1   |  MFU 2   |  ...  | |
|  |  (IDR)   |  (P)     |  (P)     |       | |
|  +----------+----------+----------+-------+ |
+---------------------------------------------+
|  MMTP (MMT Protocol)                        |
|  +------------------------------------------+|
|  | MMTP Header | Payload (MPU fragment)     ||
|  | 12 bytes    | Variable                   ||
|  +------------------------------------------+|
+---------------------------------------------+
|  Transport: UDP/IP (broadcast/multicast)    |
|             QUIC stream (reliable unicast)  |
|             QUIC datagram (unreliable)      |
+---------------------------------------------+
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
| Packet Sequence Number | Object ID |
| AL-FEC repair | Track "video/repair" |

For ATSC 3.0 ingest, the gateway preserves the original packet_id
assignments from the MPT (MMT Package Table).  For MoQ-originated
content using MMTP packaging, packet_id 0 is reserved for signaling
messages (Section 4.5), with media tracks assigned sequentially
starting from 1 (video=1, audio=2, etc.).  The mapping is signaled
in the catalog.

### 4.2. Object Payload

Each MoQ object carries exactly one MMTP packet, preserving the
original wire format:

```
MoQ Object Payload {
  MMTP Header (12+ bytes),
  MPU Fragment (MPU metadata or MFU payload),
}
```

Publishers MUST NOT reassemble, re-fragment, or modify MMTP packet
contents.  This enables relays to forward MMTP packets to
broadcast-capable receivers without transcoding, and preserves
the single-encoder pipeline where one MMTP output feeds both
ATSC 3.0 RF and MoQ CDN simultaneously.

This requirement applies to the MoQ object payload contents.
MoQ-level operations — including caching, priority-based dropping,
group expiry, and subscriber-specific filtering — operate on MoQ
object metadata (group_id, object_id, priority) and are unaffected.

The MMTP header is not encrypted, allowing relays to inspect packet
type, sequence, and FEC metadata without accessing media content.

MMTP packets are sized to fit within the network MTU (~T bytes,
typically 1312 bytes for 1500-byte MTU minus protocol overhead).
This MTU-sized design enables three delivery modes:

| MoQ Forwarding Preference | Transport | Use Case |
|---------------------------|-----------|----------|
| Stream | Reliable QUIC stream | Low-loss unicast |
| Datagram | QUIC datagram | Unreliable FEC-protected delivery |
| (multicast) | UDP datagram | Scalable broadcast/multicast |

When using the Datagram forwarding preference per
[I-D.ietf-moq-transport], each MoQ datagram object carries one
MMTP packet.  QUIC datagrams are unreliable — FEC is the loss
recovery mechanism.  Subscribers using datagram delivery SHOULD
subscribe to the repair track per [I-D.ramadan-moq-fec].

The same MMTP packet works on all three transports without
modification.  A receiver can combine source symbols from a QUIC
stream with repair symbols from multicast UDP for FEC recovery.

The latency difference between these delivery modes is significant:
QUIC reliable stream retransmission requires 1-2 RTT (~100-200ms
on wireless last-mile per typical WiFi/cellular jitter).  Datagram
delivery with FEC (interleave depth D=1 at 30fps) recovers in
~33ms — the time to receive one FEC block, with no round-trip
needed.  Per [I-D.ietf-moq-transport] Section 6.1.2, the Datagram
forwarding preference provides unreliable delivery suitable for
MMTP tracks on lossy links where FEC provides loss recovery.

### 4.3. Group Boundaries

Group boundaries align with MPU boundaries:
- Group N contains all MMTP packets of MPU N
- Object 0 of each group contains the MPU metadata (mmpu/moov boxes)
- Subsequent objects contain MMTP packets with MFU payloads
- The first object of each group SHOULD have RAP Flag = 1

Object 0 carries MPU metadata (mmpu/moov boxes), serving the same
role as a group header in other packaging formats.  Implementations
with a packaging-specific group header mechanism SHOULD map MPU
metadata to that mechanism.

### 4.4. Ordering Model

The MPU Sequence Number (mpuSeq) maps to MoQ Group ID (Section 4.1)
and serves as the ordering key across media decode, FEC recovery,
and MoQ transport.  Implementation guidance for frame ordering and
late-frame handling is provided in Appendix A.

### 4.5. Signaling Messages

ATSC 3.0 MMTP streams carry signaling messages (PA, MPI, clock
reference) in addition to media payloads.  When ingesting ATSC 3.0
content:

- **PA (Package Access) messages**: Publishers ingesting ATSC 3.0
  content MUST publish PA messages containing the MPT (MMT Package
  Table) on a dedicated signaling track (e.g., "signaling"), as the
  MPT provides the authoritative packet_id-to-asset mapping.
  MoQ-native publishers SHOULD publish PA messages for
  interoperability with ATSC 3.0 receivers.
- **MPI (MMT Presentation Information)**: Publishers MAY include
  MPI data in the catalog or as signaling track objects.
- **Clock references**: Mapped to MoQ object timestamps per
  Section 9.

Signaling messages that are only relevant to the broadcast transport
layer (e.g., IP-level configuration) SHOULD NOT be forwarded to
MoQ subscribers.

## 5. Relay Transcoding

Relays bridge between MMTP and LOC/CMAF formats to serve subscribers
with different capabilities.  MMTP is the canonical format for
FEC-enabled and multicast delivery; LOC is the lightweight format
for non-FEC reliable unicast.

### 5.1. MMTP to LOC Unwrapping

Relays that receive MMTP packets from broadcast, multicast, or
MMTP-publishing sources MAY offer the same content as LOC tracks
for subscribers that do not need FEC:

```json
{
  "tracks": [
    { "name": "video",     "packaging": "mmtp",
      "fec": { "algorithm": "raptorq", "repairTrack": "video/repair", ... } },
    { "name": "video/repair", "packaging": "mmtp" },
    { "name": "video-loc", "packaging": "loc", "altGroup": 1 }
  ]
}
```

When performing MMTP-to-LOC unwrapping, the relay:

1. Reassembles MFU fragments from MMTP packets (using Packet Counter
   and Packet Sequence Number)
2. Strips the MMTP header and MPU framing
3. Emits raw codec payload (NAL units) as LOC objects with appropriate
   LOC properties:
   - Timestamp: from MMTP header NTP timestamp
   - Video Frame Marking: RAP flag maps to keyframe indicator
   - Video Config: extracted from MPU metadata (mmpu/moov boxes)

The LOC track does not include FEC — subscribers on reliable QUIC
streams rely on QUIC retransmission for loss recovery.

MMTP-to-LOC transcoding SHOULD be performed at the origin or
first-hop relay, not at every edge server.  Relays SHOULD cache
the LOC output so downstream subscribers receive LOC objects
without repeated transcoding.

### 5.2. CMAF and MPU ISOBMFF Compatibility

MMTP's MPU (Media Processing Unit) container is ISOBMFF, branded
"mpuf" per [I-D.bouazizi-mmtp] Section 4.1.  This provides
structural compatibility with CMAF [I-D.ietf-moq-cmsf]:

| CMAF Concept | MPU/MFU Relationship |
|--------------|----------------------|
| Init segment (ftyp+moov) | MPU metadata (fragment_type=0) — byte-identical |
| Fragment (moof+mdat) | Reconstructed from MFU metadata + payloads (not byte-identical) |
| Sample | MFU payload (raw AVCC bytes extracted from mdat) |
| Segment boundary | MPU/group boundary |

CMAF fragments (moof+mdat) are NOT stored directly in MFU
packets.  MFU payloads contain raw codec samples (AVCC NAL units
for H.264, raw AAC frames for audio) extracted from the fMP4 mdat
box.  A relay reconstructing CMAF from MMTP builds the moof box
from MFU metadata:

- `moof.trun.sample_size` from MFU payload length
- `moof.trun.sample_flags` from MMTP RAP flag (keyframe)
- `moof.tfdt.baseMediaDecodeTime` from MMTP NTP timestamp
- `moof.mfhd.sequence_number` from MPU sequence number
- `mdat` payload from reassembled MFU bytes (already AVCC)

The reconstruction steps are:

1. Collecting MFU fragments for each MPU (using Packet Sequence
   Number ordering and fragmentation indicator)
2. Building ISOBMFF boxes from MFU metadata (moov from MPU
   metadata packet, moof+mdat constructed per sample)

CDN operations (ad injection, ABR switching, manifest
manipulation) work at the CMAF level after relay reassembly.  Ad
injection can also operate at the MMTP level by replacing MPU
sequences (groups) — the MPU sequence number provides natural
splice points.

## 6. FEC Integration

### 6.1. MMTP as Universal FEC Container

MMTP is the FEC container for all MoQ transports.  Both source and
repair tracks for FEC-enabled content use "mmtp" packaging per
[I-D.ramadan-moq-fec].

MMTP natively carries AL-FEC metadata in the MMTP header (FEC Type
field) and in per-packet FEC Payload ID and OTI fields (see
Section 3.1).  Repair track objects carry raw MMTP repair packets
(FEC Type=2) without modification:

- MMTP Header (12 bytes, FEC Type=2)
- FEC Payload ID (9 bytes: SS_Start, RSB_length, RS_ID)
- Object Transmission Information (12 bytes: RFC 6330 OTI)
- Repair Symbol Data (T bytes)

MoQ relays pass MMTP repair packets through as-is.  This passthrough
design enables multi-path FEC delivery: identical MMTP repair packets
across ATSC 3.0 RF, SSM, QUIC datagrams, and MoQ QUIC
streams.  Receivers apply the same FEC recovery logic regardless of
delivery path.

### 6.2. Per-Group Encoding

Each FEC source block is scoped to at most D consecutive MoQ
groups, where D is the interleave depth (default 1).  With D=1,
each group is independently encoded.  FEC blocks never span more
than D groups.

FEC parameters (algorithm, K, P, T, interleave depth) MAY be
signaled in the catalog for discovery before the first repair
packet.  FEC configuration is delivered via the AL-FEC signaling
message per [ISO.23008-1] Section C.6 (message_id=0x0203).  The
AL-FEC message carries maximum_k and maximum_p; SSB_length in
each repair packet carries the actual K for that block.  K may
vary per block (e.g., fewer symbols after a scene cut).
Mid-stream K changes are signaled by a new AL-FEC message at
source block boundaries.

### 6.3. Source Symbol Model

In ssbg_mode0 (the RECOMMENDED mode for MMTP), each MMTP source
packet (FEC Type=1) is one source symbol.  The 32-bit Source FEC
Payload ID (SS_ID per [ISO.23008-1] Section C.5.2) is a monotonic
counter appended after the payload.  The receiver determines
block membership and ESI using SS_Start and SSB_length from the
corresponding repair packet (Section 6.2 of [I-D.ramadan-moq-fec]).

This 1:1 mapping between MMTP packets and FEC source symbols means:
- No FEC-layer fragmentation or reassembly
- Each MoQ object = one MMTP packet = one source symbol
- Same packet on QUIC streams, QUIC datagrams, and multicast UDP

## 7. Field Mappings

This section defines the field-level mappings between MoQ transport,
LOC format, and MMTP.

### 7.1. MoQ Transport ↔ MMTP

The following fields map directly between MoQ transport framing and
the MMTP header:

```
MoQ track name       → MMTP packet_id (via catalog)
MoQ group_id         → MMTP MPU Sequence Number
MoQ object_id        → MMTP Packet Sequence Number within MPU
MoQ forwarding pref  → QUIC stream (reliable) or datagram (unreliable)
```

The MoQ track name to MMTP packet_id mapping is established in the
catalog.  Each track entry in the catalog corresponds to one MMTP
packet_id.  For multicast endpoints, the `packetId` field in the multicast
catalog endpoint makes this mapping explicit.

MoQ group_id and MMTP MPU Sequence Number are numerically identical.
MoQ object_id maps to MMTP Packet Sequence Number within the MPU.
Object 0 of each group carries MPU metadata (mmpu/moov); subsequent
objects carry MFU payloads.

### 7.2. MMTP → LOC Unwrapping

The field mapping for MMTP-to-LOC unwrapping (Section 5.1):

```
MMTP packet_id       → MoQ track name (via catalog)
MMTP MPU Seq Num     → MoQ group_id
MMTP MFU fragments   → Reassemble → LOC frame object
MMTP Timestamp       → LOC timestamp property
MMTP RAP flag        → LOC video frame marking
MMTP MPU metadata    → LOC video config property
```

### 7.3. FEC Field Mapping

MMTP carries FEC metadata natively, eliminating the need for MoQ
extension headers:

```
MMTP FEC Type=1      → Source symbol (ssbg_mode0: 1 packet = 1 symbol)
MMTP FEC Type=2      → Repair symbol
MMTP Source FEC PID   → SS_ID = source ESI
MMTP Repair FEC PID   → SS_Start (SBN), RSB_length (K), RS_ID (repair ESI)
AL-FEC signaling msg  → RFC 6330 OTI (12 bytes via message_id 0x0203)
Catalog fec fields    → Discovery before first packet
```

Source packets (FEC Type=1) carry a Source FEC Payload ID with the
source symbol's ESI.  Repair packets (FEC Type=2) carry a Repair FEC
Payload ID (SS_Start, RSB_length, RS_ID) plus the full 12-byte
RFC 6330 OTI, enabling self-describing FEC recovery without prior
catalog access.

## 8. Multicast Integration

MMT content can be delivered via IP multicast (SSM, AMT) and TreeDN
for scalable distribution.

When MMT is delivered over multicast, MMTP packets are transmitted
as UDP datagrams with the standard MMTP header intact.  MMTP is
self-describing — each packet carries track routing (packet_id),
timestamps, sequence numbers, and FEC metadata natively.  No
additional multicast framing or encapsulation is needed.

MMTP over SSM operates as a unidirectional data stream and does not
require a MoQ session for reception.  This is the native delivery
path for ATSC 3.0 and ARIB STD-B60 receivers.  MoQ relays at
network edges terminate the multicast path and bridge MMTP content
into the MoQ application layer via QUIC/WebTransport.

The multicast catalog extension for endpoint discovery and multi-path
delivery is defined in [I-D.ramadan-moq-multicast].

## 9. ARIB STD-B60 Compatibility

ARIB STD-B60 [ARIB-B60] (Japan) uses the same [ISO.23008-1]
foundation as ATSC 3.0, with the same MMTP wire format per
[I-D.bouazizi-mmtp] Section 3.  MMTP packets are interoperable
between ATSC 3.0 and ARIB STD-B60 systems at the wire level.

Key differences from ATSC 3.0:

- **Signaling**: ARIB STD-B60 uses TLV-SI (Type-Length-Value
  Service Information) for service discovery and signaling, while
  ATSC 3.0 uses SLT (Service Layer Table) and S-TSID (Service-based
  Transport Session Instance Description).  Both carry equivalent
  information for MMTP session configuration.

- **Timestamp epoch**: MMTP timestamps use NTP short format
  (32-bit: 16-bit seconds + 16-bit fractional) per
  [I-D.bouazizi-mmtp] Section 3 `timestamp` field.  ARIB STD-B60
  uses UTC epoch.  ATSC 3.0 may use GPS epoch in some
  implementations.  Publishers bridging between ATSC 3.0 (which
  may use GPS epoch) and ARIB STD-B60 (UTC epoch) MUST normalize
  timestamps to UTC for MoQ delivery.

- **FEC**: Both systems support AL-FEC via the same MMTP FEC Type
  fields.  FEC parameters are carried in S-TSID (ATSC 3.0) or
  TLV-SI (ARIB) for broadcast, and in the MoQ catalog `fec`
  extension (Section 10) for MoQ delivery.

MMTP timestamps map directly to MoQ object timestamps, enabling
seamless bridging between broadcast and MoQ delivery paths.

## 10. Catalog Signaling

The MoQ catalog signals MMT packaging via the `packaging` field
per [I-D.ietf-moq-catalogformat].

### 10.1. Packaging Registration

This document registers "mmtp" as a MoQ Streaming Format packaging
value:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "mmtp" | MMTP-encapsulated media per [I-D.bouazizi-mmtp] | This document |

MMTP packaging indicates that each MoQ object payload contains one
MMTP packet (source or repair).  The MMTP header provides packet
type, timestamp, sequence number, and FEC metadata natively.

Example catalog with FEC-enabled MMTP tracks and non-FEC LOC
alternative:

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
      },
      "fec": {
        "algorithm": "raptorq",
        "symbolSize": 1312,
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "repairTrack": "video/repair"
      }
    },
    { "name": "video/repair", "packaging": "mmtp" },
    { "name": "video-loc", "packaging": "loc", "altGroup": 1 }
  ]
}
```

Subscribers choose MMTP packaging for FEC-protected delivery on
any transport (streams, datagrams, multicast) or LOC packaging for
lightweight non-FEC delivery on reliable QUIC streams.

### 10.2. Backwards Compatibility

Publishers offering MMTP tracks SHOULD also offer LOC or CMAF
alternatives via the `altGroup` mechanism
([I-D.ietf-moq-catalogformat] Section 3.2).  Subscribers that do
not recognize the "mmtp" packaging value ignore those tracks and
subscribe to supported alternatives.  Unknown catalog extensions
are ignored per [I-D.ietf-moq-catalogformat] Section 3.1.

## 11. Security Considerations

MMT content protection uses Common Encryption (CENC) which is
preserved through MoQ transport.  The MMTP header is not encrypted,
allowing relays to inspect packet type and sequence without
accessing media content.

When bridging content-protected ATSC 3.0 streams, DRM license
acquisition differs: broadcast uses in-band EME key delivery, while
MoQ requires out-of-band license server URLs.  Publishers bridging
content-protected streams SHOULD include license server
configuration in the catalog or signaling track.  The specific DRM
bridging mechanism is out of scope for this document.

### 11.1. Multicast Security

For MMTP over multicast, the MMTP header is not encrypted, allowing
SSM (S,G) filtering and sequence-number-based replay detection.
MMTP-native authentication mechanisms per [I-D.bouazizi-mmtp] apply,
including signed_mmt_message for per-packet authentication.
AMT relay trust is delegated per [RFC7450].

## 12. IANA Considerations

This document requests registration of a MoQ Streaming Format
packaging value in the registry established by
[I-D.ietf-moq-catalogformat]:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "mmtp" | MMTP-encapsulated media per [I-D.bouazizi-mmtp] | This document |

## 13. References

### 13.1. Normative References

[RFC2119]
    Bradner, S., "Key words for use in RFCs to Indicate
    Requirement Levels", BCP 14, RFC 2119, March 1997.

[RFC8174]
    Leiba, B., "Ambiguity of Uppercase vs Lowercase in
    RFC 2119 Key Words", BCP 14, RFC 8174, May 2017.

[RFC6330]
    Luby, M., et al., "RaptorQ Forward Error Correction Scheme
    for Object Delivery", RFC 6330, August 2011.

[I-D.ietf-moq-transport]
    Curley, L., et al., "Media over QUIC Transport",
    draft-ietf-moq-transport (work in progress).

[I-D.bouazizi-mmtp]
    Bouazizi, I., "MMT Protocol (MMTP)",
    draft-bouazizi-mmtp-01 (work in progress).

### 13.2. Informative References

[I-D.ietf-moq-loc]
    Jennings, C., et al., "Low Overhead Container for Media
    over QUIC", draft-ietf-moq-loc (work in progress).

[I-D.ietf-moq-catalogformat]
    Curley, L., et al., "Common Media Server Format for
    Media over QUIC", draft-ietf-moq-catalogformat
    (work in progress).

[I-D.ietf-moq-cmsf]
    Law, W., "CMSF: A CMAF Compliant Implementation of
    MOQT Streaming Format", draft-ietf-moq-cmsf
    (work in progress).

[I-D.ramadan-moq-fec]
    Ramadan, O., "Forward Error Correction for Media over QUIC",
    draft-ramadan-moq-fec (work in progress).

[I-D.ramadan-moq-multicast]
    Ramadan, O., "Multicast Delivery and Endpoint Discovery
    for Media over QUIC", draft-ramadan-moq-multicast
    (work in progress).

[RFC7450]
    Bumgardner, G., "Automatic Multicast Tunneling", RFC 7450,
    DOI 10.17487/RFC7450, February 2015.

[ISO.23008-1]
    ISO, "Information technology - High efficiency coding and
    media delivery in heterogeneous environments - Part 1:
    MPEG media transport (MMT)", ISO/IEC 23008-1:2023.
    (Informative.  A freely available description of the MMTP
    wire format is provided by [I-D.bouazizi-mmtp].  The
    normative ISO standard provides additional detail including
    AL-FEC Annex C and complete signaling table definitions.)

[ATSC-A331]
    ATSC, "Signaling, Delivery, Synchronization, and Error
    Protection", A/331:2025, February 2025.

[ARIB-B60]
    ARIB, "MMT-Based Media Transport Scheme in Digital
    Broadcasting Systems", ARIB STD-B60, Version 1.14.

## Appendix A. Implementation Guidance: Frame Ordering

This appendix provides non-normative guidance for receivers handling
MMTP media frames delivered via MoQ.

The MPU Sequence Number (mpuSeq), mapped to MoQ Group ID, is the
ordering key across media decode, FEC recovery, and MoQ transport.
Receivers tracking the last decoded mpuSeq can apply the following
heuristics:

- **Keyframes (RAP=1)**: Always decode; reset decode-order tracking.
- **Delta frames (RAP=0)**: Decode only if mpuSeq > lastDecodedMpuSeq.
  Dropping out-of-order delta frames avoids reference frame mismatch
  when frames arrive via different delivery paths with different
  latencies.
- **FEC repair**: Use mpuSeq to derive SBN for source block alignment.

This ordering model applies identically across MoQ unicast, SSM,
and ATSC 3.0 broadcast delivery paths.  Receivers MAY
implement alternative strategies (e.g., jitter buffers, reordering
windows) appropriate to their deployment context.

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
