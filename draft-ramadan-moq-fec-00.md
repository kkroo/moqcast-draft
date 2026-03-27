# Forward Error Correction for Media over QUIC

```
Internet-Draft                                              O. Ramadan
Intended status: Standards Track                             Blockcast
Expires: January 2027                                       July 2026

        Forward Error Correction for Media over QUIC (MoQ)
                      draft-ramadan-moq-fec-00

Abstract

   This document specifies Forward Error Correction (FEC) for Media
   over QUIC (MoQ).  It defines FEC configuration signaling, catalog
   extensions, repair track naming, repair object formats, and
   interleaving strategies.  Supported FEC schemes are RaptorQ
   (RFC 6330), Reed-Solomon (RFC 5510), and XOR.  A repair container
   abstraction allows repair objects to carry native MoQ-framed
   symbols or container-specific payloads (e.g., MMTP), enabling
   multi-path FEC where receivers apply identical recovery regardless
   of transport path (MoQ/QUIC, SSM multicast, or ATSC 3.0 broadcast).

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
       4.2.  FEC_CONFIG Extension Header (Provisional)
       4.3.  FEC Algorithms
       4.4.  RaptorQ Object Transmission Information
       4.5.  Reed-Solomon Parameters
       4.6.  Repair Container Types
   5.  Catalog FEC Extension
       5.1.  Catalog Fields
       5.2.  Relationship to FEC_CONFIG
   6.  Repair Track Convention
       6.1.  Track Naming
       6.2.  Subscription Model
   7.  Repair Object Format
       7.1.  Native MoQ Repair Format
       7.2.  Repair Symbol Generation
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
       12.2. Relay-Generated Repair
       12.3. Multi-Path FEC Delivery
   13. ATSC 3.0 Compatibility
       13.1. Mapping to A/331 AL-FEC
       13.2. S-TSID to FEC_CONFIG Conversion
   14. Interaction with CMAF Packaging
       14.1. CARP Integration
   15. Security Considerations
   16. IANA Considerations
   17. References
       17.1. Normative References
       17.2. Informative References
   Appendix A.  Example Message Flows
   Appendix B.  Catalog Example
   Appendix C.  MMTP Repair Container Mapping (Informational)
   Authors' Addresses
```

## 1. Introduction

Media over QUIC (MoQ) [I-D.ietf-moq-transport] provides
publish/subscribe transport for real-time media.  While QUIC provides
reliable delivery through retransmission, retransmission latency is
unacceptable for ultra-low-latency applications such as live sports
and interactive broadcasts.

Forward Error Correction (FEC) enables receivers to recover lost
packets using redundant repair data without retransmission.

This document defines:

1. A FEC_CONFIG message for signaling FEC parameters per subscription
2. A provisional FEC_CONFIG extension header for SUBSCRIBE_OK
3. A catalog extension for out-of-band FEC discovery
4. A naming convention for repair tracks
5. The format of repair objects containing FEC symbols
6. Interleaving strategies for burst loss protection
7. Compatibility mappings for ATSC 3.0 and ARIB STD-B60
8. Multi-path delivery across MoQ/QUIC, SSM multicast, and ATSC 3.0

The mechanism complements MoQ packaging formats including CMAF
[I-D.ietf-moq-cmsf], LOC [I-D.ietf-moq-loc], and MMT
[I-D.ramadan-moq-mmt] by operating as a separate protection layer.

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

**Source Symbol**: A fixed-size unit of source data for FEC encoding.

**Repair Symbol**: A symbol generated by the FEC encoder for
recovering lost source symbols.

**FEC Block**: The combination of K source symbols and P repair symbols.

**Interleave Depth**: The number of media units (e.g., frames) spanned
by a single source block.

**Object Transmission Information (OTI)**: Parameters required to
configure a RaptorQ decoder, per [RFC6330].

**Source Block Number (SBN)**: Zero-based index identifying a source
block within a session.

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

1. Subscriber sends SUBSCRIBE for source track
2. Publisher responds with SUBSCRIBE_OK
3. Publisher MAY send FEC_CONFIG to indicate FEC availability
4. Subscriber sends SUBSCRIBE for repair track if FEC desired
5. Publisher sends source objects followed by repair objects
6. Subscriber uses repair symbols to recover lost source symbols

FEC parameters MAY alternatively be discovered via catalog (Section 5)
before subscribing.

## 4. FEC Configuration

### 4.1. FEC_CONFIG Message

The FEC_CONFIG message is sent by the publisher after SUBSCRIBE_OK to
advertise FEC parameters.

```
FEC_CONFIG Message {
  Message Type (i) = 0x50,
  Subscribe ID (i),
  FEC Enabled (1),
  FEC Algorithm (8),
  Repair Container (8),
  Source Symbols Per Block (i),
  Repair Symbols Per Block (i),
  Symbol Size (i),
  Interleave Depth (i),
  OTI Length (i),
  Object Transmission Information (..),
}
```

**Subscribe ID**: The subscription this configuration applies to.

**FEC Enabled**: 1 if FEC is available, 0 otherwise.

**FEC Algorithm**: Identifier per Section 4.3.

**Repair Container**: Format of repair payloads per Section 4.6.
Default 0x00 is native MoQ (Section 7); 0x01 is MMTP passthrough
(Appendix C).

**Source Symbols Per Block (K)**: Source symbols per FEC block.
MUST be >= 1.

**Repair Symbols Per Block (P)**: Repair symbols per block.
MUST be >= 1.

**Symbol Size (T)**: Bytes per symbol.  All symbols in a block MUST
have the same size.  For RaptorQ, MUST be a multiple of the Symbol
Alignment parameter (typically 8 bytes per RFC 6330).

**Interleave Depth**: Media units spanned by one source block.
Higher values protect against longer bursts but increase latency.

**OTI Length**: Length of OTI field in bytes.  0 if not applicable.

**Object Transmission Information**: Algorithm-specific parameters.
For RaptorQ, the concatenation of 8-byte Common FEC OTI and 4-byte
Scheme-Specific FEC OTI (12 bytes total) per [RFC6330] Sections
3.3.2 and 3.3.3.

### 4.2. FEC_CONFIG Extension Header (Provisional)

NOTE: This section is provisional and depends on a future
[I-D.ietf-moq-transport] extension adding extension header support on
control messages.  As of moq-transport-15, extension headers are
defined only for object payloads.  Implementations MUST use FEC_CONFIG
(Section 4.1) until such an extension is adopted.

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

Relays MUST cache and forward this extension without modification.
Publishers MUST NOT send both FEC_CONFIG message and extension header
for the same subscription.

### 4.3. FEC Algorithms

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | N/A       |
| 0x01  | XOR       | This document |
| 0x02  | RaptorQ   | [RFC6330] |
| 0x03  | Reed-Solomon (GF2^8) | [RFC5510] |
| 0x04-0xFF | Reserved | IANA |

**XOR (0x01)**: Each repair symbol is the XOR of all K source symbols.
Recovers exactly one lost symbol per block.

**RaptorQ (0x02)**: Fountain code per [RFC6330].  Recovers from any
loss pattern given K received symbols (source or repair).  Recommended
for most applications.

**Reed-Solomon (0x03)**: Erasure code over GF(2^8) per [RFC5510].
Recovers up to P lost symbols.

### 4.4. RaptorQ Object Transmission Information

When FEC Algorithm is RaptorQ (0x02), the OTI field contains the
8-byte Common FEC OTI and 4-byte Scheme-Specific FEC OTI per
[RFC6330], totaling 12 bytes:

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

Transfer Length is computed per source block from the cumulative size
of source objects in the block.

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

### 4.6. Repair Container Types

| Value | Container | Reference |
|-------|-----------|-----------|
| 0x00  | Native MoQ (Section 7) | This document |
| 0x01  | MMTP (Appendix C) | This document |
| 0x02-0xFF | Reserved | IANA |

When 0x00, repair payloads use the native format (Section 7.1).
When 0x01, repair payloads use MMTP format (Appendix C).

Receivers MUST support container 0x00.  Other containers are OPTIONAL.

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
      "interleaveDepth": 4,
      "repairContainer": "mmtp",
      "repairTrack": "video/repair"
    }
  }]
}
```

**algorithm** (string, REQUIRED if fec present): "none", "xor",
"raptorq", or "reed-solomon".

**sourceSymbols** (integer, REQUIRED): K value.

**repairSymbols** (integer, REQUIRED): P value.

**symbolSize** (integer, REQUIRED): T value in bytes.

**interleaveDepth** (integer, OPTIONAL): Media units per FEC block.
Default 1.

**repairContainer** (string, OPTIONAL): "native" (default) or "mmtp".

**repairTrack** (string, REQUIRED): Repair track name per Section 6.1.

### 5.2. Relationship to FEC_CONFIG

FEC_CONFIG (Section 4.1) is authoritative.  Catalog fields are
informational for pre-subscription discovery.

1. FEC_CONFIG defines the subscription's FEC parameters when present
2. Catalog data guides repair track subscription decisions
3. If FEC_CONFIG differs from catalog, subscribers MUST use FEC_CONFIG
4. Subscribers SHOULD log warnings on FEC_CONFIG/catalog mismatch
5. The provisional extension header (Section 4.2) MUST NOT be the
   sole signaling mechanism

For multicast delivery (no back channel), catalog signaling is
REQUIRED.

Relays MUST forward FEC_CONFIG unmodified and MUST NOT strip it in
favor of catalog or extension header signaling.

## 6. Repair Track Convention

### 6.1. Track Naming

For source track [namespace, track_name], the repair track MUST be:

```
[namespace, track_name, "repair"]
```

| Source Track | Repair Track |
|--------------|--------------|
| ["live", "video"] | ["live", "video", "repair"] |
| ["broadcast", "video", "1080p"] | ["broadcast", "video", "1080p", "repair"] |
| ["stream", "audio", "en"] | ["stream", "audio", "en", "repair"] |

### 6.2. Subscription Model

Subscribers SHOULD subscribe to the source track first, then to the
repair track if FEC_CONFIG indicates availability and FEC is desired.

Publishers MUST NOT require repair track subscription.  Subscribers
MAY rely solely on QUIC retransmission.

Subscribers MAY subscribe to repair tracks using out-of-band knowledge
(e.g., catalog) without first receiving FEC_CONFIG.

## 7. Repair Object Format

### 7.1. Native MoQ Repair Format

This format applies when Repair Container is 0x00 (or catalog
"native"/omitted).  Each repair object contains exactly one repair
symbol with no additional header:

```
Repair Object Payload {
  Repair Symbol (Symbol Size bytes),
}
```

SBN and ESI are derived from MoQ framing:

```
SBN = floor(group_id / interleave_depth)
ESI = K + object_id
```

Object_id 0 corresponds to ESI K, object_id P-1 to ESI K+P-1.

#### 7.1.1. Repair Track Group Structure

Publishers MUST set repair track group_id equal to the SBN.  Object
IDs MUST be sequential from 0, where object_id R = repair symbol
ESI K+R.

This convention holds for all container modes.  While CMAF source
objects require explicit FEC Payload ID extensions (Section 8.1)
because CMAF chunks do not map 1:1 to symbols, repair objects ARE
1:1 with symbols, so the derivation is valid.

Publishers MUST publish exactly one repair symbol per MoQ object.
Bundling multiple symbols creates a single point of failure defeating
FEC.  Separate objects allow independent delivery and priority
handling.

### 7.2. Repair Symbol Generation

**RaptorQ**: Per [RFC6330] Section 5.3.  Repair ESI starts at K.

**Reed-Solomon**: Per [RFC5510] using Vandermonde matrix construction.

**XOR**: `repair = source[0] XOR source[1] XOR ... XOR source[K-1]`

## 8. MoQ Extension Headers for FEC

These extensions use the MoQ transport extension header mechanism
[I-D.ietf-moq-transport].  Each is self-describing (type + length).
Presence is determined by catalog; no per-object flags are needed.

### 8.1. FEC Payload ID Extension

```
FEC Payload ID Extension {
  Extension Type (i) = 0xNN,
  Length (i) = 8,
  Source Block Number (32),
  Encoding Symbol ID (32),
}
```

REQUIRED on CMAF source objects when FEC is enabled.  CMAF chunks are
variable-size (VBR), and the object-to-source-block mapping is not
1:1.  The explicit SBN+ESI enables correct FEC association.

NOT needed on LOC source objects.  For LOC:

```
SBN = floor(group_id / interleave_depth)
ESI = object_id
```

NOT present on repair objects (SBN/ESI derived per Section 7.1).

### 8.2. Object Length Extension

```
Object Length Extension {
  Extension Type (i) = 0xNN,
  Length (i),
  Original Object Length (i),
}
```

Original Object Length is a QUIC variable-length integer (1-8 bytes,
per RFC 9000 Section 16).

REQUIRED on CMAF source objects when FEC is enabled, to strip
zero-padding from the last source symbol.  NOT needed on LOC objects.

### 8.3. Auth Tag Extension

```
Auth Tag Extension {
  Extension Type (i) = 0xNN,
  Length (i) = N,
  Auth Tag (N),
}
```

N is fixed per stream and signaled in the catalog (auth scheme + tag
size).  OPTIONAL on both source and repair objects, all container modes.

Provides end-to-end content authentication across untrusted relays.
Auth on repair objects prevents compromised relays from injecting
malicious repair symbols that corrupt FEC recovery.  When auth is
enabled, receivers SHOULD verify repair auth tags before FEC decoding.

## 9. Block and Group Alignment

### 9.1. Single-Group Blocks (Interleave Depth = 1)

Each FEC block corresponds to exactly one MoQ Group:

```
Group 0:  [Obj 0] [Obj 1] ... [Obj K-1]  ->  FEC Block 0
Group 1:  [Obj 0] [Obj 1] ... [Obj K-1]  ->  FEC Block 1
```

SBN in repair objects equals Group ID.

### 9.2. Multi-Group Blocks (Interleave Depth > 1)

With depth D > 1, each FEC block spans D consecutive Groups:

```
Groups 0-3 (D=4):  ->  FEC Block 0, SBN = 0
Groups 4-7 (D=4):  ->  FEC Block 1, SBN = 1
```

SBN = floor(Group_ID / D).  First Group of block = SBN * D.

Repair objects for Block N are published after the last source object
of Group (N+1)*D - 1.

## 10. Interleaving

Interleaving spreads source symbols across time for burst loss
protection.  With depth D, symbols from D consecutive media units
form one source block:

```
Media Units:    M0   M1   M2   M3   M4   M5   M6   M7
                 |    |    |    |    |    |    |    |
Interleave=4:   [--- Block 0 ---]  [--- Block 1 ---]
                 S0   S1   S2   S3   S4   S5   S6   S7
                                |                    |
                        Block 0 Repair:      Block 1 Repair:
                           R0, R1               R2, R3
```

A burst affecting M1-M2 loses S1,S2 from Block 0.  With P >= 2,
Block 0 is recoverable.

| Application | Interleave Depth | Latency Impact |
|-------------|------------------|----------------|
| Interactive (gaming) | 1-4 | 33-133ms at 30fps |
| Low-latency live | 4-8 | 133-267ms at 30fps |
| Broadcast | 30 | ~1s at 30fps |
| ATSC 3.0 default | 60 | ~2s at 30fps |

## 11. Priority and Congestion

Repair tracks SHOULD use lower priority than source tracks.

| Track Type | Priority | Notes |
|------------|----------|-------|
| Control/Signaling | 0-1 | Highest |
| Source Media | 2-4 | Protected |
| Repair Symbols | 6-7 | Dropped first |

Under congestion, source media is preserved while repair overhead is
shed.  Receivers degrade to QUIC retransmission when FEC is
unavailable.

## 12. Hybrid Unicast and Multicast Delivery

### 12.1. Multicast and AMT Delivery

IP multicast (SSM per [RFC4607]) and AMT ([RFC7450]) are inherently
unreliable.  FEC enables recovery without retransmission:

- **Native SSM**: Direct multicast group join
- **AMT Tunneling**: Via relays discovered through DRIAD ([RFC8777])
- **TreeDN**: Hierarchical trees per [RFC9706] with FEC at each hop

The same MoQ objects (source and repair) can be sent over both unicast
(QUIC/WebTransport) and multicast (UDP/SSM/AMT).  Unicast receivers
MAY skip FEC; multicast receivers use FEC for recovery.

```
Publisher
    |
    v
+----------------+
|   MoQ Relay    |
+-------+--------+
        |
   +----+----+-----------+
   |         |            |
   v         v            v
Unicast   SSM Mcast   AMT Tunnel
(QUIC)    (Native)    (Encap)
   |         |            |
FEC:      FEC:         FEC:
Optional  Required     Required
```

### 12.2. Relay-Generated Repair

Relays MAY generate additional repair symbols for local conditions,
using RaptorQ's fountain code property.  Use cases:

- Edge relays increase overhead for lossy last-mile (WiFi, cellular)
- Regional relays adapt FEC to local loss patterns
- Relays regenerate repair when upstream symbols were lost

When generating repair:

1. The relay MUST decode the source block first
2. Generated repair SHOULD use ESIs >= K + P_publisher
3. The relay MAY use a separate track (e.g., "video/repair/relay")

Security: a compromised relay could inject malicious repair symbols.
Mitigations:

1. Publishers SHOULD use Auth Tag (Section 8.3) on source objects
   for post-recovery integrity verification
2. Receivers MUST distinguish publisher vs relay repair trust levels
3. Relay repair SHOULD use distinct tracks for separate trust policies

```
Publisher (K=32, P=8, 25% overhead)
    |
    | Source ESI: 0-31, Repair ESI: 32-39
    v
+------------------+
| Backbone Relay   |  (passes through)
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
 [Lossy   [Clean
  WiFi]    Fiber]
```

### 12.3. Multi-Path FEC Delivery

When data arrives via multiple paths (e.g., MoQ/QUIC + SSM + ATSC 3.0
RF), receivers combine FEC symbols from all paths:

1. **Format consistency**: All paths for the same source track SHOULD
   use the same repair container format
2. **Deduplication**: Receivers MUST deduplicate by SBN+ESI before
   FEC decoding
3. **Per-packet OTI**: Containers with per-packet OTI (e.g., MMTP)
   provide immediate OTI on mid-stream join from any path
4. **Reliable + unreliable**: Source on MoQ/QUIC, repair on multicast
   is valid.  Receivers MAY skip FEC when the reliable path delivers
   all source symbols

## 13. ATSC 3.0 Compatibility

### 13.1. Mapping to A/331 AL-FEC

| ATSC A/331 Concept | MoQ FEC Equivalent |
|--------------------|--------------------|
| `<FECParameters>` | FEC_CONFIG or catalog `fec` field |
| `<RepairFlow>` | Repair track subscription |
| `fecOTI` (in S-TSID) | OTI field in FEC_CONFIG |
| Source TOI range | Group ID range per Section 9 |
| `maximumDelay` | Interleave Depth * frame duration |
| `overhead` | (P / K) * 100 |

For ATSC 3.0 MMT mode ingest, use Repair Container 0x01 (MMTP).
See Appendix C.

### 13.2. S-TSID to FEC_CONFIG Conversion

```
S-TSID (per ATSC A/331):
<FECParameters maximumDelay="1000" overhead="25"
               fecOTI="F=32;T=1312;Z=4;N=1;Al=8">
  <ProtectedObject tsi="1">
    <SourceTOI x="0" y="255"/>
  </ProtectedObject>
</FECParameters>

Note: fecOTI uses A/331 field names.  F = source symbols per block
(K in this document), T = symbol size, Z = source blocks, N =
sub-blocks, Al = alignment.  RFC 6330 uses F for Transfer Length
(40 bits) -- a different concept.

Converts to FEC_CONFIG:
  FEC Algorithm: 0x02 (RaptorQ)
  Source Symbols Per Block: 32  (F)
  Repair Symbols Per Block: 8  (25% of 32)
  Symbol Size: 1312  (T)
  Interleave Depth: 4  (Z)
  OTI: [12-byte Common + Scheme-Specific per RFC 6330]
```

## 14. Interaction with CMAF Packaging

This specification works alongside CMAF packaging [I-D.ietf-moq-cmsf].
FEC encoding treats CMAF chunk payloads as source symbols.  The FEC
layer is agnostic to media container format.

```
+------------------------------------------------+
|              MoQ Track Namespace               |
|                                                |
|  Source Track (CMAF packaged)                  |
|  +------------------------------------------+ |
|  | Object 0: CMAF Header (ftyp, moov)       | |
|  | Object 1: CMAF Chunk (moof + mdat)       | |
|  | Object 2: CMAF Chunk (moof + mdat)       | |
|  +------------------------------------------+ |
|                                                |
|  Repair Track (this specification)             |
|  +------------------------------------------+ |
|  | Object 0: Repair for Block 0             | |
|  | Object 1: Repair for Block 1             | |
|  +------------------------------------------+ |
+------------------------------------------------+
```

### 14.1. CARP Integration

CARP [I-D.law-moq-carp] uses the same FEC catalog fields (Section 5.1)
and multicast endpoint format ([I-D.ramadan-moq-multicast] Section 7)
as any other MoQ packaging format.  No CARP-specific FEC signaling is
needed.

CMAF source objects with FEC enabled MUST carry the FEC Payload ID and
Object Length extensions (Sections 8.1, 8.2).  CARP receivers use
these identically to non-CARP CMAF receivers.

## 15. Security Considerations

### 15.1. FEC Does Not Provide Confidentiality

FEC repair symbols are derived from source data.  If source data is
encrypted, repair symbols operate on ciphertext.  FEC itself provides
no confidentiality.

### 15.2. Repair Symbol Manipulation

An attacker modifying repair symbols could cause incorrect source
reconstruction.  Integrity protection of repair tracks is RECOMMENDED
(e.g., QUIC packet protection).

### 15.3. Amplification

Repair symbols consume additional bandwidth.  Publishers MUST NOT
generate excessive repair.  The overhead (P/K) SHOULD be <= 50%.

### 15.4. Multicast FEC Security

Over multicast (SSM/AMT):

1. **No QUIC Protection**: Use DTLS-SRTP or equivalent for
   confidentiality
2. **Repair Injection**: Verify FEC block consistency (e.g., valid
   media structure after recovery)
3. **Source Authentication**: SSM verifies (S,G); AMT delegates to
   relay authentication
4. **Block Integrity**: MAY use block-level checksums; on failure,
   fall back to unicast retransmission

## 16. IANA Considerations

### 16.1. MoQ Message Type

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

Note: 0x50 is tentative.  Implementations SHOULD use 0xF000-0xFFFF
until IANA allocation.

### 16.2. MoQ Extension Header Type

| Type | Name | Reference |
|------|------|-----------|
| 0x50 | FEC_CONFIG | This document |

Note: Provisional pending control message extension headers in
[I-D.ietf-moq-transport].  Different IANA registry than Section 16.1.

### 16.3. FEC Algorithm Registry

"MoQ FEC Algorithms" registry, Specification Required:

| Value | Algorithm | Reference |
|-------|-----------|-----------|
| 0x00  | None      | This document |
| 0x01  | XOR       | This document |
| 0x02  | RaptorQ   | [RFC6330] |
| 0x03  | Reed-Solomon | [RFC5510] |
| 0x04-0xFF | Unassigned | |

### 16.4. Repair Container Registry

"MoQ Repair Container Types" registry, Specification Required:

| Value | Container | Reference |
|-------|-----------|-----------|
| 0x00  | Native MoQ | This document, Section 7 |
| 0x01  | MMTP | This document, Appendix C |
| 0x02-0xFF | Unassigned | |

### 16.5. MoQ Object Extension Header Types

| Type | Name | Reference |
|------|------|-----------|
| 0xNN | FEC Payload ID | This document, Section 8.1 |
| 0xNN | Object Length | This document, Section 8.2 |
| 0xNN | Auth Tag | This document, Section 8.3 |

Values TBD in coordination with MOQ WG.  Use experimental range
until allocated.

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

[I-D.law-moq-carp]
           Law, W., "CMAF-Aware Real-time Protocol",
           draft-law-moq-carp (work in progress).

[RFC4607]  Holbrook, H. and B. Cain, "Source-Specific Multicast
           for IP", RFC 4607, DOI 10.17487/RFC4607, August 2006.

[RFC7450]  Bumgardner, G., "Automatic Multicast Tunneling",
           RFC 7450, DOI 10.17487/RFC7450, February 2015.

[RFC8777]  Holland, J., "DNS Reverse IP AMT Discovery", RFC 8777,
           DOI 10.17487/RFC8777, April 2020.

[RFC9706]  Holland, J., et al., "TreeDN: Tree-Based CDN for Live
           Streaming to Mass Audiences", RFC 9706, December 2024.

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
    |    K=32, P=8, T=1312,                |
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
```

### A.2. Recovery Example

Receiver gets source objects 0,1,3-9 (object 2 lost) and repair
objects 0,1,2.  With K=10, P=3:

- 9 source symbols (ESI 0,1,3-9)
- 3 repair symbols (ESI 10,11,12)
- Total: 12 >= K=10 -- RaptorQ recovers ESI 2

### A.3. Per-Object Overhead Comparison

| Format | Overhead | Components |
|--------|----------|------------|
| Native (0x00), LOC source | 0 bytes | SBN/ESI from MoQ coords |
| Native (0x00), LOC + auth | N bytes | Auth Tag only |
| Native (0x00), repair | 0 bytes | Raw symbol, SBN/ESI derived |
| Native (0x00), repair + auth | N bytes | Symbol + Auth Tag |
| CMAF + FEC extensions | 9-12 bytes | FEC PID (8) + Obj Length (1-4) |
| CMAF + FEC + auth | 9+N to 12+N | FEC PID + Obj Length + Auth |
| MMTP (0x01) | 33 bytes | MMTP Hdr (12) + FEC PID (9) + OTI (12) |

LOC source objects have zero FEC overhead.  CMAF adds 9-12 bytes
(~68-90 bytes/s at 30fps, depth 4).  MMTP carries per-packet OTI for
self-describing operation and ATSC 3.0 compatibility.

## Appendix B. Catalog Example

Complete catalog with FEC and multicast:

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
        "interleaveDepth": 4,
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
        "algorithm": "xor",
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

The `multicast` field uses the format defined in
[I-D.ramadan-moq-multicast] Section 7.

## Appendix C. MMTP Repair Container Mapping (Informational)

NOTE: This appendix is informational.  It creates no normative
dependency on [ISO.23008-1].

### C.1. Motivation

The MMTP container (0x01) enables FEC interoperability between MoQ CDN
delivery and MMTP-based broadcast systems (SSM/ASM multicast, ATSC 3.0
MMT mode).  Repair payloads are unmodified MMTP repair packets per
[ISO.23008-1], enabling identical FEC recovery across all paths.

| Broadcast Path | MoQ CDN Path | Container |
|---|---|---|
| ATSC 3.0 MMT (RF) | MMTP passthrough | mmtp (0x01) |
| SSM + ATSC 3.0 | MMTP passthrough | mmtp (0x01) |
| MoQ only | raptorq native | native (0x00) |
| ATSC 3.0 ROUTE (RF) | CMAF over MoQ | native (0x00) |

### C.2. Wire Format

```
MoQ Object Payload (Repair Container 0x01) {
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

Extract FEC parameters at fixed byte offsets:

```
offset 0:   MMTP Header (12 bytes) -- skip
offset 12:  SS_Start (3 bytes, big-endian) -> SBN
offset 15:  RSB_length (3 bytes) -- skip
offset 18:  RS_ID (3 bytes) -> ESI derivation
offset 21:  OTI (12 bytes) -> RFC 6330 OTI
  offset 21: Transfer Length F (5 bytes, 40-bit BE)
  offset 26: Reserved (1 byte)
  offset 27: Symbol Size T (2 bytes, BE)
  offset 29: Num Source Blocks Z (1 byte)
  offset 30: Num Sub-Blocks N (2 bytes)
  offset 32: Alignment Al (1 byte)
offset 33:  Repair Symbol Data (T bytes) -> memcpy to decoder
```

K (source symbols per block) = ceil(F / T).

### C.4. Per-Packet OTI

Unlike native format where OTI is signaled once via FEC_CONFIG, MMTP
carries OTI in every repair packet (12 bytes at offset 21).  This
provides immediate OTI on mid-stream join and robustness to
FEC_CONFIG loss.  Overhead: ~720 bytes/s at typical rates (8 repair
symbols/block, ~7.5 blocks/s at 30fps depth=4).

### C.5. Relay Behavior

Relays ingesting MMTP sources pass repair packets through unmodified:

1. Receive MMTP repair packet (FEC Type=2)
2. Publish as MoQ object on repair track
3. Set Repair Container to 0x01 in FEC_CONFIG

### C.6. Relationship to FEC_CONFIG

When Repair Container is 0x01:

- FEC_CONFIG is OPTIONAL (OTI is per-packet)
- If sent, FEC_CONFIG K/P/T/depth are informational and SHOULD
  match per-packet OTI
- FEC_CONFIG remains useful for repair track discovery and signaling
  FEC availability before the first repair packet

## Authors' Addresses

Omar Ramadan
Blockcast
Email: omar@blockcast.net
