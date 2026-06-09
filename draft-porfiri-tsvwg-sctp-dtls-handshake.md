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
  RFC6347:
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
is used as a key-management method for the SCTP DTLS Chunk mechanism.
It specifies how a TLS handshake establishes the initial security
context for an SCTP association and how subsequent TLS handshakes
provide key updates and re-authentication. The goal is to enable
authenticated and confidential communication over SCTP using the
DTLS Chunk, leveraging standardized TLS 1.3 features for key
management and rekeying.

--- middle

# Introduction {#introduction}

The Stream Control Transmission Protocol (SCTP) {{RFC9260}} is a
transport protocol designed to support message-oriented communication
with features such as multi-streaming and multi-homing.  In many
deployments, particularly telecommunication networks and WebRTC data
channels, it is essential to provide confidentiality, integrity, and
peer authentication for SCTP traffic.

{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}} defines a mechanism for
securing SCTP by encapsulating SCTP chunks within DTLS 1.3 records at
the chunk level.  That specification defines the DTLS chunk format,
negotiation procedures, and an abstract API for key management, but
delegates the actual key management to external methods identified by
a DTLS Key Management Identifier.

This document defines one such method: it uses TLS 1.3 {{RFC8446}}
handshakes carried as SCTP user messages to perform mutual
authentication and derive keying material for the DTLS Chunk
Protection Operator.  The combination of the SCTP DTLS Chunk and the
key-management defined in this document is referred to as "TLS for
DTLS in SCTP".

The key advantages of this approach are:

* It requires no extensions to TLS 1.3 to support long-lived sessions.

* It is based on TLS 1.3 rather than DTLS 1.3, leveraging widely
  available TLS implementations.

## Conventions

{::boilerplate bcp14}

In this document, \|\| denotes concatenation of byte sequences.

## Terminology {#terminology}

This document uses the following terms:

Association:
: An SCTP association.

Connection:
: A TLS 1.3 connection used for key management.

DTLS Key Context (DKC):
: The keying material (record payload key, sequence number key, and
  IV) for both send and receive directions, together with the replay
  window and last used sequence number.  Each DKC is identified by a
  tuple of (SCTP Association, restart indicator, DTLS epoch).

Initiator:
: The endpoint assigned the client role during SCTP association
  establishment.  This corresponds to the "client" role (C bit) in
  the DTLS Key Management Parameter of
  {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

Primary DKC:
: A DTLS Key Context used to protect regular SCTP association traffic.

Responder:
: The endpoint assigned the server role during SCTP association
  establishment.  This corresponds to the "server" role (S bit) in
  the DTLS Key Management Parameter of
  {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

Restart DKC:
: A DTLS Key Context reserved exclusively for the SCTP association
  restart procedure.

## Abbreviations

AEAD:
: Authenticated Encryption with Associated Data

DKC:
: DTLS Key Context

DTLS:
: Datagram Transport Layer Security

PPID:
: Payload Protocol Identifier

SCTP:
: Stream Control Transmission Protocol

TLS:
: Transport Layer Security

ULP:
: Upper Layer Protocol


# Overview {#overview}

This section provides an informational overview of TLS for DTLS in
SCTP.  Normative procedures are specified in {{procedures}}.

## Architecture {#architecture}

~~~~~~~~~~~ aasvg
+---------------+ +-------------------------------+
|      ULP      | |            TLS 1.3            |
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
|               | |    +----------+----------+  | |
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
{: #overview-layering title="Architecture" artwork-align="center"}

Application data is never transmitted in TLS application_data records.
Instead, application data is sent via SCTP DATA chunks protected by
the DTLS Chunk Protection Operator.  TLS 1.3 is used solely for key
management: performing handshakes, deriving keys via the TLS Exporter,
and then closing the TLS connection.

## Protocol Flow Summary

The protocol operates in three phases:

1. **Initial Establishment:** After SCTP association setup (with
   DTLS Key Management Parameter negotiation), a TLS 1.3 handshake
   derives the initial Primary and Restart DKCs.  Protection is then
   enforced.

2. **Rekeying:** Either endpoint initiates a new TLS 1.3 handshake
   to derive fresh DKCs for the next epoch, providing forward secrecy
   and re-authentication.  Old DKCs are removed after draining.

3. **SCTP Restart:** The pre-established Restart DKC protects the
   COOKIE ECHO/COOKIE ACK exchange, followed by a new TLS handshake
   to establish fresh Primary and Restart DKCs.


# TLS Configuration Requirements {#tls-config}

## TLS Version

This document defines the usage of TLS 1.3 {{RFC8446}}.  Earlier
versions of TLS MUST NOT be used.  Only one version of TLS MUST be
used during the lifetime of an SCTP Association.

## Cipher Suite Constraints

Parameters not marked as "Y" in the "Recommended" column of TLS
registries are NOT RECOMMENDED to support.  Non-AEAD cipher suites
or cipher suites without confidentiality MUST NOT be supported.
Cipher suites and parameters that do not provide ephemeral
key-exchange MUST NOT be supported.

The cipher suites negotiated in the key-management TLS connection
MUST only include those supported by the DTLS Chunk Protection
Operator.  The DTLS Chunk provides an API to query supported cipher
suites (see Section 7.3 of {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}).

## Authentication and Identity {#tls-auth}

TLS for DTLS in SCTP MUST be mutually authenticated.  It is
RECOMMENDED to use certificate-based authentication.

When certificates are used, the application is responsible for
certificate policies, certificate chain validation, and identity
authentication.  The application defines what the identity is and
how it is encoded.  Guidance on server certificate validation can be
found in {{RFC9525}}.

All security decisions MUST be based on the peer's authenticated
identity, not on its transport layer identity.  Since SCTP
associations can use multiple IP addresses per endpoint, DTLS records
may arrive from different source IP addresses than those originally
authenticated.

Clients and servers MUST NOT accept a change of identity during the
setup of a new TLS connection, but MAY accept negotiation of stronger
algorithms and security parameters.

## Rekeying Policy {#rekey-policy}

Implementations MUST have policies for how often to set up new TLS
connections with ephemeral key exchange.  Implementations SHOULD
rekey at least every hour and every 100 GB of data, which is a common
policy for IPsec {{ANSSI-DAT-NT-003}}.

Implementations MUST set up a new TLS connection using a full
handshake before any of the certificates expire.

The PSK key exchange mode psk_ke MUST NOT be used as it does not
provide ephemeral key exchange.  TLS Key Update MUST NOT be used.

TLS 1.3 tickets MAY be used for resumption (valid up to seven days).
Resumption can be used to chain the connections, increasing security
by forcing an adversary to break them in sequence {{KTH-NCSA}}.

The endpoints MUST limit the number of simultaneous TLS connections
to one.


# TLS Message Transport {#tls-user-message}

TLS records and control messages for key-management are sent as SCTP
user messages using reliable in-order delivery on stream 0 with the
DTLS Key Management Messages PPID (4242)
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.

Each SCTP user message uses the format defined in
{{sctp-dtls-user-message}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|T|   Epoch     |                                               |
+-+-+-+-+-+-+-+-+            Payload                            |
|                                                               |
|                               +-------------------------------+
|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-dtls-user-message title="Key Management User Message Structure" artwork-align="center"}

T (Message Type): 1 bit
: Indicates the type of payload carried in this user message.
  A value of 0 indicates that the payload contains TLS records.
  A value of 1 indicates that the payload is a control message
  (see {{control-messages}}).

Epoch: 7 bits
: The 7 lowest bits of the DTLS Key Context epoch that this
  message corresponds to — i.e., the DKC that will be created from
  this handshake, or that already exists.  The receiver uses this
  field to associate incoming data with the correct key-management
  session.

Payload: variable length
: When T=0, one or more complete TLS records.  When T=1, a control
  message as defined in {{control-messages}}.

## Control Messages {#control-messages}

When the T bit is set to 1, the payload of the user message is a
control message with the following format:

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Ctrl Type    |         Control Data (variable)               |
+-+-+-+-+-+-+-+-+                                               |
~~~~~~~~~~~
{: #control-message-format title="Control Message Format" artwork-align="center"}

Ctrl Type: 8 bits
: Identifies the control message type.

Control Data: variable length
: Type-specific data.  May be empty.

The following control message type is defined:

| Ctrl Type | Name                    | Description                        |
|-----------|-------------------------|------------------------------------|
| 0x01      | Protection Established  | Signals that DTLS chunk protection has been enforced |
{: #control-message-types title="Control Message Types"}

### Protection Established {#protection-established}

The Protection Established control message (Ctrl Type = 0x01) is sent
by the Responder to the Initiator after the Responder has installed
all keys and enforced DTLS chunk protection.  This message carries no
Control Data (the payload following the Ctrl Type byte is empty).

Upon receiving this message, the Initiator enforces DTLS chunk
protection and informs the ULP that the association is protected.


# Key Derivation {#dtls-key-derivation}

## Role Determination {#role-determination}

Role determination and method selection follow the procedure defined
in Section 5.1 of {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}.  After
the SCTP association is established, the key-management function
retrieves from the SCTP stack's DTLS chunk API the assigned role
(Initiator/client or Responder/server), the selected DTLS Key
Management Method, and the downgrade prevention data (both endpoints'
DTLS Key Management Parameters) used as input to key derivation.

## Exporter Context {#exporter-context}

DTLS Key Contexts are derived using the TLS Exporter as defined in
Section 7.5 of {{RFC8446}}.  The exporter context is constructed as
the concatenation of the following fields:

| Field | Length | Value |
|-------|--------|-------|
| Direction | 1 byte | 0x00 = Client, 0x01 = Server |
| Key role | 1 byte | 0x00 = primary/traffic, 0x01 = restart |
| Key type | 1 byte | 0x00 = Key, 0x01 = SN_KEY, 0x02 = IV |
| Initiator KM Param | variable | DTLS Key Management Parameter sent by the Initiator |
| Responder KM Param | variable | DTLS Key Management Parameter sent by the Responder |

Each DTLS Key Management Parameter (Section 4.1 of
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}) is included as the
sequence of bytes sent on the wire, including the parameter header
and excluding padding.

This construction ensures that any modification to the DTLS Key
Management Parameter during the SCTP handshake (a downgrade attack)
results in mismatched keys and association failure.

## Exporter Labels {#exporter-labels}

A single TLS Exporter label is used to derive all keying material:

    EXPORTER_TLS_FOR_DTLS_IN_SCTP

The specific key material (direction, role, and type) is
differentiated by the exporter context ({{exporter-context}}).  Each
combination of Direction, Key role, and Key type values produces a
distinct export, yielding 12 values in total (2 directions × 2 roles
× 3 types).

The Initiator (TLS client) installs exports with Direction=Client as
its write keys and Direction=Server as its read keys.  The Responder
does the reverse.

The length of exported material depends on the negotiated cipher
suite.

## DKC Installation {#dkc-installation}

Each successful TLS handshake produces two DKCs:

* A Primary DKC for regular SCTP traffic.
* A Restart DKC for the SCTP restart procedure.

The first DKC established for any SCTP association MUST use DTLS
epoch 3.  Each subsequent Primary DKC uses the next consecutive
epoch.  After an SCTP restart, the epoch resets to 3.

The Restart DKC MUST be maintained in a well-defined state
(initialized but never used for regular traffic) so that both
endpoints have a consistent view of sequence numbers and replay
window.


# Procedures {#procedures}

## Initial Establishment {#initial-establishment}

~~~~~~~~~~~ aasvg

Initiator                                            Responder
    |                                                    |
 1. +------------------------[INIT]--------------------->|
    |<---------------------[INIT-ACK]--------------------+
    +--------------------[COOKIE ECHO]------------------>| 2.
 3. |<--------------------[COOKIE ACK]-------------------+
    |                                                    |
    |  Key Manager                           Key Manager |
    |    |                                          |    |
 4. +--->| TLS START                      TLS START |<---+
    |    |                                          |    |
    |    +---------[DATA(TLS Client Hello)]------->| 5. |
    |    |<-[DATA(TLS Server Hello ... Finished)]--+ 6. |
    | 7. +--[DATA(TLS Certificate ... Finished)]-->| 8. |
    |    |<--[DATA(Protection Established)]--------+    |
 9. |    |                                          |    |
    |                                                    | -.
10. +------------[DTLS CHUNK(DATA(APP DATA))]----------->|   | APP DATA
    +<-----------[DTLS CHUNK(DATA(APP DATA))]------------+   +---------
    |                         ...                        |   |

~~~~~~~~~~~
{: #initial-establishment-diagram title="Initial Establishment" artwork-align="center"}

The procedure is as follows:

1. The Initiator sends INIT containing the DTLS Key Management
   Parameter (Section 4.1 of {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}})
   with this method's identifier (see {{sec-iana-psi}}) in its
   preference-ordered list.

2. The Responder enters ESTABLISHED state.  It retrieves the agreed
   DTLS Key Management Method and role from the SCTP stack (e.g.,
   using the "Get Agreed DTLS Key Management Method and Role" API
   defined in Section 7.2 of {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}})
   and verifies that the selected method matches the one defined in
   this document (see {{sec-iana-psi}}) and that the assigned role
   is server (Responder).

3. The Initiator enters ESTABLISHED state.  It performs the same
   retrieval and verification as the Responder, confirming that the
   assigned role is client (Initiator).  Note: if the role assignment
   differs from the SCTP-level Initiator/Responder (e.g., due to
   tie-breaking), the endpoint assigned the client role is the one
   that performs
   step 4.

4. The Initiator's key manager starts a TLS 1.3 handshake, limiting
   offered cipher suites to those supported by the DTLS Chunk
   Protection Operator.  TLS messages are sent per
   {{tls-user-message}}.

5. The Responder receives TLS ClientHello and generates its response.
   If a HelloRetryRequest is needed, an additional round-trip occurs
   before proceeding.

6. The Responder uses the TLS Exporter to derive the client
   (Initiator) key material for both the Primary and Restart DKCs
   and installs it as its read (receive) key material, per
   {{dtls-key-derivation}}.  It then sends its TLS response messages.

7. The Initiator receives the TLS server messages, exports and
   installs all Primary and Restart DKC keys: the client key material
   as its write (send) key and the server key material as its read
   (receive) key.  It then sends its TLS
   Certificate/CertificateVerify/Finished, protected by the DTLS
   Chunk using the new Primary DKC.

8. The Responder decrypts the DTLS-chunk-protected TLS messages,
   completes the handshake, exports and installs the server key
   material for both the Primary and Restart DKCs as its write
   (send) key.  It enforces DTLS chunk protection for all future
   packets, informs the ULP that the association is protected, and
   sends a Protection Established control message
   ({{protection-established}}) to the Initiator.

9. The Initiator receives the Protection Established control message,
   enforces DTLS chunk protection for all future packets, and
   informs the ULP that the association is protected.

10. Application traffic can begin.

If the TLS handshake fails, the SCTP association MUST be aborted.

After key installation, the TLS connection SHOULD be closed promptly.

## Rekeying {#rekeying}

### Triggering Criteria

Rekeying is triggered by implementation-configurable criteria,
including:

* Time elapsed since last peer authentication.
* Volume of data transferred since last forward-secrecy rekeying.
* Approaching the cipher suite's AEAD usage limits.

### Procedure {#rekey-procedure}

~~~~~~~~~~~ aasvg

Initiator                                            Responder
    |                                                    |
    |  (traffic continues using epoch N DKC)             |
    |                                                    |
    |  Key Manager                           Key Manager |
    |    |                                          |    |
 1. |    +---------[DATA(TLS Client Hello)]------->|    |
 2. |    |<-[DATA(TLS Server Hello ... Finished)]--+    |
 3. |    +--[DATA(TLS Certificate ... Finished)]-->| 4. |
    |                                                    |
    |  (traffic transitions to epoch N+1 DKC)            |
    |                                                    |
    |  (after draining, remove epoch N DKC)              |
    |                                                    |

~~~~~~~~~~~
{: #rekey-diagram title="Rekeying Procedure" artwork-align="center"}

Either endpoint may initiate rekeying.  The procedure is as follows:

1. The Initiator sends a TLS ClientHello.  TLS messages are carried
   inside DTLS chunks (the association is already protected).

2. The Responder receives and processes the ClientHello.  It exports
   the client key material for both the Primary and Restart DKCs and
   installs it as its read (receive) key.  It then sends its TLS
   ServerHello through Finished messages to the Initiator.

3. The Initiator receives the TLS server messages and installs both
   sets of keys: the client key material as its write (send) key and
   the server key material as its read (receive) key, for both the
   Primary and Restart DKCs.  It then sends its TLS
   Certificate/CertificateVerify/Finished.  The Initiator starts the
   drain timer to remove the old (epoch N) DKC.

4. The Responder receives and verifies the Finished message.  It
   exports and installs the server key material for both the Primary
   and Restart DKCs as its write (send) key.  The Responder starts
   the drain timer to remove the old (epoch N) DKC.

The new DKCs use epoch N+1 (where N is the current epoch).  Both old
(epoch N) and new (epoch N+1) DKCs coexist temporarily.  After no
longer than 120 seconds (one Maximum Segment Lifetime), the old DKCs
MUST be removed.

All rekeying MUST use ephemeral key exchange.  TLS Key Update MUST
NOT be used.

### Simultaneous Rekey Resolution

As either endpoint can initiate a TLS handshake at the same time,
either endpoint may receive a TLS ClientHello when it has already
sent its own.  In this case, the ClientHello from the Initiator
(the endpoint with the client role) SHALL be processed, and the
other SHALL be dropped.

### Key Transition State Machine

This specification allows up to 2 Primary DKCs to be active at the
same time.  The following state machine governs the transition:

~~~~~~~~~~~ aasvg

           +---------+
           |  INIT   |  Association started
           +----+----+  No TLS H/S yet
                |
                | 1. TLS initial H/S
                |    completed
                V
           +---------+
+--------->|  YOUNG  |
+ +------->|         +--------------------+
| |        +----+----+                    |
| |             |                         |
| |             | 2. Client Hello         | 3. Aging event
| |             |    from peer            |
| |             V                         V
| |        +---------+  4. Client H +---------+  5. TLS H/S
| |        | REMOTE  |    tie-break |  LOCAL  |    Timeout
| |        |  OLD    |<-------------+  AGED   +-----+
| |        +----+----+              +----+----+     |
| | 7. Flush    |                        |          |
| +-------------+                        |          |
|                     6. Server Hello    |          |
|               +------------------------+          |
|               |                                   |
|               V                                   V
|          +---------+                         +---------+
|          |  LOCAL  |                         |  ABORT  |
|          |   OLD   |                         |         |
|          +-----+---+                         +---------+
|   8. Flush     |
+----------------+

~~~~~~~~~~~
{: #dtls-rekeying-state-diagram title="State Diagram for Rekeying" artwork-align="center"}

INIT:
: Initial state.  Only event 1 (TLS handshake completed) transitions
  to YOUNG.

YOUNG:
: Only the Current DKC is populated.  Event 2 (peer ClientHello)
  transitions to REMOTE OLD; event 3 (aging) transitions to LOCAL
  AGED.

LOCAL AGED:
: A supervision timer runs.  Event 5 (timeout) causes ABORT.  Event
  4 (peer ClientHello with tie-break yielding server role) transitions
  to REMOTE OLD.  Event 6 (ServerHello received) transitions to LOCAL
  OLD.

REMOTE OLD:
: Both Old and Current DKCs exist.  Old DKC is used for sending until
  the first packet encrypted with Current DKC is received from the
  peer.  Event 7 (flush timer expires) transitions to YOUNG.

LOCAL OLD:
: Both Old and Current DKCs exist.  Current DKC is used for sending.
  Event 8 (flush timer expires) transitions to YOUNG.

In REMOTE OLD and LOCAL OLD, if a new ClientHello or Aging event
arrives, the flushing timer is cleared and behavior is as in YOUNG.


## SCTP Association Restart {#sctp-restart}

### Prerequisites

For protected SCTP restart to succeed:

* Both endpoints MUST have a valid Restart DKC.
* The Restart DKC MUST be stored securely and persistently to
  survive crash events (see
  Section 10.4 of {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}).
* Both endpoints MUST have indicated restart support (R bit) in the
  DTLS Key Management Parameter.

### Restart Procedure {#restart-procedure}

~~~~~~~~~~~ aasvg

Initiator                                            Responder
    |                                                    |
 1. |  (install restart keys from storage)               |
    |                                                    | -.
 2. +------------------------(INIT)--------------------->|   | Plain
    |<---------------------(INIT-ACK)--------------------+   +-------
    |                                                    | -'
    |                                                    | -.
 3. +-------------[DTLS CHUNK(COOKIE ECHO)]------------->|   | Protected
 4. |<------------[DTLS CHUNK(COOKIE ACK)]---------------+   +----------
    |                                                    | -'
    |                                                    |
    |  (TLS handshake for new keys, steps 6-11)          |
    |                                                    |
12. +------------[DTLS CHUNK(DATA(APP DATA))]----------->|   APP DATA
    +<-----------[DTLS CHUNK(DATA(APP DATA))]------------+
    |                                                    |

~~~~~~~~~~~
{: #restart-diagram title="SCTP Restart Procedure" artwork-align="center"}

1. The restarting endpoint (Initiator) retrieves the restart key
   material from persistent secure storage and installs the Restart
   DKC for both send and receive directions.

2. Initiator sends INIT (VTag=0), Responder replies INIT-ACK in
   plain text per {{RFC9260}}.  Both include the DTLS Key Management
   Parameter with the same method list but a new random Tie Breaker.

3. Initiator sends COOKIE ECHO in a DTLS chunk protected with the
   Restart DKC (R bit set).

4. Responder replies COOKIE ACK in a DTLS chunk protected with the
   Restart DKC.  ULP traffic MAY begin immediately using the Restart
   DKC.

5. Both endpoints have a new established association.  Each endpoint
   immediately enforces DTLS chunk protection (using the Restart DKC)
   and then retrieves the agreed DTLS Key Management Method and role
   from the SCTP stack (e.g., using the "Get Agreed DTLS Key
   Management Method and Role" API defined in Section 7.2 of
   {{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}}) and verifies that the
   selected method matches the one defined in this document (see
   {{sec-iana-psi}}) and that the assigned role is as expected.  The
   ULP MAY be informed that the association is protected at this
   point.  The endpoint with the client role initiates step 6.

6. The Initiator (client) starts a TLS 1.3 handshake, limiting
   offered cipher suites to those supported by the DTLS Chunk
   Protection Operator.  TLS messages are sent per
   {{tls-user-message}}, protected by the Restart DKC.

7. The Responder receives the TLS ClientHello and generates its
   response.  It exports the client key material for the Primary DKC
   and installs it as its read (receive) key.  It then sends its TLS
   response messages.

8. The Initiator receives the TLS server messages, exports and
   installs the Primary DKC keys: the client key material as its
   write (send) key and the server key material as its read (receive)
   key.  It then sends its TLS Certificate/CertificateVerify/Finished,
   protected by the DTLS Chunk using the new Primary DKC.

9. The Responder decrypts the DTLS-chunk-protected TLS messages,
   completes the handshake, exports and installs the server key
   material for the Primary DKC as its write (send) key.  It
   enforces DTLS chunk protection using the new Primary DKC and
   sends a Protection Established control message
   ({{protection-established}}) to the Initiator.

10. The Initiator receives the Protection Established control message
    and enforces DTLS chunk protection using the new Primary DKC.

11. Both endpoints export and install the new Restart DKC key material
    (both send and receive directions).  The old Restart DKC is then
    removed and the new Restart DKC is committed to persistent secure
    storage.

After restart, the new Primary DKC MUST use epoch 3 (the epoch
resets).

The Responder MUST NOT change the Restart DKC during the restart
procedure.  After the new Restart DKC is installed, the old one is
removed.

It is RECOMMENDED to complete the TLS handshake and install new DKCs
as soon as possible after restart, to minimize the window during
which no Restart DKC is available for a subsequent restart.

Note: There MAY exist a short time gap after association establishment
where no Restart DKC is yet installed.  If an SCTP restart is
initiated during that time, it will fail.  However, this is unlikely
as the restarting endpoint sends INIT multiple times with exponential
back-off.


# Error Handling {#error-handling}

TLS has its own error reporting via TLS alert messages.  When a TLS
handshake error occurs, the TLS alert is sent in an SCTP user message
(see {{tls-user-message}}) with the DTLS Key Management Messages PPID
(4242).

If a TLS handshake fails during initial establishment, the SCTP
association MUST be aborted.

If a TLS handshake fails during rekeying, and the current DKC has not
yet reached its usage limits, the implementation SHOULD retry the
handshake.  If retry is not possible or the current DKC is aged
beyond policy limits, the association MUST be aborted.


# Security Considerations {#security-considerations}

## General

The security considerations given in {{RFC8446}}, {{RFC9147}}, and
{{RFC9260}} also apply to this document.  BCP 195 {{RFC9325}}
{{RFC8996}} provides recommendations and requirements for improving
the security of deployed services that use TLS.  BCP 195 MUST be
followed.

## Privacy Considerations

Although TLS for DTLS in SCTP provides privacy for user messages and
almost all SCTP chunks, the SCTP common header, DTLS chunk header,
and DTLS record header are not confidentiality protected.  An
attacker can correlate TLS connections over the same SCTP association
using the SCTP common header.

To provide identity protection, it is RECOMMENDED to use
certificate-based authentication in TLS 1.3 and to not reuse tickets.
TLS 1.3 with external PSK authentication does not provide identity
protection.

By mandating ephemeral key exchange and cipher suites with
confidentiality, TLS for DTLS in SCTP effectively mitigates many
forms of passive pervasive monitoring.  Frequent rekeying forces
attackers to perform dynamic key exfiltration and limits the amount
of compromised data due to key compromise.


# IANA Considerations {#iana-considerations}

## DTLS Key Management Method Identifier {#sec-iana-psi}

IANA is requested to assign one DTLS Key Management Method Identifier
in the "DTLS Key Management Method" registry defined by
{{I-D.draft-ietf-tsvwg-sctp-dtls-chunk}} to identify the
key-management method defined in this document.

| Identifier | Key Management Method Name     | Reference | Contact       |
| 192        | TLS for DTLS in SCTP           | RFC-TBD   | Draft Authors |
{: #iana-psi title="DTLS Key Management Method Identifier" cols="r l l l"}

## TLS Exporter Labels {#iana-export-label}

IANA is requested to register the following value in the TLS
Exporter Label Registry {{RFC5705}} with Reference RFC-TBD and empty
Comment.

| Value | DTLS-OK | Recommended |
| EXPORTER_TLS_FOR_DTLS_IN_SCTP | Y | N |
{: #iana-tls-exporter title="TLS Exporter Label" cols="l l l"}
