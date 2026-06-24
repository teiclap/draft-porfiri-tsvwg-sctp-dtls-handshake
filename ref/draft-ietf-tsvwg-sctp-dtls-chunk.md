---
docname: draft-ietf-tsvwg-sctp-dtls-chunk-latest
title: Stream Control Transmission Protocol (SCTP) DTLS Chunk
abbrev: SCTP DTLS Chunk
obsoletes: 6083
cat: std
ipr: trust200902
wg: TSVWG
area: Transport
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"

venue:
  group: Transport Area Working Group (tsvwg)
  mail: tsvwg@ietf.org
  github: gloinul/draft-westerlund-tsvwg-sctp-DTLS-chunk

author:
-
   ins:  M. Westerlund
   name: Magnus Westerlund
   org: Ericsson
   email: magnus.westerlund@ericsson.com
-
   ins: J. Preuß Mattsson
   name: John Preuß Mattsson
   org: Ericsson
   email: john.mattsson@ericsson.com
-
   ins: C. Porfiri
   name: Claudio Porfiri
   org: Ericsson
   email: claudio.porfiri@ericsson.com
-
   ins: M. Tüxen
   name: Michael Tüxen
   org: Münster University of Applied Sciences
   abbrev: Münster Univ. of Appl. Sciences
   street: Stegerwaldstrasse 39
   code: 48565
   city: Steinfurt
   country: Germany
   email: tuexen@fh-muenster.de

informative:
  RFC6083:
  RFC6458:
  RFC8446:
  I-D.ietf-tsvwg-rfc4895-bis:
  I-D.ietf-tsvwg-dtls-chunk-key-management:
  I-D.westerlund-tsvwg-sctp-DTLS-handshake:
    target: "https://datatracker.ietf.org/doc/draft-westerlund-tsvwg-sctp-dtls-handshake/"
    title: "Datagram Transport Layer Security (DTLS) in the Stream Control Transmission Protocol (SCTP) DTLS Chunk"
    author:
      -
       ins:  M. Westerlund
       name: Magnus Westerlund
       org: Ericsson
       email: magnus.westerlund@ericsson.com
      -
       ins: J. Preuß Mattsson
       name: John Preuß Mattsson
       org: Ericsson
       email: john.mattsson@ericsson.com
      -
       ins: C. Porfiri
       name: Claudio Porfiri
       org: Ericsson
       email: claudio.porfiri@ericsson.com
    date: Jul 2025

  ETSI-TS-38.413:
    target: "https://www.etsi.org/deliver/etsi_ts/138400_138499/138413/18.05.00_60/ts_138413v180500p.pdf"
    title: "NG Application Protocol (NGAP) version 18.5.0 Release 18"
    date: Mar 2025

normative:
  RFC4820:
  RFC4895:
  RFC5061:
  RFC8126:
  RFC9147:
  RFC9260:

  TLS-CIPHER-SUITES:
    target: "https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4"
    title: "TLS Cipher Suites"
    date: Nov 2023

updates:
  RFC5061

--- abstract

This document describes a method for adding Datagram Transport Layer
Security (DTLS) based authentication and cryptographic protection to the
Stream Control Transmission Protocol (SCTP).

This SCTP extension is intended to enable communication privacy for
applications that use SCTP as their transport protocol and allows applications
to communicate in a way that is designed to prevent eavesdropping and detect
tampering or message forgery.
Once enabled, this also applies to the SCTP payload as well as the SCTP
control information.

Applications using this SCTP extension can use most of the transport features
provided by SCTP and its other extensions.
The use of the SCTP Authentication extension defined in RFC 4895 is incompatible
with the extension defined in this document but would not provide any
additional service.
This implies that the Dynamic Address Reconfiguration as specified in RFC 5061
can only be used as described in this document.

This document obsoletes RFC 6083 and updates RFC 5061.

--- middle

# Introduction and Protocol Overview {#introduction}

This document extends the Stream Control Transmission Protocol (SCTP), as
specified in {{RFC9260}}, by defining the DTLS chunk format and the procedures
for negotiating and managing its usage.
DTLS chunk support is integrated into the SCTP implementation, enabling the
secure transfer of SCTP packets (including both control and DATA chunks).

The DTLS chunk protects a sequence of SCTP chunks by encapsulating their
DTLS-based ciphertext. This processing is based on DTLS 1.3, as specified in
{{RFC9147}}.

Key management is performed outside of the SCTP implementation and is out of scope
of this document. This process is referred to as the DTLS Key Management Method.
While these methods can also be based on DTLS 1.3, it is not a requirement.

The DTLS chunk in combination with the DTLS Key Management Method provides
mutual authentication, confidentiality, DTLS based data origin authentication,
integrity, and replay protection for SCTP packets.
The DTLS Key Management Method utilizes an API to provision the SCTP
association's DTLS chunk protection with key material to enable and rekey the
protection operations.

{{sctp-DTLS-chunk-layering}} is an example illustrating the DTLS chunk
processing in regard to SCTP and the Upper Layer Protocol (ULP) using
DTLS 1.3 as the DTLS Key Management Method.
Here the DTLS Key Management Method provides validation, i.e. using certificates,
handshaking, updating policies etc.

~~~~~~~~~~~ aasvg
+---------------+ +-------------------------------+
|      ULP      | |            DTLS 1.3           |
|               | |    +---------------------+    |
|               | | +->+    Key Exporter     +--+ |
|               | | |  +---------------------+  | |
|               | | |                           | |
|               | | |  +---------------------+  | |
|               | | +--+    Key Management   +  | |
|               | | |  +----------+----------+  | |
|               | | |             |             | |
|               | | | ContentType |             | |
|               | | |  +----------+----------+  | |
|               | | +->|        Record       |  | |
|               | |    | Protection Operator |  | |
|               | |    +----------+----------+  | |
+-------+-------+ +-----------------------------+-+
        ^                         ^             |
        |                         |             |
        +--+----------------------+             | keys
      PPID |                                    |
           V                                    V
+-----------------------------------------------+-+
|                    +---------------------+    | |
|        SCTP        |         Chunk       |<---+ |
|                    | Protection Operator |      |
|                    +---------------------+      |
+-------------------------------------------------+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-layering title="DTLS Chunk Layering
in Regard to SCTP and ULP" artwork-align="center"}

After receiving an SCTP packet and identifying the association using the
SCTP common header, the DTLS chunk is processed by the Chunk Protection Operator.
The Chunk Protection Operator performs replay protection, decryption,
and authentication. If this processing is successful, the encapsulated SCTP chunks
are further processed.

For outgoing traffic, after the SCTP stack has created the unprotected SCTP
packet containing control and/or DATA chunks, these SCTP chunks will be
processed by the Chunk Protection Operator for protection.
This results in a DTLS 1.3 record encapsulated in a DTLS chunk.

The method of secure key-management, e.g. based on DTLS 1.3, providing
initial mutual authentication, key establishment, and periodic
re-authentication and rekeying with Diffie-Hellman of the DTLS chunk
protection is defined in separate documents (see
{{key-management-considerations}}).  To prevent downgrade attacks
affecting the DTLS Key Management negotiation the DTLS Key Management
Method should implement specific procedures when deriving keys.

The Chunk Protection Operator performs protection operations on all
chunks of an SCTP packet.
No information is sent in plain text except for the following:

* The initial SCTP handshake.
* The initial DTLS Key Management traffic.
* The SCTP common header and the SCTP DTLS chunk header of protected packets.
* The INIT and INIT ACK chunks during an SCTP restart procedure.

Support of the DTLS chunk and the selection of a DTLS Key Management Method
are negotiated during the SCTP handshake using a new parameter.
DTLS Key Management and application traffic are then multiplexed
using the Payload Protocol Identifier (PPID).
This document defines the dedicated PPID 4242 for use by all DTLS Key Management
Methods.

Applications using the DTLS chunk can leverage most transport features provided by
SCTP and its extensions. However, the following limitations apply:

* Performing an SCTP restart without knowing the restart key material is not supported.
* The use of the lookup address in the Dynamic Address Reconfiguration
  extension as specified in {{RFC5061}} is not supported.

## Relationship to RFC 6083 and RFC 5061

This document obsoletes {{RFC6083}}, which defined the use of DTLS
over SCTP by encapsulating DTLS records as SCTP user data using the
SCTP-AUTH extension ({{RFC4895}}) for integrity protection of the SCTP
headers.  That approach suffered from several limitations: it could
not support SCTP user messages above 16384 bytes, it could not protect
SCTP control chunks, and it exposed significant SCTP metadata in clear
text.  The mechanism defined in this document replaces {{RFC6083}} by
integrating DTLS 1.3 record protection directly at the chunk level,
providing confidentiality and integrity for both user data and SCTP
control chunks without dependency on SCTP-AUTH.

This document updates {{RFC5061}} by restricting the use of the Dynamic
Address Reconfiguration extension when the DTLS chunk is in use.
Specifically, the lookup address feature of {{RFC5061}} MUST NOT be used,
and the dynamic address reconfiguration extension MUST NOT be used unless
DTLS chunk handling is enabled in both directions. These restrictions
are necessary because SCTP-AUTH, which {{RFC5061}} relies on for
authenticating ASCONF chunks, is incompatible with this extension.

# Conventions

{::boilerplate bcp14}

## Terminology

Chunk Protection Operator:

: The component within the SCTP implementation that performs DTLS record
  protection (encryption and authentication) on outgoing SCTP chunks and
  the corresponding verification and decryption on incoming DTLS chunks.

DTLS Key Context:

: DTLS key context includes key material (record payload key, sequence
  number protection key, and initialization vector (IV)) for sending
  and receiving, replay window for receiving, and last used sequence
  number for sending.

Primary DTLS Key Context:

: The DTLS key context used for normal SCTP association traffic, as
  distinguished from the restart DTLS key context.

Restart DTLS Key Context:

: A dedicated DTLS key context maintained for the sole purpose of
  protecting the SCTP restart procedure.

DTLS Key Management Method:

: A protocol or procedure, external to this specification, that performs
  mutual authentication, key establishment, and periodic rekeying for
  the DTLS chunk. Each method is identified by a DTLS Key Management
  Identifier.

DTLS Key Management Identifier:

: An 8-bit unsigned integer assigned by IANA that uniquely identifies a
  DTLS Key Management Method, enabling negotiation during the SCTP
  handshake via the DTLS Key Management Parameter.

Key Material:

: Key material is all cryptographic information needed for protection
  operation in one direction.


# Protocol Considerations

## DTLS Considerations {#DTLS-engines}

Once the DTLS Key Management Method has established its context, it
derives primary and restart key material for sending and receiving and
configures the Chunk Protection Operator via an API. This establishes
the necessary DTLS key contexts for SCTP chunk encryption and
decryption.  A DTLS key context for primary operations MUST be
created, while a DTLS key context for SCTP association restart SHOULD
be created.

DTLS key context includes key material such as record payload key,
sequence number protection key, and IV each for sending and receiving,
replay window for receiving, and last used sequence number for
sending. Each DTLS key context is associated with a three-value tuple
identifying the context, consisting of SCTP Association, the restart
indicator, and the DTLS epoch.

The DTLS Chunk uses a single configuration of the DTLS record format.
The DTLS Connection ID in the DTLS Record layer MUST NOT be used in
the DTLS Chunk as the full DTLS connection state is not used in the
DTLS Chunk and the DTLS key context anyway can be identified. The
length field MUST NOT be used as the DTLS chunk provides record length
information. Finally 16-bit Sequence Numbers are used as they give
maximum support for reordering and there are no byte savings possible
to ensure the 32-bit alignment for the encrypted record.

The first DTLS key context established for any SCTP association MUST
use epoch 3. Each subsequent DTLS key context will use the next
consecutive epoch value. Following the DTLS 1.3 specification {{RFC9147}} the
first DTLS record for each epoch will use sequence number 0.

The replay window for the DTLS Sequence Number needs to account for the
concurrent transmission of packets on multiple paths in multihomed associations.
In particular, this applies to packets containing HEARTBEAT chunks.  The window
size must be sufficiently large to accommodate these latency differences.

Endpoints implementing DTLS chunk MUST support DTLS records containing up to
2<sup>14</sup> (16384) bytes of plain text.

## SCTP Considerations

The SCTP authentication extension (SCTP-AUTH) defined in {{RFC4895}} is incompatible
with the extension defined in this document. Therefore, its support MUST NOT
be negotiated in combination with the support of the DTLS chunk.

In particular, the dynamic address reconfiguration defined in {{RFC5061}} cannot
use SCTP-AUTH. Instead, the DTLS chunk is used for authentication.
This introduces the following limitations:

* The lookup address MUST NOT be used.
* The dynamic address reconfiguration extension MUST NOT be used unless
  DTLS chunk handling is enabled in both directions.

To mitigate potential information leakage from packet size variations,
implementations MAY pad SCTP packets to uniform sizes.
However, the padding MUST be applied within the encryption envelope to ensure
the padding itself is protected.

Both SCTP and DTLS provide mechanisms for padding packets.
If padding of SCTP packets is desired to hide actual message sizes, it is
RECOMMENDED to use the SCTP Padding Chunk {{RFC4820}} to generate a consistent
SCTP payload size.
Support of this chunk is only required on the sender side; any SCTP receiver
will safely ignore the PAD Chunk. However, if the PAD chunk is not
supported DTLS padding MAY be used.

It should be noted that regardless of whether SCTP padding or DTLS padding
is used, the additional bytes are not accounted for by the SCTP congestion control.
Extensive use of padding has potential to worsen congestion situations, as the
SCTP association will consume more bandwidth than its fair share as determined by
congestion control.

The use of the SCTP PAD chunk is preferred, as it remains visible at the
SCTP layer. This enables future extensions or SCTP implementations to
correctly account for padding within congestion control mechanisms.
In contrast, DTLS padding conceals this packet expansion from the SCTP layer.

# New Chunk, Parameter and Error Causes

## DTLS Key Management Parameter {#protectedassoc-parameter}

The DTLS Key Management Parameter is used to negotiate the support of the
DTLS chunk and the Key Management Method used for the DTLS chunks.
The format of this chunk parameter is depicted in {{key-management-parameter}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Parameter Type = 0x8006    |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Tie Breaker                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Reserved |R|S|C| DTLS KMId #1  | DTLS KMId #2  | DTLS KMId #3  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:                                                               :
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| DTLS KMId #N  |                    Padding                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #key-management-parameter title="DTLS Key Management Parameter" artwork-align="center"}

{: vspace="0"}
Parameter Type: 16 bits (unsigned integer)
: This value MUST be set to 0x8006.
  Note that this parameter type requires the receiver to ignore the
  parameter and continue processing if the parameter type is not supported.
  This is accomplished (as described {{Section 3.2.1 of RFC9260}}) by
  the use of the upper bits of the parameter type.

Parameter Length: 16 bits (unsigned integer)
: This value holds the length of the parameter in bytes, which is the
  number of DTLS Key Management identifiers (N) plus 9.

Tie Breaker: 32 bits (unsigned integer)
: This is a 32-bit random number to be used to determine the client and
  server role for the Key Management Method.

Reserved: 5 bits (unsigned integer)
: The reserved bits MUST be set to 0 by the sender and MUST be ignored by the
  receiver.

R bit: 1 bit
: The (R)estart supported bit.

S bit: 1 bit
: The (S)erver role supported bit.

C bit: 1 bit
: The (C)lient role supported bit.

DTLS Key Management Identifier: 8 bits (unsigned integer)
: Each DTLS Key Management Identifier ({{IANA-Protection-Solution-ID}})
  is an 8-bit unsigned integer value indicating a DTLS Key Management Method.
  The DTLS Key Management Methods are listed in descending order of preference,
  i.e. the first listed in the parameter is the most preferred one and the last
  listed one is the least preferred by the sender of the parameter.
  The parameter MUST include at least one DTLS Key Management Identifier.

Padding: 0, 8, 16, or 24 bits (unsigned integer)
: The padding MUST be set to 0 by the sender and MUST be ignored by the receiver.

The DTLS Key Management Parameter MAY be included in the INIT and INIT ACK chunk
and MUST NOT be included in any other chunk.
Both peers include their respective preference list and the procedure
in {{establishment-procedure}} will determine the selected roles and chosen
method.

##  DTLS Chunk (DTLS) {#DTLS-chunk}

The DTLS chunk is used to hold the DTLS 1.3 record with the protected
payload of a plain text SCTP packet without the SCTP common header.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type = 0x41   | reserved    |R|         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Pre-Padding   |                                               |
+-+-+-+-+-+-+-+-+                                               |
|                                                               |
|                            Payload                            |
|                                                               |
|                               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                               |       Post-Padding            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-newchunk-crypt-struct title="DTLS Chunk" artwork-align="center"}

{: vspace="0"}
Type: 8 bits (unsigned integer)
: This value MUST be set to 0x41.
  It should be noted that the chunk type requires the receiver
  stop processing this SCTP packet, discard the unrecognized chunk and
  all further chunks, and report the unrecognized chunk in an ERROR
  chunk using the 'Unrecognized Chunk Type' error cause, if the receiver does
  not support this chunk type.
  This is accomplished (as described in {{Section 3.2 of RFC9260}}) by the use
  of the upper bits of the chunk type.

reserved: 7 bits
: Reserved bits for future use. These bits MUST be set to 0 by
  the sender and MUST be ignored by the receiver.

R: 1 bit (boolean)

: Restart indicator. If this bit is set this DTLS chunk is protected
  with a restart DTLS key context.


Chunk Length: 16 bits (unsigned integer)
: This value holds the length of the Payload in bytes plus 4 plus the Payload
  Pre-Padding length.

Pre-Padding: 8 bits
: The sender MUST pad with one zero byte and the receiver MUST ignore the
  padding bytes.

Payload: variable length
: This MUST contain exactly one DTLSCiphertext as specified in DTLS 1.3 {{RFC9147}}.

Post-Padding: 0, 8, 16, or 24 bits
: If the length of the Payload is not a multiple of 4 bytes, the sender
  MUST pad the chunk with all zero bytes to make the chunk 32-bit
  aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
  be ignored by the receiver.


From {{Section 4 of RFC9147}}, the DTLS record header has variable length and
is depicted in {{DTLSCiphertext-record-struct}}.

~~~~~~~~~~~ aasvg
    struct {
        opaque unified_hdr[variable];
        opaque encrypted_record[length];
    } DTLSCiphertext;
~~~~~~~~~~~
{: #DTLSCiphertext-record-struct title="DTLS DTLSCiphertext" artwork-align="center"}

The DTLSCiphertext contains the unified_hdr followed by the
encrypted_record, where unified_hdr has variable format but a single
selected configuration is used in the DTLS Chunk. The use of one byte
of Pre-Padding ensures 32-bit alignment of the encrypted_record in
relation to the start of the DTLS chunk, which allows a receiver to
perform an in-place decryption and then process the sequence of
chunks.  SCTP as specified in {{RFC9260}} guarantees that chunks start
on a 32-bit boundary.

The used DTLSCiphertext configuration contains the unified_hdr with
flags and the two least significant bits of the DTLS Epoch, a 16-bit
sequence number (S=1), no length field (L=0), and no Connection ID
(C=0). This results in a 3-byte unified_hdr (1 byte fixed header plus
2 bytes sequence number) and consequently 1 byte of Pre-Padding to
achieve 32-bit alignment of the encrypted_record. The used
DTLSCiphertext are shown in {{DTLSCiphertext-recommended}}.

~~~~~~~~~~~ aasvg

 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|0|0|1|0|1|0|E E|
+-+-+-+-+-+-+-+-+
|    16 bit     |
|Sequence Number|
+-+-+-+-+-+-+-+-+
|               |
|  Encrypted    |
/  Record       /
|               |
+-+-+-+-+-+-+-+-+

  DTLSCiphertext
    Structure
  (Used Configuration)
~~~~~~~~~~~
{: #DTLSCiphertext-recommended title="DTLSCiphertext used structure" artwork-align="center"}


## New Error Causes

This specification defines four new error causes.

### Missing DTLS Chunk Support {#enoprotected}

The DTLS Chunk Support Required error cause can be sent by the receiver of the
packet containing the INIT chunk to indicate that the receiver requires the
support of the DTLS chunk, but no DTLS Key Management Parameter was included in
the INIT chunk.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Cause Code = 100 (TBC)     |       Cause Length = 4        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #error-missing-dtls-chunk-support title="Error Missing DTLS Chunk Support" artwork-align="center"}

{: vspace="0"}
Cause Code: 16 bits (unsigned integer)
: This value MUST be set to 100 (TBC).

Cause Length: 16 bits (unsigned integer)
: This value MUST be set to 4.

This error cause MAY be included in an ABORT chunk.
It MUST NOT be included in any other chunk.

### No Common DTLS Key Management Method {#enocommonpsi}

The No Common DTLS Key Management Method error cause can be used by the receiver
of the packet containing the INIT chunk to indicate that the receiver does not
support any of the DTLS Key Management Methods offered by the sender of the packet
containing the INIT chunk.

The format of this error cause is depicted in {{error-cause-no-common-method}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Cause Code = 101 (TBC)     |       Cause Length = 4        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #error-cause-no-common-method title="Error Cause No Common DTLS Key Management Method" artwork-align="center"}

{: vspace="0"}
Cause Code: 16 bits (unsigned integer)
: This value MUST be set to 101 (TBC).

Cause Length: 16 bits (unsigned integer)
: This value MUST be set to 4.

This error cause MAY be included in an ABORT chunk.
It MUST NOT be included in any other chunk.

### DTLS Key Management Tie Breaker Collision {#tiebreakercol}

The DTLS Key Management Tie Breaker Collision error cause can be used to
indicate that both sides chose the same tie breaker.

The format of this error cause is depicted in {{error-cause-tie-breaker-collision}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Cause Code = 102 (TBC)     |       Cause Length = 4        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #error-cause-tie-breaker-collision title="Error Cause DTLS Key Management Tie Breaker Collision" artwork-align="center"}

{: vspace="0"}
Cause Code: 16 bits (unsigned integer)
: This value MUST be set to 102 (TBC).

Cause Length: 16 bits (unsigned integer)
: This value MUST be set to 4.

This error cause MAY be included in an ABORT chunk.
It MUST NOT be included in any other chunk.

### Incompatible DTLS Key Management Roles {#incompatroles}

The Incompatible DTLS Key Management Roles error cause can be used to
indicate that the roles chosen are incompatible.

The format of this error cause is depicted in {{error-cause-incompat-roles}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Cause Code = 103 (TBC)     |       Cause Length = 4        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #error-cause-incompat-roles title="Error Cause Incompatible DTLS Key Management Roles" artwork-align="center"}

{: vspace="0"}
Cause Code: 16 bits (unsigned integer)
: This value MUST be set to 103 (TBC).

Cause Length: 16 bits (unsigned integer)
: This value MUST be set to 4.

This error cause MAY be included in an ABORT chunk.
It MUST NOT be included in any other chunk.

# Procedures {#procedures}

## Establishment of a Protected Association {#establishment-procedure}

An SCTP endpoint wanting to use the DTLS chunk can operate in one of two modes:

Strict DTLS mode:
: The SCTP endpoint wants to use the DTLS chunk and is not willing to operate
  without DTLS chunk protection.

Loose DTLS mode:
: The SCTP endpoint want to use the DTLS chunk but is willing to operate without
  DTLS chunk protection.

A received DTLS Key Management Parameter is compatible, if the following two
conditions are satisfied:

1. At least one of the DTLS Key Management Identifiers listed in the parameter
   matches a locally supported one.

2. At least one of the roles (client or server) indicated in the parameter
   complements a local role.

To initiate an SCTP association, an SCTP endpoint wanting to use the DTLS chunk
MUST send an SCTP packet with an INIT chunk containing the DTLS Key Management
Parameter (see {{protectedassoc-parameter}}).
This parameter lists the supported DTLS Key Management Identifiers
(see {{key-management-parameter}}) in descending order of preference.

If an SCTP endpoint operating in strict DTLS mode receives an SCTP packet
containing an INIT chunk with a compatible DTLS Key Management Parameter,
it MUST reply with a packet containing an INIT ACK containing its own DTLS Key
Management Parameters.

If an SCTP endpoint operates in strict DTLS mode and receives an SCTP packet
with an INIT or INIT ACK chunk does not contain any DTLS Key Management Parameter
or only an incompatible one, the endpoint MUST send an SCTP packet with an
ABORT chunk.
It MAY include the appropriate error cause
"Missing DTLS Chunk Support" (see {{enoprotected}}),
"No Common DTLS Key Management Method" (see {{enocommonpsi}),
or "No Common DTLS Key Management Method" (see {{incompatroles}}).

If an SCTP endpoint operates in loose DTLS mode, it MAY continue with the
handshake in any case.

To ensure that each endpoint's Key Management Method knows which role it has and
both endpoints agree on which method that was chosen the below procedure MUST be
executed by both endpoints before entering the ESTABLISHED state.

First the Key Management role of each endpoint is determined. This is
done by evaluating the S and C bits in the two endpoints'
parameters. This falls into the following cases:

1. At least one endpoint indicates a single role (client or server) and the peer
   supports the other role. In this case the endpoint indicating a single role
   takes that role, and the other endpoint takes the reverse role.

2. Both endpoints indicate both roles, this is to be expected for endpoints
   supporting simultaneous open. In this case the role needs to be determined
   using the parameter's Tie breaker. The endpoint with the larger value SHALL be
   the server, and the other endpoint takes the client role. In case both
   endpoints have the same tie breaker value, the selection has failed and the
   association MUST be aborted.
   The ABORT chunk MAY include the SCTP error cause
   "DTLS Key Management Tie Breaker Collision" (see {{tiebreakercol}}).
   Endpoints are RECOMMENDED to attempt establishing a new SCTP association.

Once the key management roles have been established, the endpoints
determine the method to be used. The prioritized list of DTLS Key
Management Identifiers provided by the endpoint acting as the server
is evaluated, and the first identifier also supported by the peer
endpoint is selected.

When the SCTP association has been established, the process defined by the
selected DTLS Key Management Method MUST be followed for establishing
DTLS key contexts and installing them.

## DTLS Chunk Handling {#dtls-chunk-handling}

The DTLS chunk MUST NOT be bundled with any other chunk. Specifically,
it MUST appear as the first and only chunk in the packet.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #1                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            . . .                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #n                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-encrypt-chunk-states-1 title="Unprotected SCTP Packet" artwork-align="center"}

The diagram shown in {{sctp-DTLS-encrypt-chunk-states-1}} describes
the structure of an unprotected SCTP packet as described in {{RFC9260}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          DTLS Chunk                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-encrypt-chunk-states-2 title="Protected SCTP Packets" artwork-align="center"}

The diagram shown in {{sctp-DTLS-encrypt-chunk-states-2}} describes the
structure of a protected SCTP packet being sent.  Such packets contain only the
SCTP common header and one DTLS chunk.

Once the part of the DTLS key context responsible for sending DTLS chunks
has been configured by the application, all SCTP packets SHALL be sent using a DTLS chunk.

When an SCTP packet needs to be sent, the sequence of chunks is used
as `DTLSInnerPlaintext.content` and `DTLSInnerPlaintext.type` is set
to `application_data` {{RFC9147}}. Then the `DTLSCiphertext` is
computed per the DTLS 1.3 specification {{RFC9147}} and the configured
cipher suite, and the result is used as the payload. Finally the SCTP
common header is prepended.

When the DTLS chunk is used, the endpoint MUST consider the DTLS chunk header
and the overhead of DTLS to ensure that the final SCTP packet does not exceed
the PMTU.

Upon receipt of an SCTP packet in which a DTLS chunk is bundled with any
other chunk, the entire packet MUST be silently discarded.

After the application has restricted the SCTP packet handling to protected
SCTP packets only, an SCTP packet not containing a DTLS chunk MUST be
silently discarded.

When processing the payload of the DTLS chunk (i.e. the `DTLSCiphertext`),
the Restart flag in addition to the `unified_hdr` is used to find the keys for
processing the `encrypted_record` following DTLS 1.3 {{RFC9147}}.

After the `encrypted_record` has been verified and decrypted, the
corresponding chunks (the `DTLSInnerPlaintext.content`) are processed as
defined in the corresponding specifications.

If the Chunk Protection Operator experiences a non-critical error,
it MUST NOT abort the association.

## Termination of a Protected Association {#termination-procedure}

Note that the closure of any DTLS Key Management Method doesn't
compromise the capability of terminating the SCTP association gracefully as
that capability only relies on the Key Context and not on the DTLS Key
Management Method from where it has been derived.

## SCTP Restart Considerations  {#sec-restart}

This section deals with the handling of an unexpected INIT chunk
during an association lifetime as described in {{Section 5.2 of RFC9260}}
with the purpose of defining a protected restart procedure.

When the upper layer protocols require support of SCTP restart for associations
using the DTLS chunk, as in case of 3GPP NG-C protocol {{ETSI-TS-38.413}}, the
endpoint needs to support also the protected SCTP restart procedure described
below. Implementation of the protected restart procedure is RECOMMENDED; however,
it is not required, as it relies on the availability of persistent secure storage
for the restart DTLS key context. An endpoint will know that its peer supports
this protected SCTP restart procedure from the DTLS Key Management Parameter's R bit
{{key-management-parameter}}.

The protected SCTP restart procedure keeps the security characteristics of
an SCTP association using DTLS chunks.

In protected SCTP restart, INIT and INIT ACK chunks are sent
according to {{RFC9260}}, but COOKIE ECHO and COOKIE ACK chunks
are encrypted using DTLS chunks and the restart DTLS key contexts. The
endpoints MUST include the DTLS Key Management Parameter in the INIT
and INIT ACK, using the same method list, but with a new random Tie Breaker.

In order to support protected SCTP restart, the SCTP endpoints need
to allocate and maintain dedicated restart DTLS key contexts. SCTP
packets protected by these contexts will be identified in the DTLS
chunk with the R (Restart) bit set (see {{DTLS-chunk}}).  Both SCTP
endpoints need to ensure that restart DTLS key contexts are preserved
for supporting the protected SCTP restart use case.

For a protected SCTP endpoint to be available for protected SCTP restart,
the DTLS chunk requires access to a DTLS key context associated with the
SCTP association. This key context MUST be maintained in a well-defined
state such that both endpoints have a consistent view of the DTLS sequence
numbers and replay window (i.e., initialized but never used). The SCTP restart
DTLS key MUST NOT be used for any purpose other than SCTP association restart.

An SCTP endpoint wanting to be able to initiate a protected SCTP restart needs
to store securely and persistently the restart keys, and related DTLS epoch,
indexed so that when performing a restart with a peer endpoint, the restarting
endpoint can identify the right restart key and DTLS epoch and
initialize the restart DTLS key context for when restarting the SCTP
association.

The keys and epoch need to be stored securely and persistently so that they
survive the events that are causing the protected SCTP restart procedure to be
used, for instance a crash of the SCTP stack. The security considerations for
persistent secure storage of key material are further discussed in
{{sec-consideration-storage}}.

A DTLS chunk using the restart DTLS key context is identified by
having the R bit (Restart Indicator) set in the DTLS chunk (see
{{sctp-DTLS-chunk-newchunk-crypt-struct}}).  There's exactly one
active restart DTLS key context at a time, the newest. However, a crash at
the time having completed the DTLS Key Management exchange but failing to
commit the DTLS key context to persistent secure storage could result
in loss of the latest DTLS key context. Therefore, the endpoints
SHOULD retain the old restart DTLS key context until the
DTLS Key Management confirms the new ones are committed to secure storage.
For example, this can ensure that signals to terminate the old DTLS
key contexts (including the restart context) are never sent until the
new restart DTLS key context has been committed to
storage.

Implementations will also need to consider the risk of two-time pads,
i.e. the usage of the same nonce twice. An SCTP endpoint restarted and
sent a number of packets using the restart key context thus using
sequence number 1 to 8. If this endpoint restarts again, it is crucial
that the restarted endpoint does not reuse sequence number 1 to 8 a second
time. Ensuring that a sequence number is never reused in the context
of a crash may be hard, thus the simpler solution is to remove a restart
key context that is being used from the persistent storage to prevent
reuse.

~~~~~~~~~~~ aasvg

Initiator                                     Responder
    |                                             | -.
    |                                             |   +-------
    +--------------------(INIT)------------------>|   | Plain
    |<-----------------(INIT ACK)-----------------+   +-------
    |                                             | -'
    |                                             | -.
    |                                             |   +-------
    +---------[DTLS CHUNK(COOKIE ECHO)]---------->|   | Protected
    |<--------[DTLS CHUNK(COOKIE ACK)]------------+   +-------
    |                                             | -'
    |                                             |

~~~~~~~~~~~
{: #DTLS-chunk-restart title="Handshake of SCTP Restart for DTLS in SCTP" artwork-align="center"}

The {{DTLS-chunk-restart}} shows how the control chunks being
used for SCTP association restart are transported within DTLS in SCTP.

A restarted SCTP association MUST continue to use the restart DTLS key context,
for user traffic until a new primary DTLS key context will be available. The
implementors SHOULD initiate a rekeying as soon as possible,
and derive the primary and restart keys so that the time when no
restart DTLS key context is available is kept to a minimum. Note that another
restart attempt prior to having created new restart DTLS key context
for the new SCTP association will result in the endpoints being unable
to restart the SCTP association.

After restart, the next primary DTLS key context MUST use epoch 3,
i.e. the epoch value is reset. After having derived a new
primary DTLS key context, the endpoint installs it
and starts using it. The new restart DTLS key context is only installed
after all old in-flight restart packets have been received.

An SCTP endpoint supporting only normal SCTP restart and involved in
an SCTP association using DTLS chunks SHOULD NOT attempt to restart
the association. The effect will be that the restart initiator will
receive INIT ACK but then all sent packets with COOKIE ECHO will be
dropped until the peer endpoint times out the SCTP association from lack
of any response from the restarting node.

An SCTP endpoint supporting only legacy SCTP restart and involved
in an SCTP association using DTLS chunks, when receiving a COOKIE ECHO
chunk protected by DTLS chunk as described above, thus having the
R bit (Restart Indicator) set in the DTLS chunk (see
{{sctp-DTLS-chunk-newchunk-crypt-struct}}), MUST silently discard it.

# DTLS Key Management Method Considerations {#key-management-considerations}

This document specifies the mechanisms for protecting SCTP associations using
DTLS chunks, including an API for managing the corresponding key material.
While this document defines the interface, the upper layer is responsible
for the actual key management.

DTLS Key Management Methods define the procedures for initial key generation,
key updates and peer authentication. These procedures are out of scope for
this document.
Currently two DTLS Key Management Methods with different properties (such as
mutual authentication and rekeying) are defined:

* {{I-D.ietf-tsvwg-dtls-chunk-key-management}}
* {{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}

An SCTP endpoint MAY support multiple DTLS Key Management Methods subject to
implementation requirements and local security policies.

Every DTLS Key Management Method

* MUST be registered in the IANA Registry {{IANA-Protection-Solution-ID}}
  to receive a unique identifier, enabling negotiation during the SCTP handshake.
* SHOULD use the PPID from {{sec-iana-ppid}} to ensure that the DTLS Key
  Management Method related user messages are processed by the relevant entity.
* SHOULD ensure that the local receive keys are installed before the peer
  installs the corresponding send keys.
* MUST include in its key derivation the DTLS Key Management Parameter
  (including the parameter header and excluding the optional padding) as the
  sequence of bytes sent and received over the network during the SCTP handshake,
  to mitigate downgrade attacks.

# Abstract API  {#abstract-api}

This section describes an abstract API that is needed between a
DTLS Key Management Method and the DTLS chunk. This is an
example API and there are alternative implementations.

This API enables the cryptographic protection operations by setting
record payload key, sequence number keys, and initialization vector
(IV) for primary and restart DTLS contexts in both send and receive
direction. The record payload key is used by the cipher suite for DTLS
record protection ({{Section 5.2 of RFC8446}}). The initialization
vector (IV) is random material used to XOR with the sequence number to
create the nonce per {{Section 5.3 of RFC8446}}.  The sequence number
key is used to encrypt the sequence number ({{Section 4.2.3 of
RFC9147}}).

As this API references the key material as for send or receive, it will
be the responsibility of the Key Management Method used to define how
it maps the endpoint's role to which keys are for send and receive
direction respectively.

## Set Supported DTLS Key Management Parameters

Prior to attempting to establish an SCTP association an SCTP endpoint
needs to configure which DTLS Key Management Methods it supports, as
well as the roles and if restart is supported. This information is
included in the DTLS Key Management Parameter, which, when these
parameters are set, will be included in either INIT or INIT ACK chunks.

An endpoint can either support, client only, server only, or client
and server. The latter is for situations where both endpoints may attempt
to establish an SCTP association towards each other, potentially
causing a simultaneous sending of INIT chunks. Depending on port
configuration SCTP supports this happening during the association
establishment, and the result will be a single SCTP association. However,
to select the same Key Management Method on both sides, the SCTP stack
will resolve the key management role for this association.

Request: Set Supported DTLS Key Management Methods

Parameters:

* SCTP Association Handle:
: The handle to what may become an SCTP Association or a server port
accepting association establishment.

* Key Management Roles Supported:
: The endpoint indicates if it can act as client only, server only,
  or client and server.

* Support SCTP Restart (boolean):
: Indicate if the endpoint is capable of supporting SCTP restart when DTLS chunk
  has been negotiated. If both endpoints indicate flag then a restart has the
  potential to succeed.

* Requires DTLS Chunk security (boolean):
: SCTP association is only established if both endpoints support DTLS
  Chunk and have included the DTLS Key Management Parameter.

* List of Identifiers:
: A prioritized list of DTLS Key Management identifiers
  {{IANA-Protection-Solution-ID}} that are supported, from the most
  preferred to the least preferred.

Reply: Success or Error

Parameters: None

## Get Agreed DTLS Key Management Method and Role

After an SCTP association has been established the key management
function needs to know which method was agreed on and which role this
endpoint will have. The function provides both endpoints' actual
values from the DTLS Key Management Parameter, used in key derivation
to prevent downgrade attacks.

Request: Get Agreed DTLS Key Management Method and Role

Parameters:
* SCTP Association Handle:
: The handle to an established SCTP Association.

Reply: Success or Error

Parameters:

* Key Management Role:
: The role that was selected for this endpoint, one of client or server.

* Key Management Method:
: The selected Key Management Method as a DTLS Key Management Identifier.

* Down Grade Prevention Data:
: In network byte order the whole of the DTLS Key Management
  Parameter including header and excluding padding that the endpoint
  with the client role offered, followed by the corresponding
  parameter content of the endpoint with the server role.


## Cipher Suite Capabilities

The DTLS Key Management Method needs to know which cipher suites defined
for use with DTLS 1.3 that are supported by the DTLS chunk and its
protection operations block. All TLS cipher suites that are defined are
listed in the TLS cipher suite registry {{TLS-CIPHER-SUITES}} at IANA
and are identified by a 2-byte value. Thus this needs to return a list
of all supported cipher suites to the higher layer.

Request : Get Cipher Suites

Parameters : none

Reply   : Cipher Suites

Parameters : list of cipher suites

## Establish Send Key Material

The DTLS chunk can use one out of multiple sets of cipher suite and
corresponding key materials.

The following information needs to be provided when setting send key material:

Request : Establish Send Key Material

Parameters :

* SCTP Association:
: Reference to the SCTP association to set the key material for.

* Restart indication:
: A bit indicating whether the key material is for restart purposes

* DTLS Epoch:
: The DTLS epoch this key material is valid for. Note that Epoch lower than
  3 are not expected as they are used during DTLS handshake.

* Cipher Suite:
: 2 bytes cipher suite identification for the DTLS 1.3 cipher suite used
  to identify the DTLS AEAD algorithm to perform the DTLS record protection.
  The cipher suite is fixed for a (SCTP Association, Key) pair.

* Key Material:
: The cipher suite specific binary object containing all necessary
  information for protection operations such as Record Payload Key,
  Sequence Number Key and IV. The secret will be used by the DTLS 1.3
  client to encrypt the record. Binary arbitrary long object depending
  on the cipher suite used.


Reply: Established or Failed


## Establish Receive Key Material

The DTLS chunk can use one out of multiple sets of cipher suite and
corresponding key materials.

The following information needs to be provided when setting receive key material:

Request : Establish Receive Key Material

Parameters :

* SCTP Association:
: Reference to the SCTP association to set the key material for.

* Restart indication:
: A bit indicating whether the key material is for restart purposes

* DTLS Epoch:
: The DTLS epoch these keys are valid for. Note that Epoch lower than
  3 are not expected as they are used during DTLS handshake. The DTLS
  epoch must be the next in turn.

* Cipher Suite:
: 2 bytes cipher suite identification for the DTLS 1.3 cipher suite used
  to identify the DTLS AEAD algorithm to perform the DTLS record protection.
  The cipher suite is fixed for a (SCTP Association, Key) pair.

* Key Material:
: The cipher suite specific binary object containing all necessary
  information for protection operations such as Record Payload Key,
  Sequence Number Key and IV. The secret will be used by the DTLS 1.3
  client to encrypt the record. Binary arbitrary long object depending
  on the cipher suite used.

Reply : Established or Failed


## Destroy Send Key Material

A function to destroy the send key material for a given epoch for the
Primary or Restart DTLS Key Context for a given SCTP Association.

Request : Destroy Send Key Material

Parameters :

* SCTP Association

* Restart indication

* DTLS Epoch

Reply: Destroyed

Parameters : Success or Error

## Destroy Receive Key Material

A function to destroy the receive key material for a given epoch for
the Primary or Restart DTLS Key Context for a given SCTP Association.

Request: Destroy Receive Key Material

Parameters:

* SCTP Association

* Restart indication

* DTLS Epoch

Reply: Destroyed

Parameters: Success or Error

## Set Key to Use

Set which key to use to protect the future SCTP packets sent by the
SCTP Association.

Request: Set Key used

Parameters:

* SCTP Association

* Restart indication

* DTLS Epoch

Reply: Key set

Parameters: true or false

## Require Protected SCTP Packets

A function to configure an SCTP association to require that all SCTP
packets, excepts those with INIT or INIT ACK chunks, being received on
this endpoint are required to be protected in a DTLS chunk going
forward. Any unprotected SCTP packets to this SCTP association will be
discarded.

Parameters:

* SCTP Association

Reply: Acknowledgement


## Get AEAD Encryption Invocations

Get the number of AEAD encryption invocations (protected SCTP packets)
for a given epoch.

Request : Get AEAD Encryption Invocations

Parameters :

* SCTP Association

* Restart indication

* DTLS Epoch

Reply: AEAD Encryption Invocations

Parameters : non-negative integer

## Get AEAD Decryption Invocations

Get the number of AEAD decryption invocations for a given epoch.

Request : Get AEAD Decryption Invocations

Parameters :

* SCTP Association

* Restart indication

* DTLS Epoch

Reply: AEAD Decryption Invocations

Parameters : non-negative integer

## Get Failed AEAD Decryption Invocations

Get the number of failed AEAD decryption invocations for a given epoch.

Request : Get Failed AEAD Decryption Invocations

Parameters :

* SCTP Association

* Restart indication

* DTLS Epoch

Reply: Failed AEAD Decryption Invocations

Parameters : non-negative integer

## Configure Replay Protection

The DTLS replay protection in this usage is expected to be fairly
robust. Its depth of handling is related to maximum network path
reordering that the receiver expects to see during the SCTP
association. However as the actual reordering in number of packets is
a combination of how delayed one packet may be compared to another,
times the actual packet rate. This value may become larger for some
applications and may need to be tuned. Thus, having the potential for
setting this a more suitable value depending on the use case should be
supported.

Note this sets the configuration across any DTLS key context for a
given SCTP Association and both primary and restart usages.

Request : Configure Replay Protection

Parameters :

* SCTP Association

* Restart indication

* Configuration parameters

Reply: Replay Protection Configured

Parameters : true or false

# Socket API Considerations {#socket-api}

This section describes how the socket API defined in {{RFC6458}} needs to be
extended to provide a way for the application to control the usage of the
DTLS chunk.

A 'Socket API Considerations' section is contained in all SCTP-related
specifications published after {{RFC6458}} describing an extension for which
implementations using the socket API as specified in {{RFC6458}} would require
some extension of the socket API.
Please note that this section is informational only.

Please also note, that the API described in this section can change in a
non-backwards compatible way during the evolution of this document due to
changed functionality or gained experience during the implementation.

A socket API implementation based on {{RFC6458}} is extended by supporting
several new ``IPPROTO_SCTP``-level socket options and a new flag for
``recvmsg()``.

## ``SCTP_ASSOC_CHANGE`` Notification
When an ``SCTP_ASSOC_CHANGE`` notification (specified in {{Section 6.1.1 of
RFC6458}}) is delivered indicating a ``sac_state`` of ``SCTP_COMM_UP`` or
``SCTP_RESTART`` for an SCTP association where both peers support the
DTLS chunk, ``SCTP_ASSOC_SUPPORTS_DTLS`` should be listed in the
``sac_info`` field.

## A New Flag for ``recvmsg()`` (``MSG_PROTECTED``)

This flag is returned by ``recvmsg()`` in ``msg_flags`` for all user messages
for which all DATA chunks were received in protected SCTP packets.
This also means that if ``sctp_recvv()`` is used, ``MSG_PROTECTED`` is returned
in the ``*flags`` argument.

## Functions

### ``sctp_dtls_nr_cipher_suites()``

``sctp_dtls_nr_cipher_suites()`` returns the number of cipher suites supported
by the SCTP implementation.

The function prototype is:

~~~ c
unsigned int
sctp_dtls_nr_cipher_suites(void);
~~~

This function can be used in combination with ``sctp_dtls_cipher_suites()``.

### ``sctp_dtls_cipher_suites()``

``sctp_dtls_cipher_suites()`` returns the cipher suites supported by the
SCTP implementation.

The function prototype is:

~~~ c
int
sctp_dtls_cipher_suites(uint8_t cipher_suites[][2], unsigned int n);
~~~

and the arguments are
``cipher_suites``:
: An array where the supported cipher suites are stored. A cipher suite is
  represented by two ``uint8_t`` using the IANA assigned values in the
  TLS cipher suite registry {{TLS-CIPHER-SUITES}}.

``n``:
: The number of cipher suites which can be stored in ``cipher_suites``.

``sctp_dtls_cipher_suites`` returns ``-1``, if ``n`` is smaller than the number
of cipher suites supported by the stack. If ``n`` is equal to or larger than
the number of cipher suites supported by the SCTP implementation, the
cipher suites are stored in ``cipher_suites`` and the number of supported
cipher suites is returned.

## Socket Options

The following table provides an overview of the ``IPPROTO_SCTP``-level socket
options defined by this section.

| Option Name                      | Data Type                    | Set | Get |
| ``SCTP_DTLS_LOCAL_CONFIG``       | ``struct sctp_dtls_config``  | X   | X   |
| ``SCTP_DTLS_GET_CONFIG``         | ``struct sctp_dtls_config``  |     | X   |
| ``SCTP_DTLS_GET_LOCAL_KM_PARAM`` | ``struct sctp_dtls_kmp``     |     | X   |
| ``SCTP_DTLS_GET_PEER_KM_PARAM``  | ``struct sctp_dtls_kmp``     |     | X   |
| ``SCTP_DTLS_SET_SEND_KEYS``      | ``struct sctp_dtls_keys``    | X   |     |
| ``SCTP_DTLS_ADD_RECV_KEYS``      | ``struct sctp_dtls_keys``    | X   |     |
| ``SCTP_DTLS_DEL_RECV_KEYS``      | ``struct sctp_dtls_keys_id`` | X   |     |
| ``SCTP_DTLS_ENFORCE_PROTECTION`` | ``struct sctp_assoc_value``  | X   | X   |
| ``SCTP_DTLS_REPLAY_WINDOW``      | ``struct sctp_assoc_value``  | X   | X   |
| ``SCTP_DTLS_GET_STATS``          | ``struct sctp_dtls_stats``   |     | X   |
{: #socket-options-table title="Socket Options" cols="l l l l"}

``sctp_opt_info()`` needs to be extended to support:

* ``SCTP_DTLS_LOCAL_CONFIG``,
* ``SCTP_DTLS_GET_CONFIG``,
* ``SCTP_DTLS_GET_LOCAL_KM_PARAM``,
* ``SCTP_DTLS_GET_PEER_KM_PARAM``,
* ``SCTP_DTLS_ENFORCE_PROTECTION``,
* ``SCTP_DTLS_REPLAY_WINDOW``, and
* ``SCTP_DTLS_GET_STATS``.

### Get or Set the Local DTLS Key Management Configuration (``SCTP_DTLS_LOCAL_CONFIG``)

This socket option allows to get and set the DTLS Key Management identifiers being
sent to the peer during the handshake, the supported DTLS roles, and determines
whether the restart operation is supported and the use of the DTLS chunk is
required.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_config {
        sctp_assoc_t sdc_assoc_id;
        uint16_t sdc_flags;
        uint16_t sdc_nr_kmids;
        uint8_t sdc_kmids[];
};
~~~

``sdc_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  For ``setsockopt()``, only ``SCTP_FUTURE_ASSOC`` can be used.

``sdc_flags``:
: This field may contain any of the following flags and is composed of a
  bitwise OR of these values.
  ``SCTP_DTLS_CLIENT``:
  : All Key Management Methods in `sdc_kmids` can operate as a DTLS client.

  ``SCTP_DTLS_SERVER``:
  : All Key Management Methods in `sdc_kmids` can operate as a DTLS server.

  ``SCTP_DTLS_RESTART``:
  : All Key Management Methods in `sdc_kmids` support the restart operation.

  ``SCTP_DTLS_REQUIRED``:
  : The endpoint requires the use of the DTLS chunk.

``sdc_nr_kmids``:
: The number of entries in ``sdc_kmids``.

``sdc_kmids``:
: The DTLS Key Management identifiers which will be sent to the peer
  in the DTLS Key Management Parameter in the sequence provided.

This socket option can be used with ``setsockopt()`` only for SCTP endpoints in
the ``SCTP_CLOSED`` state for configuration.
It can be used with ``getsockopt()`` in any state.

### Get the DTLS Key Management Configuration (``SCTP_DTLS_GET_CONFIG``)

This socket option reports the DTLS Key Management Parameters negotiated during
the handshake.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_config {
        sctp_assoc_t sdc_assoc_id;
        uint16_t sdc_flags;
        uint16_t sdc_nr_kmids;
        uint8_t sdc_kmids[];
};
~~~

``sdc_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdc_flags``:
: This field contains any of the following flags and is composed of a bitwise
  OR of these values.
  ``SCTP_DTLS_CLIENT``:
  : Indicates that the Key Management Method provided in `sdc_kmids` acts as
    a client.

  ``SCTP_DTLS_SERVER``:
  : Indicates that the Key Management Method provided in `sdc_kmids` acts as
    a server.

  ``SCTP_DTLS_RESTART``:
  : Indicates that the Key Management Method provided in `sdc_kmids` supports
    the restart operation.

``sdc_nr_kmids``:
: The number of entries in ``sdc_kmids``.
  A value of 1 indicates that DTLS chunk support was successfully negotiated,
  while a value of 0 indicates negotiation failed.

``sdc_kmids``:
: The DTLS Key Management identifier which was negotiated.

This socket option will fail on any SCTP endpoint in state ``SCTP_CLOSED``,
``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Get the Local DTLS Key Management Parameter (``SCTP_DTLS_GET_LOCAL_KM_PARAM``)

This socket option provides the DTLS Key Management Parameter sent by the
endpoint during the handshake.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_kmp {
        sctp_assoc_t sdk_assoc_id;
        uint32_t sdk_nr_bytes;
        uint8_t sdk_bytes[];
};
~~~

``sdk_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdk_nr_bytes``:
: The number of entries in ``sdk_bytes``.

``sdk_bytes``:
: The DTLS Key Management Parameter sent by the endpoint as the sequence
  of bytes sent over the network. This includes the parameter header and
  excludes the optional parameter padding.

This socket option will fail on any SCTP endpoint in state ``SCTP_CLOSED``,
``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Get the Peer DTLS Key Management Parameter (``SCTP_DTLS_GET_PEER_KM_PARAM``)

This socket option provides the DTLS Key Management Parameter received by the
endpoint during the handshake.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_kmp {
        sctp_assoc_t sdk_assoc_id;
        uint32_t sdk_nr_bytes;
        uint8_t sdk_bytes[];
};
~~~

``sdk_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdk_nr_bytes``:
: The number of entries in ``sdk_bytes``.

``sdk_bytes``:
: The DTLS Key Management Parameter received by the endpoint as the sequence
  of bytes received over the network. This includes the parameter header and
  excludes the optional parameter padding.

This socket option will fail on any SCTP endpoint in state ``SCTP_CLOSED``,
``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Set Send Keys (``SCTP_DTLS_SET_SEND_KEYS``)

Using this socket option allows to add a particular set of keys used for
sending DTLS chunks.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_keys {
        sctp_assoc_t sdk_assoc_id;
        uint8_t sdk_cipher_suite[2];
        uint16_t sdk_restart;
        uint64_t sdk_epoch;
        uint16_t sdk_key_len;
        uint16_t sdk_iv_len;
        uint16_t sdk_sn_key_len;
        uint16_t sdk_unused;
        uint8_t sdk_keys[];
};
~~~

``sdk_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdk_cipher_suite``:
: The cipher suite for which the keys are used.

``sdk_restart``:
: If the value is ``0``, the regular keys are added, if a value different
  from ``0`` is used, the restart keys are added.

``sdk_unused``:
: This field is ignored.

``sdk_epoch``:
: The epoch for which the keys are added.

``sdk_key_len``:
: The length of the key specified in ``sdk_keys``.

``sdk_iv_len``:
: The length of the initialization vector specified in ``sdk_keys``.

``sdk_sn_key_len``:
: The length of the sequence number key specified in ``sdk_keys``.

``sdk_keys``:
: The key of length ``sdk_key_len`` directly followed by the initialization
  vector of length ``sdk_iv_len`` directly followed by the sequence number key
  of length ``sdk_sn_key_len``.

This socket option can only be used on SCTP endpoints in states other than
``SCTP_LISTEN``, ``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.
If the socket option is successful, all affected DTLS chunks sent will use the
specified keys until the keys are changed again by another call of this
socket option.

### Add Receive Keys (``SCTP_DTLS_ADD_RECV_KEYS``)

Using this socket option allows to add a particular set of keys used for
receiving DTLS chunks.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_keys {
        sctp_assoc_t sdk_assoc_id;
        uint8_t sdk_cipher_suite[2];
        uint16_t sdk_restart;
        uint64_t sdk_epoch;
        uint16_t sdk_key_len;
        uint16_t sdk_iv_len;
        uint16_t sdk_sn_key_len;
        uint16_t sdk_unused;
        uint8_t sdk_keys[];
};
~~~

``sdk_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdk_cipher_suite``:
: The cipher suite for which the keys are used.

``sdk_restart``:
: If the value is ``0``, the regular keys are added, if a value different
  from ``0`` is used, the restart keys are added.

``sdk_unused``:
: This field is ignored.

``sdk_epoch``:
: The epoch for which the keys are added.

``sdk_key_len``:
: The length of the key specified in ``sdk_keys``.

``sdk_iv_len``:
: The length of the initialization vector specified in ``sdk_keys``.

``sdk_sn_key_len``:
: The length of the sequence number key specified in ``sdk_keys``.

``sdk_keys``:
: The key of length ``sdk_key_len`` directly followed by the initialization
  vector of length ``sdk_iv_len`` directly followed by the sequence number key
  of length ``sdk_sn_key_len``.

This socket option can only be used on SCTP endpoints in states other than
``SCTP_LISTEN``, ``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Delete Receive Keys (``SCTP_DTLS_DEL_RECV_KEYS``)

Using this socket option allows to remove a particular set of keys used for
receiving DTLS chunks.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_keys_id {
   sctp_assoc_t sdki_assoc_id;
   uint32_t sdki_restart;
   uint64_t sdki_epoch;
}
~~~

``sdki_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdki_restart``:
: If the value is ``0``, the regular keys are removed, if a value different
  from ``0`` is used, the restart keys are removed.

``sdki_epoch``:
: The epoch for which the keys are removed.

This socket option can only be used on SCTP endpoints in states other than
``SCTP_CLOSED``, ``SCTP_LISTEN``, ``SCTP_COOKIE_WAIT`` and
``SCTP_COOKIE_ECHOED``.

### Get or Set Protection Enforcement (``SCTP_DTLS_ENFORCE_PROTECTION``)

Enabling this socket option on an SCTP endpoint enforces that received
SCTP packets are only processed, if they are protected.
All received packets with the first chunk not being an INIT chunk, INIT ACK
chunk, or DTLS chunk will be silently discarded.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_assoc_value {
   sctp_assoc_t assoc_id;
   uint32_t assoc_value;
};
~~~

``assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``assoc_value``:
  The value `0` represents that the option is off, any other value represents
  that the option is on.

This socket option is off by default on any SCTP endpoint.
Once protection has been enforced by enabling this socket option on an
SCTP endpoint, it cannot be disabled again.
Whether protection has been enforced on an SCTP endpoint can be queried
in any state other than ``SCTP_CLOSED``.
Protection can be enforced in any state other than ``SCTP_CLOSED``,
``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Get Statistic Counters (``SCTP_DTLS_GET_STATS``)

This socket options allows to get various statistic counters for a
specific SCTP endpoint.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_stats {
   sctp_assoc_t sds_assoc_id;
   uint64_t sds_dropped_unprotected;
   uint64_t sds_aead_failures;
   uint64_t sds_recv_protected;
   uint64_t sds_sent_protected;
   /* There will be added more fields before the WGLC. */
   /* There might be additional platform specific counters. */
};
~~~

``sds_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sds_dropped_unprotected``:
: The number of unprotected packets received for this SCTP endpoint after
  protection was enforced.

``sds_aead_failures``:
: The number of AEAD failures when processing received DTLS chunks.

``sds_recv_protected``:
: The number of DTLS chunks successfully processed.

``sds_sent_protected``:
: The number of DTLS chunks sent.

### Get or Set the Replay Protection Window Size (``SCTP_DTLS_REPLAY_WINDOW``)

This socket option can be used to configure the replay protection for SCTP
endpoints.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_assoc_value {
   sctp_assoc_t assoc_id;
   uint32_t assoc_value;
};
~~~

``assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{CURRENT|ALL}_ASSOC``.

``assoc_value``:
  The maximum number of DTLS chunks in the replay protection window.

# IANA Considerations {#IANA-Consideration}

This document adds registry entries or registries in the Stream Control
Transmission Protocol (SCTP) Parameters group handled by IANA:

* One new registry for the Key Management IDs

* One new SCTP Chunk Types and its corresponding Chunk Type Flags registry

* One new SCTP Chunk Parameter Type

* Two new SCTP Error Cause Codes

* A new Payload Protocol Identifier

## DTLS Key Management Method Identifiers {#IANA-Protection-Solution-ID}

Note: The RFC Editor is requested to replace RFC-To-Be with a reference to this document.

IANA is requested to create a new registry called "DTLS Key Management Method".
This registry is part of the Stream Control Transmission Protocol (SCTP)
Parameters grouping.

The purpose of this registry is to assign DTLS Key Management Method
Identifier for any DTLS Key Management Method used for the extension described
in this document.
Each entry will be assigned an 8-bit unsigned integer value from the suitable range.

| Identifier | Key Management Method Name                                     | Reference | Contact       |
| 0          | DTLS Chunk with Pre-shared cryptographic parameters            | RFC-To-Be | Draft Authors |
| 1-191      | Available for Assignment using Specification Required policy   |           |               |
| 192-254    | Available for Assignment using First Come, First Served policy |           |               |
| 255        | Reserved for extension of the identifier space                 |           |               |
{: #iana-psi title="DTLS Key Management Method Identifiers" cols="r l l l"}

New entries in the range 1-191 are registered following the Specification Required policy
as defined by {{RFC8126}}.  New entries in the range 192-254 are first come, first served with
expert review. The expert reviewers' primary purpose is to ensure that the registration is
relevant and not performed to consume the number space.

## SCTP Chunk Type

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Types" registry, IANA is requested to update the reference for the
DTLS chunk as depicted in {{iana-chunk-types}} with a reference to this
document.

| ID Value | Chunk Type        | Reference |
| 0x41     | DTLS Chunk (DTLS) | RFC-To-Be |
{: #iana-chunk-types title="DTLS Chunk Type" cols="r l l"}

IANA is requested to add the corresponding registration table for the chunk
flags of the DTLS chunk with the initial contents shown in {{iana-chunk-flags}}:

| Chunk Flag Value | Chunk Flag Name  | Reference |
| 0x01             | R bit            | RFC-To-Be |
| 0x02             | Unassigned       |           |
| 0x04             | Unassigned       |           |
| 0x08             | Unassigned       |           |
| 0x10             | Unassigned       |           |
| 0x20             | Unassigned       |           |
| 0x40             | Unassigned       |           |
| 0x80             | Unassigned       |           |
{: #iana-chunk-flags title="DTLS Chunk Flags" cols="r l l"}

## SCTP Chunk Parameter Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Parameter Types" registry, IANA is requested to update the reference for
the DTLS Key Management as depicted in {{iana-chunk-parameter-types}} with a
reference to this document.

| ID Value | Chunk Parameter Type | Reference |
| 0x8006   | DTLS Key Management  | RFC-To-Be |
{: #iana-chunk-parameter-types title="DTLS Key Management Chunk Parameter" cols="r l l"}


## SCTP Error Cause Codes {#IANA-Extra-Cause}

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Error Cause Codes" registry, IANA is requested to add the new
entries depicted below in {{iana-error-cause-codes}} with a
reference to this document.

| ID Value     | Error Cause Codes                         | Reference |
| 100 (TBC)    | Missing DTLS Chunk Support                | RFC-To-Be |
| 101 (TBC)    | No Common DTLS Key Management Method      | RFC-To-Be |
| 102 (TBC)    | DTLS Key Management Tie Breaker Collision | RFC-To-Be |
| 103 (TBC)    | Incompatible DTLS Key Management Roles    | RFC-To-Be |
{: #iana-error-cause-codes title="Error Cause Codes" cols="r l l"}

The suggested cause code will need to be confirmed by IANA.

## SCTP Payload Protocol Identifier {#sec-iana-ppid}

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Payload Protocol Identifiers" registry, IANA is requested to update the
reference for the PPID 4242 as depicted in {{iana-payload-protection-id}} with a
reference to this document.

| ID Value | SCTP Payload Protocol Identifier | Reference |
| 4242     | DTLS Key Management Messages     | RFC-To-Be |
{: #iana-payload-protection-id title="Protection Operator Protocol Identifier Registered" cols="r l l"}


# Security Considerations {#Security-Considerations}

All the security and privacy considerations of the security protocol
used as the Chunk Protection Operator apply.

DTLS replay protection MUST NOT be turned off.

## Privacy Considerations

Use of the SCTP DTLS chunk provides privacy to SCTP by protecting user
data and much of the SCTP control signaling. The SCTP association is
identifiable based on the 5-tuple where the destination IP and
port and VTag are fixed for each direction. Advanced privacy features such
as sequence number encryption might
therefore have limited effect.

## AEAD Limit Considerations

{{Section 4.5.3 of RFC9147}} defines limits on the number of records
q that can be protected using the same key as well as limits on the
number of received packets v that fail authentication with each key.
To adhere to these limits the DTLS Key Management Method can
periodically poll the DTLS protection operation function to see
when a limit has been reached or is close to being reached.
Instead of periodic polling, a callback can be used.

## Downgrade Attacks {#Downgrade-Attacks}

Downgrade attacks may attempt to force the DTLS Key Management Method
by altering the content of INIT chunk, for instance by removing
all offered DTLS Key Management Methods but the one desired. This is possible
if the attacker is an on-path attacker that can modify packets
because INIT and INIT ACK chunks are plain text.

Preventing the downgrade attacks is implemented by using the content
of the DTLS Key Management Parameter sent in the INIT chunk plus the
content of the DTLS Key Management Parameter received in the INIT ACK
chunk from the responder for deriving the keys from the handshake
secrets obtained during the DTLS initial handshake. Using the whole
content of the parameters ensures that the role selection and thus the
method selection isn't possible to manipulate.

If the attacker succeeds in changing the DTLS Key Management Parameter
in either INIT, INIT ACK or both chunks, the peers will not be able to
derive the same keys and the association will fail to complete.  Any
modification will result in association failure, thus preventing
down-grade.

In case any DTLS Key Management Method does not include the parameter
content in its key-derivation down-grade would be possible if that
DTLS Key Management Method is selected. It is up to endpoint policies
to determine what level of protection it deems necessary against
down-grade attacks.

## Persistent Secure Storage of Restart Key Context {#sec-consideration-storage}

The restart DTLS key context needs to be stored securely and persistently. Securely
as access to this security context may enable an attacker to perform a restart,
resulting in a denial of service on the existing SCTP association. It can also
give the attacker access to the ULP. Thus the storage needs to provide at least
as strong resistance against exfiltration as the main DTLS key context store.

How to realize persistent storage is highly dependent on the ULP and
how it utilizes restarted SCTP associations.  One way can be to have
an actual secure persistent storage solution accessible to the
endpoint. In other use cases the persistence part might be
accomplished by keeping the current restart DTLS key context with the
ULP State if that is sufficiently secure.


# Acknowledgments

The authors thank Hannes Tschofenig and Tirumaleswar Reddy for their
participation in the design team and their contributions to this document.
We also like to thank Amanda Baber with IANA for feedback on our IANA registry.
