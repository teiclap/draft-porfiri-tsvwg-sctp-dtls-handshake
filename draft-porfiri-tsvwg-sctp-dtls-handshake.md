---
docname: draft-porfiri-tsvwg-sctp-dtls-handshake-latest
title: Transport Layer Security (TLS) based key-management of the Stream Control Transmission Protocol (SCTP) DTLS Chunk
abbrev: TLS for DTLS in SCTP
obsoletes:
cat: std
ipr: trust200902
wg: TSVWG
area: Transport
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"

venue:
  group: Transport Area Working Group (tsvwg)
  mail: tsvwg@ietf.org
  github: teiclap/draft-porfiri-tsvwg-sctp-dtls-handshake

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

informative:
  RFC3758:
  RFC4895:
  RFC5061:
  RFC5705:
  RFC6083:
  RFC9525:
  I-D.ietf-tls-rfc8446bis:
  I-D.ietf-uta-rfc6125bis:

  ANSSI-DAT-NT-003:
    target: <https://www.ssi.gouv.fr/uploads/2015/09/NT_IPsec_EN.pdf>
    title: Recommendations for securing networks with IPsec
    seriesinfo:
      ANSSI Technical Report DAT-NT-003
    author:
      -
        ins: Agence nationale de la sécurité des systèmes d'information
    date: August 2015

  KTH-NCSA:
    target: "http://kth.diva-portal.org/smash/get/diva2:1902626/FULLTEXT01.pdf"
    title: "On factoring integers, and computing discrete logarithms and orders, quantumly"
    seriesinfo:
      "KTH, School of Electrical Engineering and Computer Science (EECS), Computer Science, Theoretical Computer Science, TCS. Swedish NCSA, Swedish Armed Forces."
    author:
      -
       ins: M. Ekerå
       name: Martin Ekerå
    date: October 2024

normative:
  RFC6347:
  RFC8446:
  RFC8996:
  RFC9147:
  RFC9260:
  RFC9325:

  I-D.draft-ietf-tsvwg-sctp-dtls-chunk:
    target: "https://datatracker.ietf.org/doc/draft-ietf-tsvwg-sctp-dtls-chunk/"
    title: "Stream Control Transmission Protocol (SCTP) DTLS chunk"
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


--- abstract

This document defines how Transport Layer Security (TLS) 1.3
is used to establish keys for securing SCTP using the DTLS Chunk mechanism.
It specifies how a TLS handshake is used to establish the initial security
context for an SCTP association and describes procedures for further TLS
handshakes are used for establishing other security contexts being used
for key updates and post-handshake authentication. The goal is to enable
authenticated, and confidential communication over SCTP using the DTLS Chunk,
leveraging standardized TLS features for key management and rekeying.

--- middle

# Introduction {#introduction}

The Stream Control Transmission Protocol (SCTP) is a transport protocol
designed to support message-oriented communication with features such as
multi-streaming and multi-homing.
In many deployments, particularly those telecommunication networks and WebRTC
data channels, it is essential to provide confidentiality, integrity, and peer
authentication for SCTP traffic.

{{RFC6083}} defines a mechanism for securing SCTP by encapsulating application
payload in DTLS 1.0/1.2, establishing a secure channel between SCTP endpoints
and relying on SCTP-AUTH {{RFC4895}} to prevent attacks on the SCTP protocol
itself.

However, with the introduction of DTLS 1.3 {{RFC9147}}, the
protocol underwent significant changes, including removal of renegotiation,
a new key schedule, and support for post-handshake operations.
Without additional description, RFC 6083 cannot be used with DTLS 1.3.

{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}} defines an mechanism alternative to
{{RFC6083}} for securing Application Payload and SCTP by encapsulating SCTP's
chunks in DTLS 1.3, establishing a secure channel between SCTP endpoints.

This document describes the usage of the Transport Layer
Security version 1.3 (TLS) {{RFC8446}} protocol for key-management
of the SCTP DTLS Chunk packet protection
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}} securing Stream Control
Transmission Protocol (SCTP) {{RFC9260}}.  This combination of
specifications is intended as a replacement to DTLS/SCTP
{{RFC6083}} and usage of SCTP-AUTH {{RFC4895}}. The combination of
SCTP DTLS Chunk and the key-management defined in this document we
refer to as TLS for DTLS in SCTP.

This document describes:

* How the TLS 1.3 handshake establishes the initial security context between
  SCTP endpoints.

* How keying material for the DTLS chunk is derived and associated with the SCTP
  association.

* How multiple TLS 1.3 handshakes are used for key updates and post-handshake
  authentication, for supporting long-lived secure sessions.



## Conventions

{::boilerplate bcp14}


## Terminology

   This document uses the following terms:

   Association:
   : An SCTP association.

   Connection:
   : A TLS 1.3 connection.

   DTLS Key context:
   : Keys, derived from a TLS 1.3 connection, and all relevant data that needs
   to be provided to the SCTP DTLS Chunk.  Each DTLS key context is associated
   with a three value tuple identifying the context, consisting of SCTP
   Association, the restart indicator, and the DTLS epoch.

   Primary DTLS Key context:
   : A DTLS Key context used to protect the regular SCTP traffic, i.e. not a
    restart DTLS Key context.

   Restart DTLS Key context:
   : A DTLS Key context to be used for an SCTP Association Restart

   Stream:
   : A unidirectional stream of an SCTP association.  It is
   uniquely identified by a stream identifier.

   TLS for DTLS in SCTP:
   : The set of data, process and procedures described in this document.

   Traffic:
   : The stream of DATA and Control chunks being sent on any stream between SCTP
   Endpoints in the scope of an Association

## Abbreviations

   AEAD:
   : Authenticated Encryption with Associated Data

   DKC:
   : DTLS Key Context

   DTLS:
   : Datagram Transport Layer Security

   MTU:
   : Maximum Transmission Unit

   PPID:
   : Payload Protocol Identifier

   SCTP:
   : Stream Control Transmission Protocol

   TLS:
   : Transport Layer Security

   ULP:
   : Upper Layer Protocol


# Charactesistics

TLS for DTLS in SCTP provides mutual authentication of endpoints, data
confidentiality, data origin authentication, data integrity
protection, and data replay protection of SCTP packets. Ensuring
these security services to the application and its upper layer
protocol over SCTP.  Thus, it allows client/server applications to
communicate in a way that is designed with communications privacy
and preventing eavesdropping and detect tampering or message
forgery.

Applications using TLS for DTLS in SCTP can use all currently existing
transport features provided by SCTP and its extensions, in some
cases with some limitations, as specified in
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}. TLS for DTLS in SCTP supports:

* preservation of message boundaries.

* no limitation on number of unidirectional and bidirectional streams.

* ordered and unordered delivery of SCTP user messages.

* the partial reliability extension as defined in {{RFC3758}}.

* multi-homing of the SCTP association per {{RFC9260}}.

* the dynamic address reconfiguration extension as defined in
      {{RFC5061}} (Limitations apply).

* User messages of any size.

* SCTP Packets with a protected set of chunks up to a size of
     2<sup>14</sup> (16384) bytes.

The main benefit of this key-management solution over the solution
proposed by the WG is two fold:

* First, that this solution do not require any extensions to
  (D)TLS 1.3 to be implemented to enable long lived sessions.

* Secondly, that it is TLS 1.3 based rather than DTLS 1.3. The
  availability of DTLS 1.3 even just with minimal core functionality
  is extremely limited.  Thus, having a solution based on TLS where
  there a multiple available implementations, and no need to await
  additional implementation work is a significant benefit.


# Architecture {#architecture}

This document describes how Transport Layer Security (TLS) 1.3 is used
to establish keys for securing SCTP using the DTLS Chunk as defined in
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.  This approach combines the
performance and encryption flexibility of DTLS Chunks with the
integrated key management capabilities of TLS and the multiple TLS
connection approach.

The key characteristics of the solution are as follows:

* Application data is protected using DTLS Chunks.

* The TLS handshake is used to establish keying material, algorithms,
  and parameters for use with DTLS Chunks as well as authenticate the
  peer.

* Rekeying and re-authentication is achieved by opening a new TLS
  connection over the secured SCTP assocation. Thus mutual
  authentication and a new ephemeral key exchange is performed,
  enabling deriving a new DTLS Key Context with forward secrecy for
  the next DTLS Chunk epoch.

In this document we use the terms DTLS Key context for indicating the
pair of keys and the initilization vector (IV), derived from a TLS1.3
connection, and all relevant data that needs to be provided to the
SCTP DTLS Chunk to enable DTLS encryption, decryption and
authentication. A complete bi-directional DTLS Key context includes
Keys for sending and receiving, replay window, and last used sequence
number. Each DTLS key context is associated with a three value tuple
identifying the context, consisting of SCTP Association, the restart
indicator, and the DTLS epoch.

The Upper Layer Protocol's Application data is never transmitted in
TLS record-layer application_data records.  Instead, application data
is sent via SCTP DATA chunks which are protected by the DTLS Chunk
Protection Operator.  This operator encapsulates all SCTP chunks into
a DTLS Chunk, applying the negotiacted DTLS cipher suit's protection.

The figure {{overview-layering}} illustrates the architecture,
highlighting the role of the upper-layer protocol (ULP), which acts as
the consumer of SCTP's transport services.  The ULP may interface
directly with the SCTP stack or operate through the TLS 1.3
key-management library.

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
|               | | +->|    TLS Record       |  | |
|               | |    | Protection Operator |  | |
+               | |    +----------+----------+  | |
+-------+-------+ +-----------------------------+-+
        ^                          ^            |
        |                          |            |
        +--+-----------------------+            | keys
      PPID |                                    |
           V                                    V
+-----------------------------------------------+-+
|                    +---------------------+    | |
|        SCTP        |     DTLS Chunk      |<---+ |
|                    | Protection Operator |      |
|                    +---------------------+      |
+-------------------------------------------------+
~~~~~~~~~~~
{: #overview-layering title="Architecture"}

## TLS for DTLS in SCTP

TLS for DTLS in SCTP starts with a SCTP association where DTLS 1.3 Chunk
usage has been negotiated and this key-management method has been
agreed in the SCTP INIT and INIT-ACK.

Following the initial SCTP association setup, a TLS 1.3 handshake is
performed to mutually authenticate the endpoints and to derive keying
material for the DTLS Chunk Protection Operator.  The TLS exporter, as
defined in Section 7.5 of {{RFC8446}}, is used to derive this keying
material, i.e. the initial DTLS Key Contexts.  It leverages the same
cryptographic algorithms that were negotiated during the TLS handshake
for use with the DTLS Record Layer, thereby eliminating the need for
separate algorithm negotiation for the DTLS Chunk.  However, this
approach requires that only algorithm suites compatible with both TLS
1.3 and the DTLS Chunk be configured and supported in TLS session.
Once the DTLS Key Contexts have been derived, the TLS1.3 connection
is closed.

Whenever either peers need to re-key, the peer asking for re-keying
will initiate a new TLS 1.3 handshake to mutually authenticate the
endpoints and to derive keying material for the DTLS Chunk Protection
Operator. Once again the TLS exporter, as defined in Section 7.5 of
{{RFC8446}}, is used to derive this keying material, and from that
material the new DTLS Key Contexts. Once the DTLS Key Contexts have
been derived, the TLS1.3 connection is closed.

The new DTLS Key Contexts will be used for replacing the aged DTLS
Key Contexts as specified in {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

## Setting the Keys Initially

The basic functionality and how things are related is described
below:

* The process starts with a SCTP association where DTLS 1.3 Chunk
   usage has been negotiated and this key-management method has been
   agreed in the SCTP INIT and INIT-ACK. Here we assume that the
   management method is the one described in this document.

* After initial handshake, a mechanism in SCTP takes the decision
   about the role of the peer with respect of TLS. One of the peers
   will be decided to be Initiator and the other to be Responder.
   The information about the role of the current node is communicated
   from SCTP to the Key Management function via API so that Key Management
   has knowledge of its own role and the lists of the Initiator's
   Key Management methods offered by the Initiator and the list of the
   Key Management methods offered by the Responder.

* To initialize and authenticate the peers the TLS handshake is
   initiated at the Initiator peer, the TLS handshake messages are
   exchanged as SCTP user messages with the DTLS Chunk Key-Management
   Messages PPID (see section 10.6 of
   {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}) until an initial TLS
   connection has been established. If the TLS handshake fails, the
   SCTP association is aborted.

* With successful handshake and authentication of the peer the Key
   Management will use a Key Exporter as defined in section 7.5 of
   {{RFC8446}}, using the label defined in {{iana-export-label}}
   and a context built as an array of bytes sequenced as follows:

   1 byte : direction

   1 byte : traffic or reset key

   variable length : the list of the Key Management Methods from Initiator

   2 bytes : the Key Management Method selected by the Responder

* The Key exporter will be used for creating 4 DTLS Key Contexts,
   one DTLS Key Context per direction for Traffic Cases and one
   DTLS Key Context per direction for Restart Cases.


* The DTLS Chunk specifies that in the receiving SCTP endpoint each
   incoming SCTP packet on any of its interfaces and ports are matched
   to the SCTP association based on ports and VTAG in the common
   header. Using the indicated DTLS Key context(s) for that SCTP
   association the content of the DTLS chunk is attempted to be
   processed, including replay protection, decryption, and integrity
   checking. If decryption and integrity verification was
   successful the produced plain text of one or more SCTP chunks are
   provided for normal SCTP processing in the identified SCTP
   association along with associated per-packet meta data such as path
   received on, original packet size, and ECN bits.


The following figure shows when the key exporter is used to get key
material from the TLS connection.  Then keys are derived from that and
the diagram also shows when these keys are configured for the DTLS
chunk and also when the SCTP stack is instructed to discard received
SCTP packets, when they are unprotected. This procedure needs to be
followed to avoid deadlocking the establishment of the protected
SCTP assocation.

~~~~~~~~~~~ aasvg

Client                                             Server
------                                             ------

 Record 0
 ClientHello                -------->
 (epoch=0)
                                             +---------------+
                                             | SET RECV KEYS |
                                             +---------------+
                                                     Record 0
                            <--------             ServerHello
                                                    (epoch=0)
                                        {EncryptedExtensions}
                                                    (epoch=2)
                                                {Certificate}
                                                    (epoch=2)
                                          {CertificateVerify}
                                                    (epoch=2)
                                                   {Finished}
                                                    (epoch=2)
			
+---------------+
| SET RECV KEYS |
+---------------+
| SET SEND KEYS |
+---------------+
 Record 1
 {Certificate}              -------->
 (epoch=2)
 {CertificateVerify}
 (epoch=2)
 {Finished}
 (epoch=2)
                                             +---------------+
                                             | SET SEND KEYS |
                                             +---------------+
                                                     Record 1
                            <--------                   [ACK]
                                                    (epoch=3)
+--------------------+                  +--------------------+
| ENFORCE PROTECTION |                  | ENFORCE PROTECTION |
+--------------------+                  +--------------------+

~~~~~~~~~~~
{: #setting-keys-initially title="Setting the Keys initially"}

Note that the epoch noted in {{setting-keys-initially}} are the TLS session's
epochs, not the epoch used for the DTLS Chunk. The DTLS chunk's initial key
context will use epoch=3.

The key derivation takes into account which protection solution identifiers
have been sent and received. This way the communication is protected against
downgrade attacks against the SCTP handshake.

## Key Management Role Determination

This key management method supports also SCTP association that was
simultanously opened by both peers. Based on the procedures in
{{RFC9260}} both endpoints may initiate opening an SCTP association
to each other and this resulting in a single SCTP association.

To enable the key-management procedure to work correctly in such cases
where an endpoint can act both as an initiator (client) and as a
responder (server) the DTLS Key Management Parameter provides
information such that the key management role is determined. This
procedures is defined in {{role-determination}}.

With the role determined and this Key Management method selected, the
above TLS handshake procedure, and associated configuration of the
DTLS chunk, is initiated by the one that got the Initiator Role. Thus
also the input to Key Management Downgrade protection is determined.


## Rekeying and Key-Life Time

When mutual re-authentication or rekeying is needed or desired by
either endpoint a new TLS handshake is exchanged performed between the
SCTP endpoints to establish a new TLS connection. New DTLS Key
contexts are created for the next DTLS chunk epoch for this SCTP
assocation from this new TLS connection. As the new (Epoch=N+1) and
old (Epoch=N) DTLS key context can coexist a simplified procedure for
enabling the new DTLS key context is used as shown below
{{setting-keys-rekey}}.

~~~~~~~~~~~ aasvg

Client                                             Server
------                                             ------

 Record 0
 ClientHello                -------->
 (epoch=0)
                                                     Record 0
                            <--------             ServerHello
                                                    (epoch=0)
                                        {EncryptedExtensions}
                                                    (epoch=2)
                                                {Certificate}
                                                    (epoch=2)
                                          {CertificateVerify}
                                                    (epoch=2)
                                                   {Finished}
                                                    (epoch=2)
				             +-------------------+
                                             | SET RECV KEYS N+1 |
                                             +-------------------+
+-------------------+
| SET RECV KEYS N+1 |
+-------------------+
| SET SEND KEYS N+1 |
+-------------------+

Record 1
 {Certificate}              -------->
 (epoch=2)
 {CertificateVerify}
 (epoch=2)
 {Finished}
 (epoch=2)

                                            +-------------------+
                                            | SET SEND KEYS N+1 |
                                            +-------------------+
                                                     Record 1
                            <--------                   [ACK]
                                                    (epoch=3)


Wait 2 min

+---------------------+                   +---------------------+
| Remove KEYS Epoch=N |                   | Remove KEYS Epoch=N |
+---------------------+                   +---------------------+
~~~~~~~~~~~
{: #setting-keys-rekey title="Setting the Keys for rekeying"}

All rekeying MUST be using ephemeral key exchange and MUST NOT use the
TLS Key-Update mechanism to avoid confusion about the properties of
the DTLS Key Contexts for the DTLS chunk. After a short while (no
longer than 2 min) to enable any outstanding packets to drain from the
network path between the endpoints, the old key context can be deleted
from the DTLS chunk's key store.

The lifetime of the TLS 1.3 connections used for deriving the DTLS Key
Context should be limited to the time strictly needed for completing
the key derivation, to ensure that at most one TLS 1.3 connection
exists at a time.


# TLS messages over SCTP User Messages  {#tls-user-message}

The key-management TLS sessions have their TLS message (individual TLS
records) sent as SCTP user messages using reliable in-order delivery
on stream 0 using the DTLS Chunk Key-Managment Messages PPID (4242)
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.  To ensure clear indication
of the usage of the TLS session one or more complete TLS record is
prefaced by a single byte indicating the epoch of the DTLS key context
being created. With these transport requirements of TLS records
between the key-managmeent function in each endpoint it is possible to
ensure that each TLS session is receiving its records in order, a
requirement for TLS.

TLS messages (records) for any Handshake TLS connection, i.e. that are
not DTLS records containing protected SCTP chunk payloads, will be
sent as SCTP user messages using the format defined in
{{sctp-dtls-user-message}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Epcoh     |                                               |
+---------------+            TLS Message                        |
|                                                               |
|                               +-------------------------------+
|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-dtls-user-message title="TLS User Message Structure"}


 Epoch: 8 bits
 : The 8 lowest bits of the full epoch counter (64-bits) of
 DTLS key context whose keys are exported from this TLS session.

 TLS Message: variable length
 : One or more TLS records.

# DTLS Chunk Integration

The {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}} contains a high-level
description of the basic DTLS in SCTP architecture, this section deals
with details related to the TLS 1.3 inband key-management
integration with SCTP.

## SCTP Association Life Cycle.

TLS for DTLS in SCTP uses inband key-management, thus the TLS handshake
for a Key-Management TLS Connection establishes a new TLS
Connection with the remote peer from which both peers will derive
DTLS Key Contexts. As soon as the SCTP State Machine enters ESTABLISHED
state, DTLS in SCTP is responsible for progressing to where the DTLS
Chunk is fully configured and the ULP will be protected.

### Protection Initialization {#protect-init}

When the SCTP Association enter ESTABLISHED state, the initiator will
start the handshake according to {{tls-handshake}}.

When a successful handshake has been completed, the Primary DTLS Key
Context and the Restart DTLS Key Context will be created by deriving
the keys and IVs from the key-management TLS 1.3 connection. These will
be installed in the Chunks Protection Operator as defined in this document to
avoid dead lock, ensure successful protection and enabling the ULP
traffic. The key-management TLS 1.3 connection should then be closed.

### SCTP Association Ongoing

When an SCTP Association is protected the established Primary DTLS key
context is used for Chunk protection operation of the payload of SCTP
chunks in each packet per the DTLS Chunk specification
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

When necessary to meet requirements on key life time or periodic
re-authentication of the peer and establishment of new forward secrecy
keys, a new TLS 1.3 key-managment connection is established as
further specified in {{parallel-dtls}}, derive and install new DTLS
Key Contexts and then close the new TLS 1.3 connection and remove the old
DTLS Key contexts.


### SHUTDOWN states

When the SCTP association leaves the ESTABLISHED state per {{RFC9260}}
to be shutdown the DTLS Chunk's key contexts are kept and continue to
protect the SCTP packet payloads through the shutdown process.

When the association reaches the CLOSED state as part of the SCTP
association closing process, any TLS 1.3 connections existing
for this association is terminated without further
transmissions.


## TLS Connection Handling {#dtls-connection-handling}

It's up to TLS key-management function to manage the TLS
connections and their related DTLS Key context in the Chunk
Protection Operator.

### Add a New TLS Connection {#add-tls-connection}

Either peer can add a new TLS 1.3 connection to the SCTP association at
any time, but no more than 1 TLS 1.3 connections can exist at the same
time. Details of the handshake are described in {{further_tls_connection}}.

As either endpoint can initiate a TLS handshake at the same time,
either endpoint may receive a TLS ClientHello message when it has
sent its own ClientHello. In this case the ClientHello from the
endpoint that had the TLS Client role in the establishment of the
previous TLS connection shall be continued to be processed and the
other dropped.

When the handshake has been completed successfully, the new DTLS Key
contexts are established by exporting keys and installing them in the
Chunk Protection Operator, and the new DTLS Key context are
immediately started to be used.
If the handshake is not completed successfully, a new TLS
handshake attempt will be tried.

### Remove an existing TLS Connection {#remove-tls-connection}

A TLS connection is removed when the derived DTLS Key Context is in use.
It is RECOMMENDED to not initiate
removal until at least one SCTP packet protected by the new DTLS Key Context
has been received, and any transmitted packets protected
using the new DTLS Key Context has been acknowledged, alternatively
waiting for one Maximum Segment Lifetime (120 seconds) has elapsed
since the last SCTP packet protected by the old DTLS Key Context was
transmitted.

Either peers can initiate the removal of a TLS connection from the
current SCTP association when needed when a new DTLS Key Context has been established.
Closing the TLS connection when the SCTP association is in
PROTECTED and ESTABLISHED state is done by having the TLS connection
sending a TLS close_notify.

### Considerations about removal of TLS Connections {#removal_tls_consideration}

Removal of a TLS connection may happen under circumstances as
described above in {{remove-tls-connection}} in different states
of the Association. This section describes how the implementation
should take care of the TLS connection removal in details:

* In case the TLS connection is removed and no useable TLS Key Context exist
for key-management DTLS 1.3 negotiation, the Association MUST be
ABORTED.

It is up to the implementation to guarantee that a DTLS Key Context exists
all the time, for avoiding that undesired TLS connection closure causes
the Association abortion.

## DTLS Key Update

TLS Key Update MUST NOT be used.  DTLS Key Context replacement MUST
be used instead, by means of creating a new TLS connection as specified
in {{parallel-dtls}}, deriving the new Primary DTLS Key Context, the
new Restart DTLS Key Context and then closing the TLS connection.

## Error Cases

As TLS has its own error reporting mechanism by exchanging TLS alert
messages no new TLS related cause codes are defined to use the error
handling defined in {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

When the handshake of TLS connection encounters an error it may report that
issue using TLS alert message to its peer by putting the created TLS
record in a SCTP user message (see {{tls-user-message}}) with the
Handshake PPID. This is independent of what to do in relation to the
SCTP association.  Depending on the severity of the error different
decisions can be taken.

However, as it is not expected that the key-management TLS
connection will have any activity at all between completing the
handshake, and the TLS connection closing, it is unlikely
that any error will occur.


# TLS Considerations

## Version of TLS

   This document defines the usage of TLS 1.3 {{RFC8446}}.
   Earlier versions of TLS MUST NOT be used. It is expected
   that TLS for DTLS in SCTP as described in
   this document will work with future versions of TLS.

   Only one version of TLS MUST be used during the lifetime of an
   SCTP Association, meaning that the procedure for replacing the TLS
   version in use requires the existing SCTP Association to be
   terminated and a new SCTP Association with the desired TLS version
   to be instantiated.

## Configuration of Key-Management TLS

### General

   Many of the TLS registries have a "Recommended" column. Parameters
   not marked as "Y" are NOT RECOMMENDED to support.
   Non-AEAD cipher suites or cipher suites without
   confidentiality MUST NOT be supported. Cipher suites and parameters
   that do not provide ephemeral key-exchange MUST NOT be supported.

   The Cipher suites negotiated in the Key-Management TLS Connection
   MUST only include those supported by the DTLS Chunk. The DTLS
   Chunk is expected to have an API capability to determine the Cipher
   Suit Capabilities, see Abstract API in Section 10.1 of
   {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

### Authentication and Policy Decisions

   TLS for DTLS in SCTP MUST be mutually authenticated. Authentication is the
   process of establishing the identity of a user or system and
   verifying that the identity is valid. DTLS only provides proof of
   possession of a key. TLS for DTLS in SCTP MUST perform identity
   authentication. It is RECOMMENDED that TLS for DTLS in SCTP is used with
   certificate-based authentication.

   When certificates are used the application using TLS for DTLS in SCTP is
   responsible for certificate policies, certificate chain validation,
   and identity authentication (HTTPS does for example match the
   hostname with a subjectAltName of type dNSName). The application
   using TLS for DTLS in SCTP defines what the identity is and how it is
   encoded and the client and server MUST use the same identity
   format. Guidance on server certificate validation can be found in
   [RFC9525]. TLS for DTLS in SCTP enables periodic transfer
   of mutual revocation information (OSCP stapling) every time a new
   parallel connection is set up. All security decisions MUST be based
   on the peer's authenticated identity, not on its transport layer
   identity.

It is possible to authenticate DTLS endpoints based on IP addresses in
certificates. SCTP associations can use multiple IP addresses per SCTP
endpoint. Therefore, it is possible that DTLS records will be sent
from a different source IP address or to a different destination IP
address than that originally authenticated. This is not a problem
provided that no security decisions are made based on the source or
destination IP addresses.

### New Connections {#new-connections}

Implementations MUST set up a new TLS connection using a full handshake
before any of the certificates expire.
It is RECOMMENDED that all negotiated and
exchanged parameters are the same except for the timestamps in the
certificates. Clients and servers MUST NOT accept a change of identity
during the setup of a new connections, but MAY accept negotiation of
stronger algorithms and security parameters, which might be motivated
by new attacks.

Allowing new connections can enable denial-of-service attacks. The
endpoints MUST limit the number of simultaneous connections to two.

To force attackers to do dynamic key exfiltration and limit the
amount of compromised data due to key compromise, implementations MUST
have policies for how often to set up new connections with ephemeral
key exchange such as ECDHE. Implementations SHOULD set up new
connections frequently to force attackers to dynamic key
extraction. E.g., at least every hour and every 100 GB of data which
is a common policy for IPsec [ANSSI-DAT-NT-003]. See
[I-D.ietf-tls-rfc8446bis] for a more detailed discussion on key
compromise and key exfiltration in (D)TLS. As recommended in
{{KTH-NCSA}}, resumption can be used to chain the connections,
increasing security by forcing an adversary to break them in sequence.

For many TLS for DTLS in SCTP deployments the SCTP association is expected to
have a very long lifetime of months or even years. For associations
with such long lifetimes there is a need to frequently re-authenticate
both client and server by setting up a new connection using a full
handshake. TLS Certificate
lifetimes significantly shorter than a year are common which is
shorter than many expected SCTP associations protected by TLS for DTLS in
SCTP.


### TLS 1.3

TLS 1.3 is used instead of TLS 1.2 being a newer protocol that
addresses known vulnerabilities and only defines strong algorithms
without known major weaknesses at the time of publication.

TLS 1.3 requires rekeying before algorithm specific AEAD limits have
been reached. Implementations setup a new TLS connection to handle
the need for new keys, alternatively terminate the SCTP Assocation.

In TLS 1.3 any number of tickets can be issued in a connection and
the tickets can be used for resumption as long as they are valid,
which is up to seven days. The nodes in a resumed connection have the
same roles (client or server) as in the connection where the ticket
was issued. Resumption can have significant latency benefits for
quickly restarting a broken DTLS/SCTP association. If tickets and
resumption are used it is enough to issue a single ticket per
connection.

The PSK key exchange mode : psk_ke MUST NOT be used as it does not
provide ephemeral key exchange.

# Establishing TLS for DTLS in SCTP

This section specifies how TLS for DTLS in SCTP is established using
Key-Management TLS Connections and the DTLS Chunk
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

A TLS for DTLS in SCTP Association is built up with a Key-Management TLS
connection, from that TLS connection Primary DTLS Key Contexts and
Restart DTLS Key Contexts are derived and used to setup
the DTLS Chunk Protection Operator(see {{overview-layering}}).

The Key-Management TLS connection is established as part of extra
procedures for the DTLS chunk initial handshake (see
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}).


## Key Management Role Determination {#role-determination}

In order to get the rules for Proper Key Derivation at the Initiator
and at the Responder, the following algorithm will be used:

* Input : Role of the node, own sequence of DTLS Key Management methods in preference order, remote sequence of DTLS Key Management methods in preference order.

* Select the chosen DTLS Key Management method starting from the Responder's in preferred order
matching the Initiator's method

* Create a key derivation list by using the full list from the Initiator plus the selected DTLS Key Management method

The key derivation list will be used for the lifetime of the Association.

TBD: Take the message flow into account which was presented by Magnus.

TBD: Provide formulas for deriving the keys and improve the message sequence
diagram.


## DTLS Key Context derivation {#dtls-key-derivation}

This section describes how DTLS Key Contexts are derived from the
TLS handshake using the TLS Exporter as defined by {{RFC8446}}.
The TLS exporter requires Context and Label parameters.

### The Context

The context for TLS Exporter is an arbitrary long array of bytes.
TLS for DTLS in SCTP requires the contexts to be using the following rules:

* The sequence of bytes MUST follow the order

1 byte : direction (Client/Server)

1 byte : traffic or reset key

variable length : the list of the Key Management Methods from Initiator

2 bytes : the Key Management Method selected by the Responder

* The list of Key Management Methods from Initiator MUST follow
exactly the order of bytes provided in the INIT/INIT-ACK
DTLS Key Management Parameter, as an example in the following
{{key-management-parameter}} the list of Key Management Methods
will use the sequence of bytes starting from the 5th up to the
10th.

* The Key Management Method selected by the Responder will follow
the same rule. If the selected Management Method in {{key-management-parameter}}
is DTLS Key Management Id 2, the sequence of the 2 bytes will
be the 7th and the 8th.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Parameter Type = 0x8006    |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  DTLS Key Management Id #1    |  DTLS Key Management Id #2    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| DTLS Key Management Id #3     | Padding                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #key-management-parameter title="DTLS Key Management Parameter" artwork-align="center"}


### The Label
  TLS exporter label specifications below is following {{RFC5705}}
  using the label defined in {{iana-export-label}}.

  There are two sets of keys: one for the Primary DTLS key context and
  one for the Restart DTLS Key Context.

  Each set consists of one
  client and one server side write key. In addition each key needs an
  Initilization Vector (IV) that is used by the record processing in
  TLS to create the nonce, See Section 5.3 of {{RFC8446}}.

  The client
  and server roles are here in relation to key-management TLS session
  roles. So the DTLS Client will install the key derived using the
  EXPORTER_DTLS_IN_SCTP_PRIMARY_CLIENT_KEY label as its write key for
  the Primary DTLScontext, and use the
  EXPORTER_DTLS_IN_SCTP_PRIMARY_SERVER_KEY as its Primary DTLS context
  read key. Correspondlingly the
  EXPORTER_DTLS_IN_SCTP_RESTART_CLIENT_KEY is used to export the key
  used by the endpoint that acted as DTLS Client as write key for the
  restart DTLS key context. And the
  EXPORTER_DTLS_IN_SCTP_RESTART_SERVER_KEY as the DTLS client's read
  key for the restart DTLS Key Context. Correspondligy the IV values
  needs to be exported using the corresponding _IV label.

  The following labels are defined:

  * EXPORTER_DTLS_IN_SCTP_PRIMARY_CLIENT_KEY
  * EXPORTER_DTLS_IN_SCTP_PRIMARY_CLIENT_IV
  * EXPORTER_DTLS_IN_SCTP_PRIMARY_SERVER_KEY
  * EXPORTER_DTLS_IN_SCTP_PRIMARY_SERVER_IV
  * EXPORTER_DTLS_IN_SCTP_RESTART_CLIENT_KEY
  * EXPORTER_DTLS_IN_SCTP_RESTART_CLIENT_IV
  * EXPORTER_DTLS_IN_SCTP_RESTART_SERVER_KEY
  * EXPORTER_DTLS_IN_SCTP_RESTART_SERVER_IV

  To ensure that downgrade attack on the protection solution offered
  is not possible the context used will be the full sequence of
  Protection Solution Identifiers as include in the DTLS 1.3 Chunk
  Protected Association (Section 4.1 of
  {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}) sent by the SCTP
  assocation initiator. Thus, any downgrade attack on this will result
  in a mismatch in produced keys as the initiator will use what it
  actually offered and the responder a truncated or modified sequence.

  The length of the exported key or IV material depends on the need for the
  negotiatated cipher suit for the protection.


## TLS Handshake {#tls-handshake}

### Protection Initilization using TLS connection {#initial_tls_connection}

   The handshake of the initial TLS connection is part of the
   TLS for DTLS in SCTP Association initialization. The key-management TLS
   connections progress interacts with the key installation
   for the SCTP associations DTLS chunk.

~~~~~~~~~~~ aasvg

Initiator                                     Responder
    |                                             | -.
    +--------------------[INIT]------------------>|   |
    |<-----------------[INIT-ACK]-----------------+   | SCTP
    +----------------[COOKIE ECHO]--------------->|   +-----
    |<----------------[COOKIE ACK]----------------+   |
    |                                             | -'
    |                                             | -.
    +-----------[DATA(TLS Client Hello)]--------->|   |
    |<---[DATA(TLS Server Hello ... Finished)]----+   | DTLS
    +----[DATA(TLS Certificate ... Finished)]---->|   +-----
    |<--------------[DATA(TLS ACK)]---------------+   |
    |                                             | -'
    |                                             | -.
    +-------[DTLS CHUNK(DATA(APP DATA))]--------->|   | APP DATA
    +<-------[DTLS CHUNK(DATA(APP DATA))]---------+   +---------
    |                    ...                      |   |
    |                    ...                      |   |

~~~~~~~~~~~
{: #sctp-TLS-initial-dtls-connection title="Handshake of initial TLS connection" artwork-align="center"}


   SCTP Handshake is strictly compliant to {{RFC9260}}. The DTLS 1.3
   Chunk Protected Association parameter (Section 4.1 of
   {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}) is included containing
   the Protection Solution identifier (See {{sec-iana-psi}}) for this
   documents key-management at a suitable preference position
   depending on local policy. And in case this key-management solution
   is the most preferred then the process continues as stated below
   and depiceted in {{sctp-protection-initilization}}.

~~~~~~~~~~~ aasvg

Initiator                                            Responder
    |                                                    |
 0. +------------------------[INIT]--------------------->|
    |<---------------------[INIT-ACK]--------------------+
    +--------------------[COOKIE ECHO]------------------>| 1.
 2. |<--------------------[COOKIE ACK]-------------------+
    |                                                    |
    |  Key Manager                           Key Manager |
    |    |                                          |    |
 3. +--->| CRYPTO UP                      CRYPTO UP |<---+
    |    |                                          |    |
    |    +---------[DATA(DTLS Client Hello)]------->| 4. |
    |    |<-[DATA(DTLS Server Hello ... Finished)]--+ 5. |
    | 6. +--[DATA(DTLS Certificate ... Finished)]-->| 7. |
    |                                                    | -.
 8. +------------[DTLS CHUNK(DATA(APP DATA))]----------->|   | APP DATA
    +<-----------[DTLS CHUNK(DATA(APP DATA))]------------+   +---------
    |                         ...                        |   |
    |                         ...                        |   |

~~~~~~~~~~~
{: #sctp-protection-initilization title="The steps of Interaction between Key-Management and DTLS Chunk API" artwork-align="center"}


   0. The Initiator initiates an SCTP Association and provides the
      DTLS 1.3 Chunk Protected Association parameter preference
      ordered list of supporter Protection Solutions. The offered
      parameter list is remembered by the Key-Management.

   1. The Responder peer enter SCTP Established, and the
      Key-Management is provided with the full ordered list of
      Protection Solutions offered in the INIT Chunk.

   2. The Initiator enters SCTP Assocationa Established and the
      Key-Management is triggered to perform the next step.

   3. Key-Management initiated a TLS 1.3 handshake with the the supported
      configuration. Taking supported Cipher-suits in the DTLS Chunk
      implementation into account when creating its TLS Client-Hello
      mesage. The TLS messages are sent per {{tls-user-message}}

   4. Responder receives TLS Client-Hello and generates
      the TLS Server Hello, etc response message(s) for the TLS handshake.
      In case the TLS server in the responder requires the use of the
      retry message an additional message exchange between TLS Client
      and TLS server is needed before one can progress to 5.

   5. Responder uses its the TLS Exporter on the DTLS Connection's
      state to derive the primary client write key and IV
      {{dtls-key-derivation}} and install them into the Chunk
      Protection Operator's Primary Key Context.
      Then it sends the TLS Server's response message(s).

   6. The TLS client receives the TLS server's messages (Server
      Hello etc.)  and can now export both the client and server write
      key for the Primary and Restart Key Contexts, however their usage
      is not yet required and SCTP packets without DTLS chunks are still
      accepted. Then the TLS Client next handshake message is sent.
      This message MUST be protected by the DTLS Chunk using the Primary
      key Context (Client Write key and IV).

   7. The responder's Chunk Protection Operator will receive the SCTP
      packets containing the DTLS chunk protected DTLS messages,
      concluding the main process of the TLS handshake.
      The responder exports the remaining keys and IVs and installs all
      Primary and Restart Server Write Key and IV, as well as restart
      client write key and IV. After that it requires all future SCTP
      Packets to be protected by DTLS Chunk. If any TLS ACK message
      is to be sent, it SHOULD be sent next.
      The initiator's and the reposnder's
      key-management can now inform the ULP that the SCTP association
      is protected and verified and traffic can be sent.

   8. The Initiator's ULP is notified that the SCTP association is
      Established and protected and it may generate user messages.

   If the TLS handshake fails the SCTP association MUST be aborted
   and an ERROR chunk with the Error in Protection error cause, with
   the appropriate extra error causes is generated, the right
   selection of "Error During Protection Handshake" or "Timeout During
   Protection Handshake or Validation".


### Handshake of further TLS connections {#further_tls_connection}

   When the SCTP Association has entered the PROTECTED state, each of
   the endpoint can initiate a TLS handshake for Key-Management when
   necessary.

   The TLS Key-Management will act as a User of SCTP, identified
   with the Key-Management PPID 4242. TLS handshake
   is sent as a SCTP user message according to {{tls-user-message}}.

   If the TLS
   implementation support configuring a MTU larger than the actual IP
   MTU it could be used as SCTP provides reliability and
   fragmentation.

   If the TLS handshake fails the SCTP association MUST generate
   an ERROR chunk with the Error in Protection error cause, with
   extra error causes "Error During Protection Handshake".

~~~~~~~~~~~ aasvg
Initiator                                               Responder
    |                                                       |
    +---------[DTLS CHUNK(DATA(TLS Client Hello))]--------->|
    +<---[DTLS CHUNK(DATA(TLS Server Hello ... Finished))]--+
    +----[DTLS CHUNK(DATA(TLS Certificate ... Finished))]-->|
    +<-------------[DTLS CHUNK(DATA(TLS ACK))]--------------+
    |                                                       |
~~~~~~~~~~~
{: #sctp-TLS-further-connection title="Handshake of further TLS connection" artwork-align="center"}

The {{sctp-TLS-further-connection}} shows a successful
handshake of a further TLS connection. Such connections can
be initiated by any of the peers. Here TLS handshake messages
are transported by means of DATA chunks with the DTLS Chunk
Key-Management Messages PPID inside DTLS Chunks.

## SCTP Association Restart {#sctp-restart}

In order to achieve an Association Restart as described in
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}, a Restart DTLS Key Context
dedicated to Restart MUST exist and be available.  Furthermore, both
peers MUST have safely stored both the current Restart DTLS Key Context.
Here we assume that Restart DTLS Key Context is maintained across
the events leading to SCTP Restart request.

### Installation of initial Restart DTLS Key Context {#init-dtls-restart-context}

As soon as the Association has reached the ESTABLISHED state
and before the Primary DTLS Key context have been installed a
Restart DTLS Key Context MUST be derived and then installed.

It MAY exist a short time gap where the Association has already
been validated state but no Restart DTLS Key Context has been installed
yet. If a SCTP Restart procedure will be initiated during that time,
it will fail and the Association will also fail. However, this is
unlikely as the Restart Init will be sent multiple times following a
exponential back-off timer and in that time the Restart DTLS Key Context is
expected to be in place.

Once installed, no traffic will be sent over the Restart DTLS Key Context
so that both endpoints will have a known DTLS context state, i.e. the
Sequence number and replay window are both just initialized to default
values for the epoch=3.

### Installation of Restart DTLS Key Context for further TLS Connections

As each subsequent Key-Management TLS connection completes the TLS handshake
and the Primary DTLS Key context has been derived and installed also
the Restart DTLS Key context MUST be installed. The closing of the previous
TLS connection MUST NOT be initiated or completed until the Restart
DTLS Key Context is in place.


### SCTP Association Restart Procedure {#sctp-assoc-restart-procedure}

The DTLS in SCTP Association Restart is meant to preserve the security
characteristics.

In order the Association Restart to proceed both Initiator and Responder
MUST use the same Restart DTLS Key Context for COOKIE-ECHO/COOKIE-ACK
handshake, that implies that the Initiator must preserve the Restart
DTLS Key Context and that the Responder MUST NOT change the Restart
DTLS Key Context during the Restart procedure.

~~~~~~~~~~~ aasvg

Initiator                                     Responder
    |                                             | -.
    |                                             |   +-------
    +--------------------(INIT)------------------>|   | Plain
    |<-----------------(INIT-ACK)-----------------+   +-------
    |                                             | -'
    |                                             | -.
    |                                             |   +-------
    +---------[DTLS CHUNK(COOKIE ECHO)]---------->|   | Protected
    |<--------[DTLS CHUNK(COOKIE ACK)]------------+   +-------
    |                                             | -'
    |                                             |
    |                                             | -.
    +-----------[DATA(TLS Client Hello)]--------->|   |
    |<---[DATA(TLS Server Hello ... Finished)]----+   | New TLS
    +----[DATA(TLS Certificate ... Finished)]---->|   | Connection
    |<--------------[DATA(TLS ACK)]---------------+   +-----------
    |                                             | -'
    |                    ...                      | -.
    |                    ...                      |   | Derive new
    |                    ...                      |   | Traffic and
    |                    ...                      |   | Restart
    |                    ...                      |   | DTLS Key
    |                    ...                      |   | Contexts
    |                    ...                      |   +------------
    |                    ...                      | -'
    |                                             | -.
    +-------[DTLS CHUNK(DATA(APP DATA))]--------->|   | APP DATA
    +<-------[DTLS CHUNK(DATA(APP DATA))]---------+   +---------
    |                    ...                      |   |
    |                    ...                      |   |

~~~~~~~~~~~
{: #sctp-assoc-restart-sequence title="SCTP Restart sequence for DTLS in SCTP" artwork-align="center"}

The {{sctp-assoc-restart-sequence}} shows a successful
SCTP Association Restart.

From procedure viewpoint the sequence is the following:

- Initiator sends INIT (VTag=0), Responder replies INIT-ACK
  in Plain Text as specified in {{RFC9260}}

- Initiator sends COOKIE-ECHO using DTLS CHUNK encrypted with the Key
  tied to the Restart DTLS Key Context

- Responder replies with COOKIE-ACK using DTLS CHUNK encrypted with
  the Restart DTLS Key Context

- Initiator and Responder handshake a new TLS connection

- The ULP can resume communication protected
  using the Restart DTLS Key Context.

- New Primary and Restart DTLS Key Context is derived and
  installed using Epoch=3. The new Traffic
  DTLS Key Context is being used for traffic.
  When Primary DTLS Key Context has been installed
  the new Restart DTLS Key Context for epoch=3 is installed.

User Data for any ULP traffic MAY be initiated immediately after
COOKIE-ECHO/COOKIE-ACK handshake using the current Restart DTLS Key Context, that
is even before a new Primary DTLS Key Context or a Restart DTLS Key Context have been
derived.  If a problem occurs before the new Restart DTLS Key Context has been
installed, the Association cannot be Restarted, thus it's RECOMMENDED
the new Restart DTLS Key Context to be installed as early as possible.

Note that, different than the initial Association establishment,
the ULP traffic is permitted immediately after the
COOKIE-ECHO/COOKIE-ACK handshake, the reason is that the
validation has already been performed prior to the restart DTLS Key Context was
created.

# Parallel DTLS Key Contexts {#parallel-dtls}

Rekeying in this specification is implemented by replacing the DTLS Key contexts
getting old with a new one by first creating a new TLS
connection, derive the new DTLS Key contexts, start using them,
then closing the TLS connection and revoking the old DTLS Key contexts.
The epoch of the new DTLS Key context will be equal to the epoch
of the old DTLS Key context + 1.

## Criteria for Rekeying

The criteria for rekeying may vary depending on the ULP requirement on
security properties, chosen cipher suits etc. Therefore it is assumed
that the implementation will be configurable by the ULP to meet its demand.

Likely criteria to impact the need for rekeying through the usage of
new TLS connection are:

   * Time duration since last authentication of the peer

   * Amount of data transferred since last forward secrecy preserving
     rekeying

   * The cipher suit's maximum key usage being reached.


## Procedure for Rekeying

This specification allows up to 2 DTLS Key Context to be active at the same
time for the current SCTP Association.
The following state machine applies.

~~~~~~~~~~~ aasvg
           +---------+
+--------->|  YOUNG  |  There's only one
|          +----+----+  DTLS Key Context until
|               |       aging criteria are met
|               |
|        AGING  |  REMOTE AGING
|               V
|          +---------+
|          |  AGED   |  When in AGED state a new TLS connection is
|          +----+----+  added and deriving new Primary DTLS Key
|               |       Context as well as a new Restart DTLS Key
|       NEW TLS |       Context
|               |
|               |
|               V
|          +---------+
|          |   OLD   |  In OLD state there are 2 active DTLS
|          +----+----+  Key Context. Traffic is switched to the new
|               |       Primary DTLS Key Context
|      SWITCH   |
|               V
|          +---------+
|          |  DRAIN  |  The aged DTLS Key Context
|          +----+----+  is drained before being ready
|               |       to be closed.
|               |
|       DRAINED | TLS close_notify
|               V
|          +---------+
|          |  DEAD   |  In DEAD state the aged
|          +----+----+  DTLS Key Context is closed
|               |
|      REMOVED  |
+---------------+

~~~~~~~~~~~
{: #dtls-rekeying-state-diagram title="State Diagram for Rekeying"}

Trigger for rekeying can either be a local AGING event, triggered by
meeting the criteria for rekeying, or a REMOTE
AGING event, triggered by receiving a TLS Connection handshake.
In such case a new TLS connection shall be added
according to {{add-tls-connection}}.

As soon as the new TLS connection completes handshaking, and the
Primary and Restart DTLS Key Contexts have been derived and installed,
the protection of the SCTP packets is moved from the old Primary DTLS
Key Context, then the procedure for closing the old TLS Primary DTLS Key Context
is initiated, see {{remove-tls-connection}}.
When Traffic has been moved to the new DTLS Key Context, the TLS connection
is released.


## Race Condition in Rekeying

A race condition may happen when both peers experience local AGING event at
the same time and start creation of a new TLS connection.

The race condition is solved as specified in {{add-tls-connection}}.


# Security Considerations

## General

The security considerations given in {{RFC8446}}, {{RFC8446}}, {{RFC6347}}, and
{{RFC9260}} also apply to this document. BCP 195 {{RFC9325}}
{{RFC8996}} provides recommendations and requirements for improving
the security of deployed services that use DTLS. BCP 195 MUST be
followed which implies that DTLS 1.0 MUST NOT be supported and are
therefore not defined.

## Privacy Considerations

Although TLS for DTLS in SCTP provides privacy for the actual user message as
well as almost all chunks, some fields are not confidentiality
protected.  In addition to the DTLS record header, the SCTP common
header and the DTLS chunk header are not confidentiality
protected. An attacker can correlate TLS connections over the same
SCTP association using the SCTP common header.

To provide identity protection it is RECOMMENDED that TLS for DTLS in SCTP is
used with certificate-based authentication in TLS 1.3 {{RFC8446}} and
to not reuse tickets.  TLS 1.3 with external PSK
authentication does not provide identity protection.

By mandating ephemeral key exchange and cipher suites with
confidentiality TLS for DTLS in SCTP effectively mitigate many forms of
passive pervasive monitoring.  By recommending implementations to
frequently set up new TLS connections with (EC)DHE force attackers to
do dynamic key exfiltration and limits the amount of compromised data
due to key compromise.

# IANA Consideration

This document requests the following registration.

## SCTP Protection Solution Identifier  {#sec-iana-psi}

IANA is requested to assign one SCTP Protection Solution Identifier to
identify the key-management defined in this document.

| Identifier | Solution Name | Reference | Contact |
| 4096 | TLS for DTLS in SCTP Handshake | RFC-TBD | Draft Authors |
{: #iana-psi title="SCTP Protection Solution Indicators" cols="r l l l"}

## TLS Exporter Labels {#iana-export-label}

IANA is requested to register the following values in the TLS Exporter
Label Registry {{RFC5705}} with Reference RFC-TO-BE and empty Comment. The registry was at the time of writing located
at: https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#exporter-labels

| Value | DTLS-OK | Recommended |
| EXPORTER_DTLS_IN_SCTP_PRIMARY_CLIENT_KEY | Y | N |
| EXPORTER_DTLS_IN_SCTP_PRIMARY_CLIENT_IV | Y | N |
| EXPORTER_DTLS_IN_SCTP_PRIMARY_SERVER_KEY | Y | N |
| EXPORTER_DTLS_IN_SCTP_PRIMARY_SERVER_IV | Y | N |
| EXPORTER_DTLS_IN_SCTP_RESTART_CLIENT_KEY | Y | N |
| EXPORTER_DTLS_IN_SCTP_RESTART_CLIENT_IV | Y | N |
| EXPORTER_DTLS_IN_SCTP_RESTART_SERVER_KEY | Y | N |
| EXPORTER_DTLS_IN_SCTP_RESTART_SERVER_IV | Y | N |
{: #iana-tls-exporter title="TLS Exporter Labels" cols="l l l"}

