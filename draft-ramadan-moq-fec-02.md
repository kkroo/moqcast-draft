# Forward Error Correction for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: January 2027                                       July 2026

        Forward Error Correction for Media over QUIC (MoQ)
                      draft-ramadan-moq-fec-02

Abstract

   This document specifies a mechanism for transmitting Forward Error
   Correction (FEC) repair data alongside source media in Media over
   QUIC (MoQ) sessions.  It defines signaling for FEC configuration
   via catalog extensions, conventions for repair track naming, and
   the format of repair objects.  The mechanism supports RaptorQ
   (RFC 6330), Reed-Solomon (RFC 5510), and XOR-based FEC schemes,
   enabling receivers to recover from packet loss without
   retransmission latency.  A repair container abstraction allows
   repair objects to carry either condensed-framed symbols or
   container-specific payloads (e.g., MMTP repair packets), enabling
   multi-path FEC delivery where receivers apply identical recovery
   logic regardless of whether data arrives via MoQ/QUIC, SSM
   multicast, or ATSC 3.0 broadcast.  This specification is designed
   for compatibility with ATSC 3.0 (A/331) and ARIB STD-B60 broadcast
   systems.

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
   3.  Protocol Overview
   4.  FEC Configuration
       4.1.  FEC Algorithms
       4.2.  RaptorQ Object Transmission Information
       4.3.  Reed-Solomon Parameters
       4.4.  Repair Container Types
   5.  Catalog FEC Extension
       5.1.  Catalog Fields
   6.  Repair Track Convention
       6.1.  Track Naming
       6.2.  Subscription Model
   7.  Repair Object Format
       7.1.  Condensed Repair Format
       7.2.  Repair Symbols
   8.  MoQ Extension Headers for FEC
       8.1.  FEC Payload ID Extension
       8.2.  Object Length Extension
       8.3.  Auth Tag Extension
   9.  Block and Group Alignment
       9.1.  Single-Group Blocks
       9.2.  Multi-Group Blocks
   10. Interleaving
   11. Priority and Congestion
   12. Hybrid Unicast and Multicast Delivery
       12.1. Multicast and AMT Delivery
       12.2. Multi-Path FEC Delivery
   13. ATSC 3.0 Compatibility
   14. Interaction with CMAF Packaging
       14.1. CARP Integration
   15. Security Considerations
       15.1. FEC Does Not Provide Confidentiality
       15.2. Repair Symbol Manipulation
       15.3. Amplification
       15.4. Multicast FEC Security
   16. IANA Considerations
       16.1. FEC Algorithm Registry
       16.2. MoQ Streaming Format Packaging
       16.3. MoQ Object Extension Header Types
   17. References
       17.1. Normative References
       17.2. Informative References
   Appendix A.  Example Message Flows
       A.3. Per-Object Overhead Comparison
   Appendix B.  Catalog Example
   Appendix C.  MMTP Repair Container Mapping (Informational)
   Authors' Addresses
```

## 1. Introduction

Media over QUIC (MoQ) [I-D.ietf-moq-transport] provides a
publish/subscribe transport for real-time media.  While QUIC provides
reliable delivery through retransmission, the latency incurred by
retransmission can be unacceptable for ultra-low-latency (ULL)
applications such as live sports, gaming, and interactive broadcasts.

Forward Error Correction (FEC) enables receivers to recover lost
packets using redundant repair data, without waiting for
retransmissions.  This is particularly valuable in wireless
environments where burst packet loss is common.

This document defines:

1. A catalog extension for FEC parameter signaling
2. A naming convention for repair tracks
3. The format of repair objects containing FEC symbols
4. Interleaving strategies to protect against burst loss
5. Compatibility mappings for ATSC 3.0 and ARIB STD-B60 broadcast systems
6. Multi-path delivery where FEC symbols arrive from multiple transport
   paths (MoQ/QUIC, SSM multicast, ATSC 3.0 broadcast) with identical
   recovery at the receiver

The mechanism is designed to complement existing MoQ media packaging
formats including CMAF [I-D.ietf-moq-cmsf], LOC [I-D.ietf-moq-loc], and
MMT [I-D.ramadan-moq-mmt] by operating as a separate protection layer.

## 2. Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

**Source Track**: A MoQ track carrying media data (e.g., video, audio).

**Repair Track**: A MoQ track carrying FEC repair symbols for a
corresponding source track.

**Source Block**: A sequence of K source symbols over which FEC
encoding is performed.

**Source Symbol**: A fixed-size unit of source data used for FEC
encoding.

**Repair Symbol**: A symbol generated by the FEC encoder that can be
used to recover lost source symbols.

**FEC Block**: The combination of K source symbols and P repair
symbols.

**Interleave Depth**: The number of media units (e.g., frames, chunks)
spanned by a single source block.

**Object Transmission Information (OTI)**: Parameters required to
configure a RaptorQ decoder, as defined in [RFC6330].

**Source Block Number (SBN)**: A zero-based index identifying a
specific source block within a session.

## 3. Protocol Overview

```
Publisher                              Subscriber
    |                                       |
    |  (Subscriber fetches catalog,         |
    |   discovers FEC params & repair track)|
    |                                       |
    |  SUBSCRIBE (source track)             |
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK                         |
    |-------------------------------------->|
    |                                       |
    |  SUBSCRIBE (repair track)             |
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK                         |
    |-------------------------------------->|
    |                                       |
    |  OBJECT (source, group=N, obj=0..K-1) |
    |-------------------------------------->|
    |                                       |
    |  OBJECT (repair, group=N, obj=0..P-1) |
    |-------------------------------------->|
    |                                       |
```

The protocol operates as follows:

1. Subscriber fetches the catalog, discovers FEC parameters and the repair track name (Section 5)
2. Subscriber sends SUBSCRIBE for source track
3. Publisher responds with SUBSCRIBE_OK
4. Subscriber sends SUBSCRIBE for repair track
5. Publisher sends source objects followed by repair objects
6. Subscriber uses repair symbols to recover any lost source symbols

## 4. FEC Configuration

### 4.1. FEC Algorithms

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | N/A       |
| 0x01  | XOR       | This document |
| 0x02  | RaptorQ   | [RFC6330] |
| 0x03  | Reed-Solomon (GF2^8) | [RFC5510] |
| 0x04-0xFF | Reserved | IANA |

**XOR (0x01)**: Simple XOR-based FEC.  Each repair symbol is the XOR
of all source symbols in the block.  Can recover exactly one lost
symbol per block.

**RaptorQ (0x02)**: RaptorQ fountain code per [RFC6330].  Can recover
from loss of any symbols as long as K symbols (source or repair) are
received.  Recommended for most applications.

**Reed-Solomon (0x03)**: Reed-Solomon erasure code over GF(2^8) per
[RFC5510].  Can recover up to P lost symbols.  Suitable for
applications requiring exact recovery guarantees or interoperability
with ATSC 3.0 systems using RS FEC.

### 4.2. RaptorQ Object Transmission Information

When FEC Algorithm is RaptorQ (0x02), the OTI field contains the
concatenation of the 8-byte Common FEC OTI (Section 3.3.2 of
[RFC6330]) and the 4-byte Scheme-Specific FEC OTI (Section 3.3.3
of [RFC6330]), totaling 12 bytes:

```
RaptorQ OTI {
  Transfer Length (40),
  Reserved (8),
  Symbol Size (16),
  Number of Source Blocks (8),
  Number of Sub-Blocks (16),
  Symbol Alignment (8),
}
```

Note: For MoQ usage, Transfer Length is typically computed per source
block from the cumulative size of source objects in the block.

### 4.3. Reed-Solomon Parameters

When FEC Algorithm is Reed-Solomon (0x03), the OTI field contains:

```
Reed-Solomon OTI {
  Field Size (8),          // Always 8 for GF(2^8)
  Max Source Symbols (16), // Maximum K value
  Max Repair Symbols (16), // Maximum P value
  Reserved (24),
}
```

### 4.4. Repair Object Format Selection

The format of repair object payloads is determined by the repair
track's packaging type in the catalog.  Each packaging type defines
its own repair object layout:

| Repair Track Packaging | Repair Object Format | Reference |
|------------------------|---------------------|-----------|
| condensed | Raw repair symbol, SBN/ESI from MoQ framing (Section 7.1) | This document |
| mmtp | MMTP repair packet (type 0x03) per ISO 23008-1 (Appendix C) | [I-D.ramadan-moq-mmt] |
| loc | LOC object with repair symbol payload (Section 7.3) | [I-D.ietf-moq-loc] |

The repair track's packaging MUST match the source track's
packaging when the source uses a format with native FEC support
(mmtp, loc).  When the source uses cmaf packaging, the repair
track uses condensed packaging because CMAF does not define a
native repair framing.

Receivers determine the repair format by inspecting the repair
track's packaging field in the catalog.  No separate "repair
container" signaling is needed — the packaging field IS the signal.

## 5. Catalog FEC Extension

### 5.1. Catalog Fields

MoQ catalogs [I-D.ietf-moq-catalogformat] signal FEC configuration
at the track level.  The catalog is the sole mechanism for
communicating FEC parameters to subscribers:

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
        "symbolSize": 1312,
        "interleaveDepth": 4,
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

The repair track is a regular catalog track whose packaging field
indicates the repair object format (Section 4.4).  In this example,
both source and repair use mmtp packaging — the MMTP type 0x03
repair packet format carries FEC Payload ID and OTI natively.

Field definitions for the `fec` object:

**algorithm** (string, REQUIRED if fec present): FEC scheme identifier.
  - "none": No FEC
  - "xor": Simple XOR
  - "raptorq": RaptorQ per RFC 6330
  - "reed-solomon": Reed-Solomon per RFC 5510

**sourceSymbols** (integer, REQUIRED): K value - source symbols per
FEC block.

**repairSymbols** (integer, REQUIRED): P value - repair symbols per
FEC block.

**symbolSize** (integer, REQUIRED): T value - bytes per symbol.

**interleaveDepth** (integer, OPTIONAL): Number of media units per
FEC block.  Default is 1.

**repairTrack** (string, REQUIRED): Track name for repair symbols,
following the convention in Section 6.1.

## 6. Repair Track Convention

### 6.1. Track Naming

For a source track with name components [namespace, track_name], the
corresponding repair track MUST be named:

```
[namespace, track_name, "repair"]
```

Examples:

| Source Track | Repair Track |
|--------------|--------------|
| ["live", "video"] | ["live", "video", "repair"] |
| ["broadcast", "video", "1080p"] | ["broadcast", "video", "1080p", "repair"] |
| ["stream", "audio", "en"] | ["stream", "audio", "en", "repair"] |

This convention allows subscribers to discover repair tracks without
additional signaling.

### 6.2. Subscription Model

Subscribers SHOULD subscribe to the source track first, then subscribe
to the repair track if:

1. The catalog indicates FEC is available for the source track, AND
2. The subscriber desires FEC protection

Publishers MUST NOT require subscription to repair tracks.  Repair
tracks are optional and subscribers MAY choose to rely solely on
QUIC retransmission.

Subscribers discover FEC availability and repair track names from the
catalog (Section 5.1).

## 7. Repair Object Format

### 7.1. Condensed Repair Format

This section defines the condensed repair format, used when the
repair track's packaging is "condensed" in the catalog.  This is
the default repair format for CMAF source tracks, which do not
have a native FEC framing.

When the repair track uses a different packaging (e.g., "mmtp" per
Appendix C, or "loc" per Section 7.3), repair object payloads
follow the packaging-specific format and this section does not apply.

Each MoQ object on a repair track contains exactly one repair symbol.
The object payload is the raw repair symbol data (Symbol Size bytes),
with no additional header.

```
Repair Object Payload {
  Repair Symbol (Symbol Size bytes),
}
```

SBN and ESI are derived from MoQ transport framing:

```
SBN = floor(group_id / interleave_depth)
ESI = K + object_id
```

where K (source symbols per block) and interleave_depth are known
from the catalog.  Repair symbols have ESI >= K by
convention: object_id 0 on the repair track corresponds to ESI K,
object_id 1 to ESI K+1, and so on up to object_id P-1 (ESI K+P-1).

### 7.1.1. Repair Track Group Structure

Publishers MUST structure repair tracks so that group_id equals
the Source Block Number (SBN).  Object IDs within each repair group
MUST be sequential starting from 0, where object_id R corresponds
to repair symbol ESI K+R.  This convention enables receivers to
derive SBN and ESI from MoQ framing without per-object metadata,
regardless of the source track's container format (LOC, CMAF, etc.).

This convention holds for all container modes.  While CMAF source
objects require explicit FEC Payload ID extensions (Section 8.1)
because CMAF chunks do not map 1:1 to source symbols, repair
objects ARE 1:1 with symbols (each object = one T-byte repair
symbol), so the derivation is valid.

Publishers MUST publish exactly one repair symbol per MoQ object.
The number of repair symbols per block (P) is known from the catalog.

### 7.2. Repair Symbol Generation

For RaptorQ, repair symbols are generated per [RFC6330] Section 5.3.
The Encoding Symbol ID (ESI) for repair symbols starts at K (the
number of source symbols).

For Reed-Solomon, repair symbols are generated per [RFC5510] using
the Vandermonde matrix construction.

For XOR, a single repair symbol is generated as:

```
repair_symbol = source_symbol[0] XOR source_symbol[1] XOR ... XOR source_symbol[K-1]
```

### 7.3. LOC Repair Format

When the repair track's packaging is "loc", each MoQ object on
the repair track is a standard LOC object containing one repair
symbol as its payload.  SBN and ESI are derived from MoQ transport
framing identically to the condensed format (Section 7.1):

```
SBN = floor(group_id / interleave_depth)
ESI = K + object_id
```

LOC repair objects MAY include an Auth Tag extension header
[I-D.ietf-moq-loc] for per-symbol authentication.  No FEC-specific
extension headers are needed.

### 7.4. Condensed Multicast Wire Format

When condensed-packaged content is delivered over IP multicast (SSM/ASM),
each UDP datagram carries one condensed multicast packet.  This format
provides audio/video demuxing (Track ID), FEC metadata (SBN/ESI), and
optional authentication — all catalog-driven with no per-packet flags.

The header layout is inspired by ALC/LCT FEC Payload ID [RFC5510]
but adds per-packet track routing and source/repair discrimination
for multi-track live media, in an 8-byte fixed header.

```
Condensed Multicast Packet {
  Track ID (16),              // 2 bytes — audio/video routing
  Repair (8),                 // 1 byte — 0=source, 1=repair
  Source Block Number (24),   // 3 bytes — SBN (wraps at ~16M blocks)
  Encoding Symbol ID (8),    // 1 byte — ESI (max 255 symbols/block)
  Auth Length (8),            // 1 byte — 0=no auth, else tag size
  Payload (..),               // Source chunk or repair symbol
  [Auth Tag (Auth Length)],   // Variable-length (HMAC, ALTA, etc.)
}
```

**Track ID** (16 bits): Identifies the media track for packet routing (analogous to MMTP packet_id).

**Repair** (8 bits): 0 for source, 1 for repair.  Explicit flag needed because K may vary per block.

**SBN** (24 bits): Source Block Number.  Wraps at ~16M blocks (~25 days at 30fps depth=4).

**ESI** (8 bits): Encoding Symbol ID.  Source: ESI < K.  Repair: ESI >= K.  Max 255 symbols/block.

**Auth Length** (8 bits): Auth tag length in bytes.  0 = no auth.  Max 255, sufficient for HMAC-SHA256 (32), Ed25519 (64), and ALTA variable-length tags.

**Payload**: Source chunk data or repair symbol (Symbol Size bytes).

**Auth Tag** (Auth Length bytes): Scheme-specific tag (HMAC, ALTA, etc.) as declared in catalog.

Fixed overhead: 8 bytes (vs MMTP 33 bytes, LOC ~14+ bytes).

On the MoQ unicast path, condensed packaging uses the headerless
format (Section 7.1) with SBN/ESI derived from MoQ transport framing.
The multicast wire format in this section applies only to UDP delivery.

## 8. MoQ Extension Headers for FEC

This section defines MoQ transport extension headers used by FEC-enabled
streams.  These extensions use the MoQ transport extension header
mechanism defined in [I-D.ietf-moq-transport].  Each extension is
self-describing (type + length).  Presence is determined by the
catalog — no per-object flags byte is needed.

These extensions are REQUIRED on CMAF source track objects where
variable-size chunks prevent SBN/ESI derivation from MoQ coordinates.
They are NOT needed on LOC objects (SBN/ESI derivable from
group_id/object_id) or repair objects (SBN/ESI from framing per
Section 7.1).

### 8.1. FEC Payload ID Extension

```
FEC Payload ID Extension {
  Extension Type (i) = 0xNN,
  Length (i) = 8,
  Source Block Number (32),
  Encoding Symbol ID (32),
}
```

The FEC Payload ID extension carries the combined Source Block Number
(SBN, 32 bits) and Encoding Symbol ID (ESI, 32 bits) for a total of
8 bytes.

### 8.2. Object Length Extension

```
Object Length Extension {
  Extension Type (i) = 0xNN,
  Length (i),
  Original Object Length (i),
}
```

The Object Length extension carries the original object size as a
QUIC variable-length integer (1-8 bytes, per RFC 9000 Section 16).
The encoding is self-describing: the two most significant bits of
the first byte indicate the field width.  The receiver needs the
original object size to strip zero-padding from the last FEC source
symbol.

### 8.3. Auth Tag Extension

```
Auth Tag Extension {
  Extension Type (i) = 0xNN,
  Length (i) = N,
  Auth Tag (N),
}
```

The Auth Tag extension carries an authentication tag whose length
is given by the MoQ extension Length field.  Tag size may be fixed
(HMAC-SHA256: always 32 bytes) or variable (ALTA
[I-D.krose-mboned-alta]: varies per packet depending on MAC chain
depth and signature presence).  The catalog declares the auth
scheme and key distribution mechanism; the per-object Length field
carries the actual tag size.

This extension is OPTIONAL on LOC source and repair track objects.
MMTP tracks use MMTP-native authentication.  Condensed multicast
packets carry auth via the Auth Length field (Section 7.4).

The Auth Tag provides end-to-end content authentication across
untrusted relays — receivers verify that content originated from
the publisher regardless of relay hops (see Section 15).

## 9. Block and Group Alignment

### 9.1. Single-Group Blocks (Interleave Depth = 1)

When Interleave Depth is 1, each FEC block corresponds to exactly
one MoQ Group:

```
Group 0:  [Obj 0] [Obj 1] ... [Obj K-1]  →  FEC Block 0
Group 1:  [Obj 0] [Obj 1] ... [Obj K-1]  →  FEC Block 1
...
```

The Source Block Number (SBN) in repair objects equals the Group ID.

### 9.2. Multi-Group Blocks (Interleave Depth > 1)

When Interleave Depth is D > 1, each FEC block spans D consecutive
Groups:

```
Groups 0-3 (D=4):  →  FEC Block 0, SBN = 0
Groups 4-7 (D=4):  →  FEC Block 1, SBN = 1
...
```

The SBN is computed as: `SBN = floor(Group_ID / Interleave_Depth)`

The first Group ID covered by a block is: `First_Group = SBN * Interleave_Depth`

Repair objects for Block N are published after the last source object
of Group `(N+1) * D - 1`.

## 10. Interleaving

Interleaving spreads source symbols across time to protect against
burst loss.  With interleave depth D, symbols from D consecutive
media units are grouped into one source block:

```
Media Units:    M0   M1   M2   M3   M4   M5   M6   M7
                 |    |    |    |    |    |    |    |
Interleave=4:   [--- Block 0 ---]  [--- Block 1 ---]
                 S0   S1   S2   S3   S4   S5   S6   S7
                                |                    |
                        Block 0 Repair:      Block 1 Repair:
                           R0, R1               R2, R3
```

A burst loss affecting M1-M2 loses symbols S1,S2 from Block 0.  With
P >= 2 repair symbols, Block 0 can still be recovered.

Typical values:

| Application | Interleave Depth | Latency Impact |
|-------------|------------------|----------------|
| Interactive (gaming, WebRTC) | 1-4 | 33-133ms at 30fps |
| Low-latency live | 4-8 | 133-267ms at 30fps |
| Broadcast | 30 | ~1 second at 30fps |
| ATSC 3.0 default | 60 | ~2 seconds at 30fps |

## 11. Priority and Congestion

Repair tracks SHOULD use lower priority than source tracks so that
repair data is dropped first under congestion.

Recommended priority assignment:

| Track Type | Priority | Notes |
|------------|----------|-------|
| Control/Signaling | 0-1 | Highest priority |
| Source Media | 2-4 | Protected |
| Repair Symbols | 6-7 | Dropped first |

Under congestion, this separation ensures source media is preserved
while repair overhead is gracefully shed.

## 12. Hybrid Unicast and Multicast Delivery

This FEC mechanism is designed to support hybrid delivery architectures
where the same media stream is delivered via multiple transport paths
simultaneously.  Multicast delivery paths (Section 3), TreeDN
integration (Section 5), and transport hierarchy (Section 6) are
defined in [I-D.ramadan-moq-multicast]; this section covers
FEC-specific considerations for hybrid delivery.

### 12.1. Multicast and AMT Delivery

IP multicast (including Source-Specific Multicast per [RFC4607]) and
Automatic Multicast Tunneling (AMT per [RFC7450]) provide efficient
one-to-many delivery but are inherently unreliable.  FEC enables receivers
to recover from packet loss without retransmission:

- **Native SSM**: Receivers join multicast group directly
- **AMT Tunneling**: Receivers without native multicast connect via AMT
  relays discovered through DRIAD ([RFC8777])
- **TreeDN Distribution**: Hierarchical multicast trees per [RFC9706]
  with FEC protection at each hop.  ISP edge routers with inline AMT
  relay capability [JUNIPER-TREEDN] can serve as TreeDN nodes,
  replacing x86 CDN servers with in-router replication while
  preserving FEC repair across the delivery path

The same MoQ objects (source and repair) can be transmitted over both
unicast (QUIC/WebTransport) and multicast (UDP/SSM/AMT) paths.  Receivers
on reliable unicast paths MAY skip FEC decoding, while receivers on lossy
multicast paths use FEC for recovery.

### 12.2. Multi-Path FEC Delivery

When source and repair data are delivered via multiple transport
paths simultaneously (e.g., MoQ/QUIC unicast and SSM multicast,
or MoQ/QUIC and ATSC 3.0 RF broadcast), receivers combine FEC
symbols from all available paths.

Key multi-path considerations:

1. **Format consistency**: All paths delivering repair symbols for the same source track SHOULD use the same repair container format.
2. **Deduplication**: Receivers MUST deduplicate symbols received via multiple paths using SBN+ESI as the unique key.
3. **Per-packet OTI**: Formats carrying per-packet OTI (e.g., MMTP, Appendix C) allow mid-stream join without waiting for catalog delivery.
4. **Reliable + unreliable combination**: Source on reliable MoQ/QUIC and repair on lossy multicast is a valid configuration; receivers skip FEC when the reliable path delivers all source symbols.

## 13. ATSC 3.0 Compatibility

This specification is designed for interoperability with ATSC A/331
[ATSC-A331] Application-Layer FEC.

| ATSC A/331 Concept | MoQ FEC Equivalent |
|--------------------|--------------------|
| `<FECParameters>` | Catalog `fec` field (Section 5.1) |
| `<RepairFlow>` | Repair track subscription |
| `fecOTI` (in S-TSID) | OTI derived from catalog fec fields |
| Source TOI range | Group ID range per Section 9 |
| `maximumDelay` | Derived from Interleave Depth x frame duration |
| `overhead` | Computed as (P / K) x 100 |

Publishers ingesting ATSC 3.0 broadcasts SHOULD preserve the original
FEC parameters and pass them through in the catalog fec fields.

For ATSC 3.0 MMT mode ingest, the repair track uses "mmtp"
packaging — repair packets are passed through as-is.  See
Appendix C for the MMTP repair format and multi-path deployment
scenarios.

## 14. Interaction with CMAF Packaging

This specification is designed to work alongside CMAF packaging for
MoQ [I-D.ietf-moq-cmsf].  The relationship is:

FEC encoding is applied to CMAF chunk payloads (moof+mdat), treating
each chunk as source symbols.  The source track uses CMAF packaging;
the repair track uses condensed packaging within the same namespace.

### 14.1. CARP Integration

The CMAF-Aware Real-time Protocol (CARP) [I-D.law-moq-carp] uses
the same FEC catalog fields (Section 5.1) and multicast endpoint
format ([I-D.ramadan-moq-multicast] Section 7) as any other MoQ
packaging format.  No CARP-specific FEC signaling is needed — CARP
catalogs include the `fec` and `multicast` objects defined in this
document and [I-D.ramadan-moq-multicast] respectively.

CMAF source objects with FEC enabled MUST carry the FEC Payload ID
and Object Length extension headers (Section 8.1, 8.2).  CARP
receivers use these extensions identically to non-CARP CMAF
receivers.

## 15. Security Considerations

### 15.1. FEC Does Not Provide Confidentiality

FEC repair symbols are derived from source data.  If source data is
encrypted, repair symbols will also be encrypted by virtue of
operating on ciphertext.  However, FEC itself provides no
confidentiality guarantees.

### 15.2. Repair Symbol Manipulation

An attacker who can modify repair symbols could cause receivers to
reconstruct incorrect source data.  Integrity protection of repair
tracks is RECOMMENDED, using the same mechanisms as source tracks
(e.g., QUIC packet protection).

### 15.3. Amplification

FEC repair symbols represent additional bandwidth.  Publishers MUST
NOT generate excessive repair symbols that could be used for bandwidth
amplification attacks.  The repair overhead (P/K) SHOULD be limited
to reasonable values (e.g., <= 50%).

### 15.4. Multicast FEC Security

When FEC is used over multicast (SSM/AMT), additional considerations
apply:

1. **No QUIC Protection**: Multicast UDP lacks QUIC's integrity
   guarantees.  Publishers SHOULD use DTLS-SRTP or equivalent for
   payload encryption when confidentiality is required.

2. **Repair Injection**: Attackers could inject malicious repair
   symbols on multicast networks.  Receivers SHOULD verify FEC block
   consistency by checking that decoded data matches expected patterns
   (e.g., valid media container structure).

3. **Source Authentication**: For SSM, receivers verify source address
   matches (S,G) subscription.  For AMT, trust is delegated to the
   AMT relay's authentication mechanisms.

4. **Block Integrity**: Receivers MAY implement block-level checksums
   to detect corrupted recovery.  If decoded source data fails
   integrity checks, receivers SHOULD request retransmission via
   unicast fallback.

## 16. IANA Considerations

### 16.1. FEC Algorithm Registry

This document requests creation of a "MoQ FEC Algorithms" registry
with the following initial values:

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | This document |
| 0x01  | XOR       | This document |
| 0x02  | RaptorQ   | [RFC6330] |
| 0x03  | Reed-Solomon | [RFC5510] |
| 0x04-0xFF | Unassigned | |

New registrations require Specification Required policy.

### 16.2. MoQ Streaming Format Packaging

This document requests registration of a MoQ Streaming Format
packaging value in the registry established by
[I-D.ietf-moq-catalogformat]:

| Packaging | Description | Reference |
|-----------|-------------|-----------|
| "condensed" | Condensed FEC repair format (Section 7.1) | This document |

The "condensed" packaging is used for CMAF repair tracks where the
source packaging (cmaf) does not have native FEC framing.  Each MoQ
object contains one raw repair symbol with SBN/ESI derived from MoQ
transport framing (Section 7.1).

Note: The "mmtp" packaging value is registered by
[I-D.ramadan-moq-mmt].  The "loc" packaging value is registered by
[I-D.ietf-moq-loc].  Repair tracks using those packagings follow
the respective format's native FEC framing.

### 16.3. MoQ Object Extension Header Types

This document requests registration of the following MoQ object
extension header types in the "MoQ Extension Header Types" registry
(note: these are object-level extensions per [I-D.ietf-moq-transport]):

| Type | Name | Reference |
|------|------|-----------|
| 0xNN | FEC Payload ID | This document, Section 8.1 |
| 0xNN | Object Length | This document, Section 8.2 |
| 0xNN | Auth Tag | This document, Section 8.3 |

Note: The values 0xNN are placeholders.  Actual values are to be
assigned in coordination with the MOQ WG.  Implementations SHOULD
use values in the experimental range until IANA allocation is
confirmed.

## 17. References

### 17.1. Normative References

[RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
           Requirement Levels", BCP 14, RFC 2119,
           DOI 10.17487/RFC2119, March 1997.

[RFC5510]  Lacan, J., Roca, V., Peltotalo, J., and S. Peltotalo,
           "Reed-Solomon Forward Error Correction (FEC) Schemes",
           RFC 5510, DOI 10.17487/RFC5510, April 2009.

[RFC6330]  Luby, M., Shokrollahi, A., Watson, M., Stockhammer, T.,
           and L. Minder, "RaptorQ Forward Error Correction Scheme
           for Object Delivery", RFC 6330, DOI 10.17487/RFC6330,
           August 2011.

[RFC8174]  Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC
           2119 Key Words", BCP 14, RFC 8174, DOI 10.17487/RFC8174,
           May 2017.

[RFC9000]  Iyengar, J., Ed. and M. Thomson, Ed., "QUIC: A UDP-Based
           Multiplexed and Secure Transport", RFC 9000,
           DOI 10.17487/RFC9000, May 2021.

[I-D.ietf-moq-transport]
           Curley, L., Pugin, K., Nandakumar, S., Vasiliev, V., and
           I. Swett, "Media over QUIC Transport",
           draft-ietf-moq-transport (work in progress).

### 17.2. Informative References

[I-D.ietf-moq-loc]
           Zanaty, M., et al., "Low Overhead Media Container",
           draft-ietf-moq-loc (work in progress).

[I-D.ietf-moq-catalogformat]
           Nandakumar, S., et al., "Common Catalog Format for
           MoQ", draft-ietf-moq-catalogformat (work in progress).

[I-D.ietf-moq-cmsf]
           Law, W., "CMSF: A CMAF Compliant Implementation of
           MOQT Streaming Format", draft-ietf-moq-cmsf
           (work in progress).

[I-D.ramadan-moq-mmt]
           Ramadan, O., "MPEG Media Transport (MMT) Packaging for
           Media over QUIC", draft-ramadan-moq-mmt (work in progress).

[I-D.ramadan-moq-multicast]
           Ramadan, O., "Multicast Delivery and Endpoint Discovery
           for Media over QUIC", draft-ramadan-moq-multicast
           (work in progress).

[RFC4607]  Holbrook, H. and B. Cain, "Source-Specific Multicast for IP",
           RFC 4607, DOI 10.17487/RFC4607, August 2006.

[RFC7450]  Bumgardner, G., "Automatic Multicast Tunneling", RFC 7450,
           DOI 10.17487/RFC7450, February 2015.

[RFC8777]  Holland, J., "DNS Reverse IP Automatic Multicast Tunneling
           (AMT) Discovery", RFC 8777, DOI 10.17487/RFC8777, April 2020.

[RFC9706]  Holland, J., et al., "TreeDN: Tree-Based Content Delivery
           Network (CDN) for Live Streaming to Mass Audiences",
           RFC 9706, December 2024.

[ATSC-A331]
           ATSC, "Signaling, Delivery, Synchronization, and Error
           Protection", A/331:2025, February 2025.

[ARIB-B60]
           ARIB, "MMT-Based Media Transport Scheme in Digital
           Broadcasting Systems", ARIB STD-B60, Version 1.14.

[ISO.23008-1]
           ISO, "Information technology - High efficiency coding and
           media delivery in heterogeneous environments - Part 1:
           MPEG media transport (MMT)", ISO/IEC 23008-1:2023.

[JUNIPER-TREEDN]
           Giuliano, L., "TreeDN - The Fix for Catastrophically
           Successful Live Streaming Events", Juniper TechPost,
           August 2025, https://community.juniper.net/blogs/lenny/
           2025/08/21/introduction-to-TreeDN

## Appendix A. Example Message Flows

### A.1. Catalog-Driven FEC

See Section 3 for the subscribe/publish message flow.  The subscriber
fetches the catalog, discovers fec fields, subscribes to both source
and repair tracks, then receives source objects (group=0, obj=0..K-1)
followed by repair objects (group=0, obj=0..P-1).

### A.2. Recovery Example

Subscriber receives source objects 0,1,3,4,5,6,7,8,9 (object 2 lost)
and repair objects 0,1,2.

With K=10, P=3, subscriber has:
- 9 source symbols (ESI 0,1,3,4,5,6,7,8,9)
- 3 repair symbols (ESI 10,11,12)
- Total: 12 symbols >= K=10

RaptorQ decoder can recover the missing source symbol (ESI 2).

### A.3. Per-Object Overhead Comparison

The following table compares per-object FEC signaling overhead across
container formats.  "MoQ Extensions (CMAF+FEC)" uses the extension
headers defined in Section 8.

| Format | Per-Object Overhead | Components |
|--------|---------------------|------------|
| Condensed (0x00), LOC source | 0 bytes (FEC only) | SBN/ESI derived from MoQ coordinates |
| Condensed (0x00), LOC source + auth | N bytes | Auth Tag extension only (N from catalog) |
| Condensed (0x00), repair | 0 bytes (no auth) | Raw symbol payload, SBN/ESI derived (Section 7.1.1) |
| Condensed (0x00), repair + auth | N bytes | Raw symbol payload + Auth Tag extension |
| MoQ Extensions (CMAF+FEC) | 9-12 bytes | FEC Payload ID (8) + Object Length (1-4 varint) |
| MoQ Extensions (CMAF+FEC+auth) | 9+N - 12+N bytes | FEC PID + Object Length + Auth Tag |
| MMTP (0x01) | 33 bytes | MMTP Header (12) + FEC Payload ID (9) + OTI (12) |

LOC and condensed formats derive SBN/ESI from MoQ coordinates (zero
FEC overhead).  CMAF adds 9-12 bytes/object for explicit SBN/ESI and
object length.  MMTP carries full per-packet OTI (33 bytes) for
self-describing operation and ATSC 3.0 compatibility.

## Appendix B. Catalog Example

Complete catalog with FEC, multicast, and multiple packaging types.
Note: repair track packaging matches source packaging (mmtp/loc),
except CMAF sources whose repair tracks use condensed packaging.

```json
{
  "version": 1,
  "namespace": "live/broadcast",
  "tracks": [
    {
      "name": "video",
      "packaging": "mmtp",
      "codec": "avc1.64001f",
      "width": 1920, "height": 1080, "framerate": 30, "bitrate": 5000000,
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32, "repairSymbols": 8,
        "symbolSize": 1312, "interleaveDepth": 4,
        "repairTrack": "video/repair"
      }
    },
    { "name": "video/repair", "packaging": "mmtp", "priority": 7 },
    {
      "name": "video.cmaf",
      "packaging": "cmaf",
      "codec": "avc1.64001f",
      "width": 1920, "height": 1080,
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32, "repairSymbols": 8,
        "symbolSize": 1312, "interleaveDepth": 4,
        "repairTrack": "video.cmaf/repair"
      }
    },
    { "name": "video.cmaf/repair", "packaging": "condensed", "priority": 7 },
    {
      "name": "audio",
      "packaging": "mmtp",
      "codec": "mp4a.40.2",
      "sampleRate": 48000, "channelCount": 2, "bitrate": 128000,
      "fec": {
        "algorithm": "xor",
        "sourceSymbols": 10, "repairSymbols": 1,
        "symbolSize": 512,
        "repairTrack": "audio/repair"
      }
    },
    { "name": "audio/repair", "packaging": "mmtp", "priority": 7 }
  ],
  "multicast": {
    "groupAddress": "232.1.1.50",
    "port": 5000,
    "sourceAddress": "192.168.1.100"
  }
}
```

## Appendix C. MMTP Repair Packaging (Informational)

NOTE: This appendix is informational.  It creates no normative
dependency on [ISO.23008-1].  Implementations that do not interwork
with MMTP sources need not implement this packaging type.

### C.1. Motivation

MMTP repair packaging enables FEC interoperability between MoQ CDN
delivery and MMTP-based broadcast systems (SSM/ASM multicast, ATSC 3.0
MMT mode).  Repair payloads are unmodified MMTP packets per
[ISO.23008-1], so receivers apply identical recovery regardless of path.

### C.2. Wire Format

When the repair track's packaging is "mmtp", each MoQ repair object
payload contains one complete MMTP repair packet:

```
MoQ Object Payload (packaging: "mmtp") {
  MMTP Header (96),              // 12 bytes, FEC Type=2
  FEC Payload ID {
    Subsequence Start (24),      // SS_Start
    Repair Symbol Block Len (24),// RSB_length
    Repair Symbol ID (24),       // RS_ID
  },
  Object Transmission Info (96), // 12 bytes, RFC 6330 OTI
  Repair Symbol Data (..),       // Symbol Size bytes
}
```

### C.3. Receiver Processing

The receiver extracts FEC parameters by reading at fixed byte
offsets.  This requires one branch on the repair container type
(determined at subscription time from the catalog), then fixed-offset
integer reads and a memcpy of symbol data:

```
offset 0:   MMTP Header (12 bytes) — skip
offset 12:  SS_Start (3 bytes, big-endian) → Source Block Number
offset 15:  RSB_length (3 bytes) — skip
offset 18:  RS_ID (3 bytes) → Encoding Symbol ID derivation
offset 21:  OTI (12 bytes) → RFC 6330 Common + Scheme-Specific OTI
  offset 21: Transfer Length F (5 bytes, 40-bit big-endian)
  offset 26: Reserved (1 byte)
  offset 27: Symbol Size T (2 bytes, big-endian)
  offset 29: Num Source Blocks Z (1 byte)
  offset 30: Num Sub-Blocks N (2 bytes)
  offset 32: Alignment Al (1 byte)
offset 33:  Repair Symbol Data (T bytes) → memcpy to decoder
```

K (source symbols per block) is derived: K = ceil(F / T).

### C.4. Per-Packet OTI

OTI is carried in every MMTP repair packet at offset 21 (12 bytes),
enabling mid-stream join without catalog and supporting mixed-K
streams on a shared repair track.

### C.5. Relay Behavior

Relays SHOULD pass MMTP repair packets through unmodified — no header
stripping, re-framing, or re-encoding.

### C.6. Relationship to Catalog

Catalog fec fields are OPTIONAL when packaging is "mmtp" (OTI is
per-packet).  If present, they SHOULD match per-packet OTI values
and are useful for repair track discovery before the first packet.

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
