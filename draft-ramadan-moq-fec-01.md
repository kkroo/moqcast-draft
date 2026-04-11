# Forward Error Correction for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: January 2027                                       July 2026

        Forward Error Correction for Media over QUIC (MoQ)
                      draft-ramadan-moq-fec-01

Abstract

   This document specifies a mechanism for transmitting Forward Error
   Correction (FEC) repair data alongside source media in Media over
   QUIC (MoQ) sessions.  It defines signaling for FEC configuration
   via both control messages and catalog extensions, conventions for
   repair track naming, and the format of repair objects.  The mechanism
   supports RaptorQ (RFC 6330) and Reed-Solomon (RFC 5510),
   enabling receivers to recover from packet loss without
   retransmission latency.  This specification is designed for
   compatibility with ATSC 3.0 (A/331) and ARIB STD-B60 broadcast systems.

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
       4.1.  FEC_CONFIG Message
       4.2.  FEC_CONFIG Extension Header (Alternative)
       4.3.  FEC Algorithms
       4.4.  RaptorQ Object Transmission Information
       4.5.  Reed-Solomon Parameters
   5.  Catalog FEC Extension
       5.1.  Catalog Fields
       5.2.  Relationship to FEC_CONFIG
   6.  Repair Track Convention
       6.1.  Track Naming
       6.2.  Subscription Model
   7.  Repair Object Format
       7.1.  Repair Object Header
       7.2.  Repair Symbols
   8.  Block and Group Alignment
       8.1.  Single-Group Blocks
       8.2.  Multi-Group Blocks
   9.  Interleaving
   10. Priority and Congestion
   11. Hybrid Unicast and Multicast Delivery
       11.1. Multicast and AMT Delivery
       11.2. Relay-Generated Repair
   12. ATSC 3.0 Compatibility
       12.1. Mapping to A/331 AL-FEC
       12.2. S-TSID to FEC_CONFIG Conversion
   13. Interaction with CMAF Packaging
   14. Security Considerations
       14.1. FEC Does Not Provide Confidentiality
       14.2. Repair Symbol Manipulation
       14.3. Amplification
       14.4. Multicast FEC Security
   15. IANA Considerations
   16. References
       16.1. Normative References
       16.2. Informative References
   Appendix A.  Example Message Flows
   Appendix B.  Catalog Example
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

1. A FEC_CONFIG message for signaling FEC parameters per subscription
2. An alternative FEC_CONFIG extension header for alignment with
   MoQ transport extension mechanisms
3. A catalog extension for out-of-band FEC signaling
4. A naming convention for repair tracks
5. The format of repair objects containing FEC symbols
6. Interleaving strategies to protect against burst loss
7. Compatibility mappings for ATSC 3.0 and ARIB STD-B60 broadcast systems

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
    |  SUBSCRIBE (source track)             |
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK                         |
    |-------------------------------------->|
    |                                       |
    |  FEC_CONFIG (optional)                |
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

1. Subscriber sends SUBSCRIBE for source track
2. Publisher responds with SUBSCRIBE_OK
3. Publisher MAY send FEC_CONFIG to indicate FEC availability
4. Subscriber sends SUBSCRIBE for repair track if FEC desired
5. Publisher sends source objects followed by repair objects
6. Subscriber uses repair symbols to recover any lost source symbols

Alternatively, FEC parameters MAY be discovered via catalog (Section 5)
before subscribing.

## 4. FEC Configuration

### 4.1. FEC_CONFIG Message

The FEC_CONFIG message is sent by the publisher after SUBSCRIBE_OK to
advertise FEC parameters.  Subscribers use this information to decide
whether to subscribe to the repair track.

```
FEC_CONFIG Message {
  Message Type (i) = 0x50,
  Subscribe ID (i),
  FEC Enabled (1),
  FEC Algorithm (8),
  Source Symbols Per Block (i),
  Repair Symbols Per Block (i),
  Symbol Size (i),
  Interleave Depth (i),
  OTI Length (i),
  Object Transmission Information (..),
}
```

**Subscribe ID**: The subscription this FEC configuration applies to.

**FEC Enabled**: 1 if FEC is available for this track, 0 otherwise.

**FEC Algorithm**: The FEC algorithm identifier (see Section 4.3).

**Source Symbols Per Block (K)**: Number of source symbols per FEC
block.  MUST be >= 1.

**Repair Symbols Per Block (P)**: Number of repair symbols generated
per block.  MUST be >= 1.

**Symbol Size**: Size of each symbol in bytes.  All symbols in a
block MUST have the same size.  For RaptorQ, MUST be a multiple of
the Symbol Alignment parameter (typically 8 bytes per RFC 6330).

**Interleave Depth**: Number of media units (frames/chunks) spanned by
one source block.  Higher values protect against longer burst losses
but increase latency.

**OTI Length**: Length of the OTI field in bytes.  0 if not applicable.

**Object Transmission Information**: Algorithm-specific parameters.
For RaptorQ, this is the concatenation of the 8-byte Common FEC OTI
and the 4-byte Scheme-Specific FEC OTI (12 bytes total) as defined
in [RFC6330] Sections 3.3.2 and 3.3.3.

### 4.2. FEC_CONFIG Extension Header (Provisional)

NOTE: This section is provisional and depends on a future extension
to [I-D.ietf-moq-transport] that adds extension header support on
control messages (e.g., SUBSCRIBE_OK).  As of moq-transport-15,
extension headers are defined only for Object payloads, not control
messages.  Implementations MUST use the FEC_CONFIG message
(Section 4.1) as the normative signaling mechanism until such an
extension is adopted.

For implementations where MoQ transport supports control message
extensions, FEC parameters MAY alternatively be conveyed as an
extension header in SUBSCRIBE_OK:

```
FEC_CONFIG Extension Header {
  Extension Type (i) = 0x50,
  Length (i),
  FEC Enabled (1),
  FEC Algorithm (8),
  Source Symbols Per Block (i),
  Repair Symbols Per Block (i),
  Symbol Size (i),
  Interleave Depth (i),
  OTI Length (i),
  Object Transmission Information (..),
}
```

Relays MUST cache and forward this extension header without
modification.

The FEC_CONFIG message (Section 4.1) is the normative mechanism.
The extension header is informational and provided for convenience.
Publishers MUST NOT send both simultaneously for the same
subscription.

### 4.3. FEC Algorithms

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | This document |
| 0x01  | RaptorQ   | [RFC6330] |
| 0x02  | Reed-Solomon (GF2^8) | [RFC5510] |
| 0x03-0xFF | Reserved | IANA |

**RaptorQ (0x01)**: RaptorQ fountain code per [RFC6330].  Can recover
from loss of any symbols as long as K symbols (source or repair) are
received.  Recommended for most applications.

**Reed-Solomon (0x02)**: Reed-Solomon erasure code over GF(2^8) per
[RFC5510].  Can recover up to P lost symbols.  Suitable for
applications requiring exact recovery guarantees or interoperability
with ATSC 3.0 systems using RS FEC.

### 4.4. RaptorQ Object Transmission Information

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

### 4.5. Reed-Solomon Parameters

When FEC Algorithm is Reed-Solomon (0x03), the OTI field contains:

```
Reed-Solomon OTI {
  Field Size (8),          // Always 8 for GF(2^8)
  Max Source Symbols (16), // Maximum K value
  Max Repair Symbols (16), // Maximum P value
  Reserved (24),
}
```

## 5. Catalog FEC Extension

### 5.1. Catalog Fields

MoQ catalogs [I-D.ietf-moq-loc] MAY include FEC configuration at the
track level:

```json
{
  "tracks": [{
    "name": "video",
    "packaging": "cmaf",
    "fec": {
      "algorithm": "raptorq",
      "sourceSymbols": 32,
      "repairSymbols": 8,
      "symbolSize": 1312,
      "interleaveDepth": 4000,
      "repairTrack": "video/repair"
    }
  }]
}
```

Field definitions:

**algorithm** (string, REQUIRED if fec present): FEC scheme identifier.
  - "none": No FEC (passthrough, optional jitter buffer)
  - "raptorq": RaptorQ per RFC 6330
  - "reed-solomon": Reed-Solomon GF(2^8) per RFC 5510

**sourceSymbols** (integer, REQUIRED): K value - source symbols per
FEC block.

**repairSymbols** (integer, REQUIRED): P value - repair symbols per
FEC block.

**symbolSize** (integer, REQUIRED): T value - bytes per symbol.

**interleaveDepth** (integer, OPTIONAL): FEC block span in
milliseconds.  The encoder computes the number of groups per block
as D = ceil(interleaveDepth / GOP_duration_ms).  Using milliseconds
rather than frame/group counts decouples FEC from frame rate —
30fps video and 46.875fps audio can share the same interleaveDepth
value.  Default is 0 (single-group blocks, D=1).

When CMAF packaging is used, the CMAF segment duration SHOULD equal
interleaveDepth so that each segment contains exactly one FEC
block's worth of source symbols, enabling CDN-side FEC repair
before forwarding to FEC-unaware HLS/DASH clients.

**repairTrack** (string, REQUIRED): Track name for repair symbols,
following the convention in Section 6.1.

### 5.2. Relationship to FEC_CONFIG

The FEC_CONFIG message (Section 4.1) is the normative signaling
mechanism.  The catalog FEC extension (Section 5.1) provides
out-of-band discovery before subscription.

Precedence rules when multiple signaling sources are present:

1. **FEC_CONFIG message** is authoritative.  If present, it defines
   the FEC parameters for the subscription.
2. **Catalog FEC fields** are informational for discovery purposes.
   Subscribers use catalog data to decide whether to subscribe to
   repair tracks before receiving FEC_CONFIG.
3. If FEC_CONFIG parameters differ from catalog values, subscribers
   MUST use FEC_CONFIG parameters for decoding.
4. Subscribers SHOULD log a warning if FEC_CONFIG differs from
   catalog, as this may indicate a configuration error.
5. The provisional extension header (Section 4.2) is informational
   only and MUST NOT be used as the sole FEC signaling mechanism.

For multicast delivery where FEC_CONFIG cannot be sent (no back
channel), catalog-based signaling is REQUIRED.

Relay behavior: Relays MUST forward FEC_CONFIG messages unmodified
to downstream subscribers.  Relays MUST NOT strip FEC_CONFIG in
favor of catalog or extension header signaling.

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

1. The FEC_CONFIG message indicates FEC is available, AND
2. The subscriber desires FEC protection

Publishers MUST NOT require subscription to repair tracks.  Repair
tracks are optional and subscribers MAY choose to rely solely on
QUIC retransmission.

Subscribers MAY subscribe to repair tracks without first receiving
FEC_CONFIG if they have out-of-band knowledge of FEC availability
(e.g., from catalog).

## 7. Repair Object Format

### 7.1. Repair Object Header

Each object on a repair track contains repair symbols for one source
block.  The object payload begins with a header:

```
Repair Object {
  Source Block Number (32),
  First Repair Symbol ESI (32),
  Num Repair Symbols (i),
  Repair Symbols (..),
}
```

**Source Block Number (SBN)**: Identifies which source block these
repair symbols protect.  See Section 8 for alignment with Group IDs.
Fixed 32-bit width is used for alignment with RFC 6330 FEC Payload ID
encoding, which uses 32-bit fields for SBN and ESI.

**First Repair Symbol ESI**: Encoding Symbol ID of the first repair
symbol in this object.  For RaptorQ, repair symbols have ESI >= K.
Fixed 32-bit width matches RFC 6330 FEC Payload ID encoding.

**Num Repair Symbols**: Number of repair symbols in this object.
Variable-length integer encoding is used here because this is a
MoQ-specific field (not constrained by RFC 6330) and typically
has small values.

**Repair Symbols**: Concatenated repair symbol data.  Each symbol is
Symbol Size bytes.

### 7.2. Repair Symbols

For RaptorQ, repair symbols are generated per [RFC6330] Section 5.3.
The Encoding Symbol ID (ESI) for repair symbols starts at K (the
number of source symbols).

For Reed-Solomon, repair symbols are generated per [RFC5510] using
the Vandermonde matrix construction.

## 8. Block and Group Alignment

### 8.1. Single-Group Blocks (Interleave Depth = 1)

When Interleave Depth is 1, each FEC block corresponds to exactly
one MoQ Group:

```
Group 0:  [Obj 0] [Obj 1] ... [Obj K-1]  →  FEC Block 0
Group 1:  [Obj 0] [Obj 1] ... [Obj K-1]  →  FEC Block 1
...
```

The Source Block Number (SBN) in repair objects equals the Group ID.

### 8.2. Multi-Group Blocks (Interleave Depth > 1)

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

### 8.3. Source Symbol ESI Derivation

Receivers MUST derive the Encoding Symbol ID (ESI) for each source
object from MoQ transport identifiers.  The ESI identifies the
symbol's position within its FEC block for RaptorQ or Reed-Solomon
decoding.

For a source object with MoQ Group_ID `G` and Object_ID `O`:

```
D = ceil(interleaveDepth_ms / GOP_duration_ms)   # groups per block
SBN = floor(G / D)                                # source block number
first_group = SBN * D                             # first group in block
symbols_per_group = ceil(K / D)                   # symbols from each group
ESI = (G - first_group) * symbols_per_group + O   # encoding symbol ID
```

The flat SS_ID (per ISO 23008-1 Section C.5.2) is reconstructed as:

```
SS_ID = SBN * K + ESI
```

This enables multi-path FEC combining: a receiver that obtains the
same MMTP packet via MoQ unicast (using Group_ID/Object_ID) and
multicast UDP (using SS_ID from the FEC Payload ID) derives
identical (SBN, ESI) coordinates, allowing symbols from any
transport path to contribute to the same FEC block recovery.

For MMTP over multicast UDP where the Source FEC Payload ID carries
SS_ID directly:

```
SBN = floor(SS_ID / K)
ESI = SS_ID % K
```

Both derivations produce the same (SBN, ESI) for the same packet.

### 8.4. Encoder Constraints

For FEC block alignment to be deterministic, encoders MUST:

1. Use a fixed keyframe interval (constant GOP duration) within
   each FEC block.  Variable GOP sizes (scene-cut adaptive
   keyframes) break the symbols_per_group calculation.

2. Signal the actual GOP duration and frame rate in both:
   - ISO 23008-1 AL-FEC signaling (for broadcast receivers)
   - MoQ catalog FEC_CONFIG (for MoQ receivers)

3. If frame rate or GOP size changes mid-session, update the MoQ
   catalog and signal a manifest discontinuity for DASH/HLS.

When CMAF packaging is used, the CMAF segment duration MUST equal
interleaveDepth_ms so that each segment boundary aligns with a FEC
block boundary.  This enables CDN relays to perform FEC recovery at
the segment level before forwarding to FEC-unaware clients.

## 9. Interleaving

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

Publishers SHOULD choose interleave depth based on:

- Expected burst loss duration
- Acceptable recovery latency
- Available bandwidth for repair overhead

Typical values:

| Application | interleaveDepth (ms) | Groups at 30fps/1s GOP | Recovery Latency |
|-------------|---------------------|------------------------|------------------|
| Interactive (gaming, WebRTC) | 33-133 | 1 | 33-133ms |
| Low-latency live | 1000-2000 | 1-2 | 1-2s |
| Broadcast | 4000 | 4 | 4s |
| ATSC 3.0 default | 8000 | 8 | 8s |

## 10. Priority and Congestion

Repair tracks SHOULD use lower priority than source tracks so that
repair data is dropped first under congestion.

Recommended priority assignment:

| Track Type | Priority | Notes |
|------------|----------|-------|
| Control/Signaling | 0-1 | Highest priority |
| Source Media | 2-4 | Protected |
| Repair Symbols | 6-7 | Dropped first |

Publishers set priority via the QUIC stream priority mechanism or
MoQ-specific priority signaling.

Under congestion, this priority separation ensures:

1. Source media is preserved as long as possible
2. Repair overhead is gracefully shed
3. Receivers degrade to QUIC retransmission when FEC is unavailable

## 11. Hybrid Unicast and Multicast Delivery

This FEC mechanism is designed to support hybrid delivery architectures
where the same media stream is delivered via multiple transport paths
simultaneously.  Multicast delivery paths (Section 3), TreeDN
integration (Section 5), and transport hierarchy (Section 6) are
defined in [I-D.ramadan-moq-multicast]; this section covers
FEC-specific considerations for hybrid delivery.

### 11.1. Multicast and AMT Delivery

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

```
Publisher
    |
    v
+----------------+
|   MoQ Relay    |
| (Origin/Root)  |
+-------+--------+
        |
   +----+----+--------------------+
   |         |                    |
   v         v                    v
Unicast   SSM Multicast      AMT Tunnel
(QUIC)    (Native)           (Encapsulated)
   |         |                    |
   v         v                    v
[Web]    [Native SSM         [Mobile/
Client    Clients]            Remote]

FEC:      FEC:                FEC:
Optional  Required            Required
```

### 11.2. Relay-Generated Repair

Relays MAY generate additional repair symbols for local network conditions.
This is enabled by the fountain code property of RaptorQ, which allows
generation of arbitrary repair symbols from decoded source data.

Use cases for relay-generated repair include:

- Edge relays serving lossy last-mile networks (WiFi, cellular) can increase
  repair overhead beyond what the publisher provides
- Regional relays can adapt FEC parameters to local loss patterns
- Relays can regenerate repair if upstream repair symbols were lost
- IWA home gateways receiving multicast over wired Ethernet can apply
  local FEC repair before redistributing via WebTransport to wireless
  clients, reducing retransmission round trips to the origin

When a relay generates its own repair symbols:

1. The relay MUST successfully decode the source block first
2. Generated repair symbols SHOULD use ESIs that do not conflict with
   publisher-generated repair (e.g., ESI >= K + P_publisher)
3. The relay MAY advertise relay-specific repair via a separate track
   namespace or track path suffix

Security considerations for relay-generated repair:

A compromised relay could inject malicious repair symbols that cause
receivers to reconstruct corrupted source data.  This risk is
elevated on multicast paths where QUIC's integrity guarantees are
absent.  To mitigate:

1. **Block-level integrity**: Publishers SHOULD include a
   per-block HMAC or content hash in the repair object header
   (as an optional trailing field) so receivers can verify that
   decoded source data matches the publisher's original.
2. **Trust model**: Receivers MUST distinguish between
   publisher-generated repair (trusted, from the origin) and
   relay-generated repair (semi-trusted).  If decoded data fails
   integrity verification using relay-generated repair, receivers
   SHOULD discard it and request retransmission via unicast.
3. **Separate tracks**: Relay-generated repair SHOULD be published
   on a distinct track (e.g., "video/repair/relay") so receivers
   can apply different trust policies.

```
Publisher (K=32, P=8, 25% overhead)
    |
    | Source ESI: 0-31, Repair ESI: 32-39
    v
+------------------+
| Backbone Relay   |  (passes through, low loss)
+--------+---------+
         |
    +----+----+
    |         |
    v         v
+--------+ +--------+
| Edge A | | Edge B |
| P=16   | | P=4    |
| (50%)  | | (12%)  |
+--------+ +--------+
    |         |
    v         v
[Lossy    [Clean
 WiFi]     Fiber]
```

## 12. ATSC 3.0 Compatibility

This specification is designed for interoperability with ATSC A/331
[ATSC-A331] Application-Layer FEC.

### 12.1. Mapping to A/331 AL-FEC

| ATSC A/331 Concept | MoQ FEC Equivalent |
|--------------------|--------------------|
| `<FECParameters>` | FEC_CONFIG message or catalog `fec` field |
| `<RepairFlow>` | Repair track subscription |
| `fecOTI` (in S-TSID) | OTI field in FEC_CONFIG |
| Source TOI range | Group ID range per Section 8 |
| `maximumDelay` | Derived from Interleave Depth × frame duration |
| `overhead` | Computed as (P / K) × 100 |

Publishers ingesting ATSC 3.0 broadcasts SHOULD preserve the original
FEC parameters and pass them through in FEC_CONFIG.

### 12.2. S-TSID to FEC_CONFIG Conversion

When converting ATSC S-TSID FEC parameters to MoQ FEC_CONFIG:

```
S-TSID (per ATSC A/331):
<FECParameters maximumDelay="1000" overhead="25"
               fecOTI="F=32;T=1312;Z=4;N=1;Al=8">
  <ProtectedObject tsi="1">
    <SourceTOI x="0" y="255"/>
  </ProtectedObject>
</FECParameters>

Note: The fecOTI attribute uses ATSC A/331 field names, which
differ from RFC 6330 terminology.  F denotes the number of source
symbols per block (equivalent to K in this document), T is symbol
size, Z is the number of source blocks, N is sub-blocks, and Al
is symbol alignment.  RFC 6330 uses F for Transfer Length (40 bits),
which is a different concept; implementors should use the A/331
schema as the reference for S-TSID parsing.

Converts to FEC_CONFIG:
  FEC Algorithm: 0x02 (RaptorQ)
  Source Symbols Per Block: 32  (from F parameter)
  Repair Symbols Per Block: 8  (25% of 32, from overhead)
  Symbol Size: 1312  (from T parameter)
  Interleave Depth: 4  (from Z parameter or derived from maximumDelay)
  OTI: [12-byte concatenation of Common FEC OTI + Scheme-Specific
        FEC OTI per RFC 6330]
```

## 13. Interaction with CMAF Packaging

This specification is designed to work alongside CMAF packaging for
MoQ [I-D.ietf-moq-cmsf].  The relationship is:

```
┌─────────────────────────────────────────────────┐
│              MoQ Track Namespace                │
├─────────────────────────────────────────────────┤
│                                                 │
│  Source Track (CMAF packaged)                   │
│  ┌─────────────────────────────────────────┐   │
│  │ Object 0: CMAF Header (ftyp, moov)      │   │
│  │ Object 1: CMAF Chunk (moof + mdat)      │   │
│  │ Object 2: CMAF Chunk (moof + mdat)      │   │
│  │ ...                                      │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  Repair Track (this specification)              │
│  ┌─────────────────────────────────────────┐   │
│  │ Object 0: Repair for Block 0            │   │
│  │ Object 1: Repair for Block 1            │   │
│  │ ...                                      │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
└─────────────────────────────────────────────────┘
```

FEC encoding is applied to CMAF chunk payloads, treating each chunk
(or portion thereof) as source symbols.  The FEC layer is agnostic to
the media container format.

## 14. Security Considerations

### 14.1. FEC Does Not Provide Confidentiality

FEC repair symbols are derived from source data.  If source data is
encrypted, repair symbols will also be encrypted by virtue of
operating on ciphertext.  However, FEC itself provides no
confidentiality guarantees.

### 14.2. Repair Symbol Manipulation

An attacker who can modify repair symbols could cause receivers to
reconstruct incorrect source data.  Integrity protection of repair
tracks is RECOMMENDED, using the same mechanisms as source tracks
(e.g., QUIC packet protection).

### 14.3. Amplification

FEC repair symbols represent additional bandwidth.  Publishers MUST
NOT generate excessive repair symbols that could be used for bandwidth
amplification attacks.  The repair overhead (P/K) SHOULD be limited
to reasonable values (e.g., <= 50%).

### 14.4. Multicast FEC Security

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

## 15. IANA Considerations

### 15.1. MoQ Message Type

This document requests registration of a new MoQ message type in the
"MoQ Message Types" registry:

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

Note: The value 0x50 is tentative and subject to coordination with
the MOQ WG chairs to avoid collision with other allocations.
Implementations SHOULD use a value in the experimental range
(0xF000-0xFFFF) until IANA allocation is confirmed.  This
registration is shared with [I-D.ramadan-moq-mmt], which references
this document for the normative FEC_CONFIG definition.

### 15.2. MoQ Extension Header Type

This document requests registration of a new MoQ extension header
type in a separate "MoQ Extension Header Types" registry (note: this
is a different IANA registry than the message type registry in 15.1,
so reuse of the value 0x50 does not create a collision):

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

Note: This registration is provisional pending adoption of control
message extension headers in [I-D.ietf-moq-transport] (see
Section 4.2).

### 15.3. FEC Algorithm Registry

This document requests creation of a "MoQ FEC Algorithms" registry
with the following initial values:

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | This document |
| 0x01  | RaptorQ   | [RFC6330] |
| 0x02  | Reed-Solomon | [RFC5510] |
| 0x03-0xFF | Unassigned | |

New registrations require Specification Required policy.

## 16. References

### 16.1. Normative References

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

[I-D.ietf-moq-transport]
           Curley, L., Pugin, K., Nandakumar, S., Vasiliev, V., and
           I. Swett, "Media over QUIC Transport",
           draft-ietf-moq-transport (work in progress).

### 16.2. Informative References

[I-D.ietf-moq-loc]
           Zanaty, M., et al., "Low Overhead Media Container",
           draft-ietf-moq-loc (work in progress).

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

### A.1. Publisher-Initiated FEC

```
Publisher                              Subscriber
    |                                       |
    |  SUBSCRIBE(sub_id=1, track=video)     |
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK(sub_id=1)               |
    |-------------------------------------->|
    |                                       |
    |  FEC_CONFIG(sub_id=1,                 |
    |    algorithm=RaptorQ,                 |
    |    K=32, P=8, symbol_size=1312,       |
    |    interleave=4, oti=...)             |
    |-------------------------------------->|
    |                                       |
    |  SUBSCRIBE(sub_id=2, track=video/repair)
    |<--------------------------------------|
    |                                       |
    |  SUBSCRIBE_OK(sub_id=2)               |
    |-------------------------------------->|
    |                                       |
    |  [Source objects, group=0, obj=0..31] |
    |-------------------------------------->|
    |                                       |
    |  [Repair objects, group=0, obj=0..7]  |
    |-------------------------------------->|
    |                                       |
```

### A.2. Recovery Example

Subscriber receives source objects 0,1,3,4,5,6,7,8,9 (object 2 lost)
and repair objects 0,1,2.

With K=10, P=3, subscriber has:
- 9 source symbols (ESI 0,1,3,4,5,6,7,8,9)
- 3 repair symbols (ESI 10,11,12)
- Total: 12 symbols >= K=10

RaptorQ decoder can recover the missing source symbol (ESI 2).

## Appendix B. Catalog Example

Complete catalog with FEC and multicast configuration:

```json
{
  "version": 1,
  "namespace": "live/broadcast",
  "tracks": [
    {
      "name": "video",
      "packaging": "cmaf",
      "codec": "avc1.64001f",
      "width": 1920,
      "height": 1080,
      "framerate": 30,
      "bitrate": 5000000,
      "fec": {
        "algorithm": "raptorq",
        "sourceSymbols": 32,
        "repairSymbols": 8,
        "symbolSize": 1312,
        "interleaveDepth": 4000,
        "repairTrack": "video/repair"
      }
    },
    {
      "name": "video/repair",
      "packaging": "fec-repair",
      "priority": 7
    },
    {
      "name": "audio",
      "packaging": "cmaf",
      "codec": "mp4a.40.2",
      "sampleRate": 48000,
      "channelCount": 2,
      "bitrate": 128000,
      "fec": {
        "algorithm": "none",
        "sourceSymbols": 10,
        "repairSymbols": 1,
        "symbolSize": 512,
        "repairTrack": "audio/repair"
      }
    }
  ],
  "multicast": {
    "group": "232.1.1.50",
    "port": 5000,
    "source": "192.168.1.100"
  }
}
```

The `multicast` field uses the simple format defined in
[I-D.ramadan-moq-multicast] Section 7.1.  For multi-endpoint
deployments, the extended format (Section 7.2) with `endpoints`
array is also available.

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
