---
docname: draft-porfiri-tsvwg-sctp-dtls-handshake-latest
title: Transport Layer Security (TLS)-Based Key Management for the Stream Control Transmission Protocol (SCTP) DTLS Chunk
abbrev: TLS Key Management for SCTP DTLS Chunk
cat: std
ipr: trust200902
wg: TSVWG
area: Transport
submissiontype: IETF

venue:
  group: Transport Area Working Group (tsvwg)
  mail: tsvwg@ietf.org
  github: teiclap/draft-porfiri-tsvwg-sctp-dtls-handshake

author:
-
   ins: C. Porfiri
   name: Claudio Porfiri
   org: Ericsson
   email: claudio.porfiri@ericsson.com
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
  RFC9525:
  RFC9149:

  ANSSI-DAT-NT-003:
    target: <https://messervices.cyber.gouv.fr/documents-guides/NT_IPsec_EN.pdf>
    title: Recommendations for securing networks with IPsec
    seriesinfo:
      ANSSI Technical Report DAT-NT-003
    author:
      -
        org: Agence nationale de la sécurité des systèmes d'information
    date: August 2015

  KTH-NCSA:
    target: "https://kth.diva-portal.org/smash/get/diva2:1902626/FULLTEXT01.pdf"
    title: "On factoring integers, and computing discrete logarithms and orders, quantumly"
    seriesinfo:
      "KTH, School of Electrical Engineering and Computer Science (EECS), Computer Science, Theoretical Computer Science, TCS. Swedish NCSA, Swedish Armed Forces."
    author:
      -
       ins: M. Ekerå
       name: Martin Ekerå
    date: October 2024

normative:
  RFC5705:
  RFC8126:
  RFC8447:
  RFC8996:
  RFC9147:
  RFC9260:
  RFC9325:
  RFC9846:
  I-D.ietf-tsvwg-sctp-dtls-chunk:


--- abstract

This document defines how Transport Layer Security (TLS) 1.3 is used as
a key management method for the Datagram Transport Layer Security
(DTLS) Chunk of the Stream Control Transmission Protocol (SCTP).
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
with features such as multiple streams for messages and multi-homing.
In many deployments, particularly telecommunication networks, it is
essential to provide confidentiality, integrity, and peer
authentication for SCTP traffic.

{{I-D.ietf-tsvwg-sctp-dtls-chunk}} defines a mechanism for
securing SCTP by encapsulating SCTP chunks within DTLS 1.3 records at
the chunk level.  That specification defines the DTLS chunk format,
negotiation procedures, and an abstract API for key management, but
delegates the actual key management to external methods identified by
a DTLS Key Management Identifier.

This document defines one such method: it uses TLS 1.3 {{RFC9846}}
handshakes carried as SCTP user messages to perform mutual
authentication and derive keying material for the DTLS Chunk Protection
Operator.  The combination of the SCTP DTLS Chunk and the key management
defined in this document is referred to as "DTLS in SCTP".

The key advantages of this approach are:

* It requires no extensions to TLS 1.3 to support long-lived sessions.

* It is based on TLS 1.3 rather than DTLS 1.3, leveraging widely
  available TLS implementations.

## Conventions

{::boilerplate bcp14}

## Terminology {#terminology}

This document uses the following terms:

Association:
: An SCTP association.

Client:
: The endpoint that has the key management client role. This
  corresponds to the "client" role (C bit) in the DTLS Key Management
  Parameter of {{I-D.ietf-tsvwg-sctp-dtls-chunk}}.

Connection:
: A TLS 1.3 connection used for key management.

Drain Timer:
: A timer that each key manager starts after switching to a new Primary
  DKC.  While it runs, the previous Primary DKC is kept to process
  packets still in flight.  When it expires, the previous Primary DKC is
  removed and the new Restart DKC replaces the installed one (see
  {{drain-timer}}).

DTLS Key Context (DKC):
: The keying material (record payload key, sequence number key, and
  IV) for both send and receive directions, together with the replay
  window and last used sequence number.  Each DKC is identified by a
  tuple of (SCTP Association, restart indicator, DTLS epoch).

Epoch:
: The DTLS epoch {{RFC9147}} that, together with the restart
  indicator, identifies a DKC.  The first DKC of an SCTP association
  uses epoch 3, and each rekeying procedure uses the next epoch.  After
  an SCTP restart, the epoch resets to 3 (see {{dkc-installation}}).

Initiator:
: The endpoint initiating the SCTP association. In case of simultaneous
  open, both SCTP endpoints may have started as Initiator.

Key Manager:
: The key management function of an endpoint.  It drives the TLS
  connection through the TLS user API, exchanges key management
  messages with its peer, and installs the DKCs.  The client key
  manager is the key manager of the Client and acts as the TLS client;
  the server key manager is the key manager of the Server and acts as
  the TLS server.

Primary DKC:
: A DTLS Key Context used to protect regular SCTP association traffic.

Responder:
: The endpoint that receives the INIT chunk from the Initiator and
  replies with an INIT ACK chunk.

Restart DKC:
: A DTLS Key Context reserved exclusively for the SCTP association
  restart procedure.

Server:
: The endpoint taking the key management server role. This corresponds
  to the "server" role (S bit) in the DTLS Key Management Parameter of
  {{I-D.ietf-tsvwg-sctp-dtls-chunk}}.

## Abbreviations

AEAD:
: Authenticated Encryption with Associated Data

DKC:
: DTLS Key Context

DTLS:
: Datagram Transport Layer Security

IV:
: Initialization Vector

KM:
: Key Manager

PPID:
: Payload Protocol Identifier

PQC:
: Post-Quantum Cryptography

PSK:
: Pre-Shared Key

RKI:
: Read Key Installed (control message)

RX:
: Receive

SCTP:
: Stream Control Transmission Protocol

TLS:
: Transport Layer Security

TX:
: Transmit

ULP:
: Upper Layer Protocol


# Overview {#overview}

This section provides an informational overview of DTLS in
SCTP.  Normative procedures are specified in {{procedures}}.

## Architecture {#architecture}

~~~~~~~~~~~ aasvg
+---------------+ +----------------------------------+
|      ULP      | |        Key Management Method     |
|               | | +----------------------------+   |
|               | | |             TLS 1.3        |   |
|               | | |    +---------------------+ |   |
|               | | | +->+    Key Exporter     +-+-+ |
|               | | | |  +---------------------+ | | |
|               | | | |                          | | |
|               | | | |  +---------------------+ | | |
|               | | | +--+    Key Management   + | | |
|               | | | |  +----------+----------+ | | |
|               | | | |             |            | | |
|               | | | | ContentType |            | | |
|               | | | |  +----------+----------+ | | |
|               | | | +->|     TLS Record      | | | |
|               | | |    | Protection Operator | | | |
|               | | |    +----------+----------+ | | |
|               | | +----------------------------+ | |
|               | |                 ^              | |
+-------+-------+ +-----------------+--------------+-+
        ^            ^              |              |
        |            | PPID=4243    | PPID=4242    |
        +--+---------+--------------+              | keys
      PPID |                                       |
           V                                       V
+--------------------------------------------------+-+
|                      +---------------------+     | |
|        SCTP          |     DTLS Chunk      |<----+ |
|                      | Protection Operator |       |
|                      +---------------------+       |
+----------------------------------------------------+
~~~~~~~~~~~
{: #overview-layering title="Architecture" artwork-align="center"}

As shown in {{overview-layering}}, the Key Management Method is layered
above SCTP alongside the Upper Layer Protocol (ULP), while the DTLS
Chunk Protection Operator sits within SCTP and performs the actual
protection of SCTP packets by encrypting and decrypting them and
performing integrity verification.

The Key Management Method instantiates a TLS 1.3 connection that is used
solely for mutual authentication and key establishment; it never carries
user data.  The key managers drive this connection through the TLS user
API (the client key manager acting as the TLS client, the server key
manager as the TLS server; role determination is specified in
{{role-determination}}) and relay the resulting TLS records to each
other as SCTP user messages, so that the TLS handshake is performed
between the two endpoints.

TLS records and key management control information travel over the same
SCTP association as the ULP's data and are distinguished by the SCTP
Payload Protocol Identifier (PPID): PPID 4242 for TLS records, PPID 4243
for control information, and the ULP's own PPID(s) for application data.
This separation lets the receiver deliver each message to the TLS
connection, to the key management logic, or to the ULP, as appropriate.
The framing and control messages are specified in {{tls-user-message}}.

When the TLS handshake completes, the key management function derives
keying material from the TLS Exporter and creates the DTLS Key Contexts
(DKCs) used by the DTLS Chunk Protection Operator to protect SCTP
packets; the TLS connection is then closed.  Key derivation and DKC
installation are specified in {{dtls-key-derivation}}.


## Protocol Flow Summary

The protocol operates in three phases:

1. **Initial Establishment:** After SCTP association setup (with
   DTLS Key Management Parameter negotiation), a TLS 1.3 handshake
   derives the initial Primary and Restart DKCs.  Protection is then
   enforced.

2. **Rekeying:** Either endpoint may trigger rekeying to derive fresh
   DKCs for the next epoch, providing forward secrecy and
   re-authentication.  The client key manager always performs the
   TLS 1.3 handshake; when the server key manager needs to rekey, it
   requests one via a Rekey Request.  After draining, the old Primary
   DKC is removed and the new Restart DKC replaces the old one.

3. **SCTP Restart:** The pre-established Restart DKC protects the
   COOKIE ECHO/COOKIE ACK exchange, followed by a new TLS handshake
   to establish fresh Primary and Restart DKCs.


# TLS Configuration Requirements {#tls-config}

## TLS Version

This document defines the usage of TLS 1.3 {{RFC9846}}.  Earlier
versions of TLS MUST NOT be used. An endpoint MUST NOT use more
than one TLS version during the lifetime of an SCTP association.

## Cipher Suite Constraints

Parameters not marked as "Y" in the "Recommended" column of TLS
registries SHOULD NOT be supported.  Non-AEAD cipher suites
or cipher suites without confidentiality MUST NOT be supported.
Cipher suites and parameters that do not provide ephemeral
key-exchange MUST NOT be supported ({{tls-config}}).

The cipher suites negotiated in the key management TLS connection
MUST include only those supported by the DTLS Chunk Protection
Operator.  The DTLS Chunk provides an API to query supported cipher
suites (see {{Section 7.3 of I-D.ietf-tsvwg-sctp-dtls-chunk}}).

## Authentication and Identity {#tls-auth}

DTLS in SCTP MUST be mutually authenticated.  It is
RECOMMENDED to use certificate-based authentication.

When certificates are used, the application is responsible for
trust anchor management, certificate chain validation, and identity
authentication.  The application defines what the identity is and
how it is encoded.  Guidance on server certificate validation can be
found in {{RFC9525}}.

All security decisions MUST be based on the peer's authenticated
identity, not on its transport layer identity.  Since SCTP
associations can use multiple IP addresses per endpoint, DTLS records
may arrive from different source IP addresses than those originally
authenticated.

The authenticated peer identity MUST remain stable across every TLS
connection established for the lifetime of the SCTP association,
including the connections used for rekeying ({{rekeying}}) and the
connection established after an SCTP restart ({{sctp-restart}}).
Clients and servers MUST NOT accept a change of identity during the
setup of a new TLS connection, but MAY accept negotiation of stronger
algorithms and security parameters.

This requirement is independent of the key management role: the Client
and Server roles are re-derived from the SCTP handshake and may differ
after an SCTP restart ({{sctp-restart}}), but an endpoint that changes
role MUST still present the same authenticated identity.

## Rekeying Considerations {#rekey-strategy}

Implementations MUST support criteria for initiating rekeying.
Implementations are RECOMMENDED to rekey at least every hour
and every 100 GB of data, which matches what is specified for IPsec in
{{ANSSI-DAT-NT-003}}.

Implementations MUST set up a new TLS connection using a full
handshake with new certificates before any certificate used in
the most recent handshake expires.

The PSK key exchange mode psk_ke MUST NOT be used as it does not
provide ephemeral key exchange ({{tls-config}}).
The TLS Key Update procedure MUST NOT be used, as it does not
provide a new ephemeral key for the key exporter.

TLS 1.3 tickets MAY be used for session resumption (see
{{session-resumption}}).

An endpoint MUST NOT have more than one key management
TLS connection per SCTP association at any time.

A rekeying procedure is in progress from step 3 to step 8
({{rekey-procedure}}) for the client key manager, and from step 4 to
step 9 for the server key manager.  A Rekey Request received while a
rekeying procedure is in progress SHOULD be ignored ({{sim-rekeying}}),
and the server key manager MUST NOT send a Rekey Request while a
rekeying procedure is in progress.

A rekeying procedure, whether triggered locally (for example, by the
criteria above or by a manual intervention) or by a Rekey Request, is
not delayed by a running drain timer ({{drain-timer}}).  If a new
rekeying procedure starts while the drain timer of the previous rekeying
procedure is running, the drain timer expires immediately: the epoch N
Primary DKC is removed, and the epoch N+1 Restart DKC replaces the epoch
N Restart DKC ({{sctp-restart-prereq}}).  The client key manager does
this when it starts the new rekeying procedure (step 3).  The server key
manager does this when it sends a Rekey Request (step 1) or when it
receives the first TLS record of the new handshake (step 4), whichever
comes first.  SCTP packets protected with the removed DKC that are still
in flight are discarded on reception, and SCTP retransmits their chunks
protected with the current Primary DKC, as it does for packets that
arrive after a regular drain timer expiry.

## Session Resumption {#session-resumption}

Support for TLS 1.3 session resumption is OPTIONAL. When supported, it
provides the following benefits:

* It avoids re-sending the full certificate chain on subsequent TLS
  connections. This saves a significant amount of message size,
  especially with post-quantum cryptography (PQC) certificates, which
  can be significantly larger than certificates based on Elliptic Curve
  Cryptography.

* It reduces processing, and thus energy consumption and latency.

* It allows successive TLS connections to be chained, increasing
  security by forcing an adversary to break them in sequence
  {{KTH-NCSA}}.

Session resumption tickets are delivered using the standard TLS 1.3
mechanisms: the server MAY send tickets unsolicited {{RFC9846}}, and
the client MAY request them {{RFC9149}}. To ensure the client has
the opportunity to obtain a ticket before the
TLS connection is torn down, the client key manager SHOULD initiate the
closure of the TLS connection, and the server key manager SHOULD NOT
close the TLS connection before the client has had the opportunity to
obtain the tickets it requested.


# Key Management Messages {#tls-user-message}

All key management messages MUST be sent as SCTP user messages
using reliable in-order delivery on stream 0.

There are two classes of these key management messages:

* SCTP user messages containing TLS records.
* SCTP user messages containing control information.

Each class is identified by its own PPID.

## TLS Records

One or more complete TLS records are sent as an SCTP user message.
These user messages MUST use PPID 4242.
Other SCTP user messages MUST NOT use this PPID.

## Control Messages {#control-messages}

Control messages MUST be sent as SCTP user messages with PPID 4243.
Each control message consists of a single byte that identifies its
type, as shown in {{control-message-format}}.  Other SCTP user
messages MUST NOT use PPID 4243.

~~~~~~~~~~~ aasvg
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|   Ctrl Type   |
+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #control-message-format title="Control Message Format" artwork-align="center"}

Ctrl Type: 8 bits
: Identifies the control message type.

The following control message types are defined:

  | Ctrl Type | Name               | Description                                                          |
  |-----------|--------------------|----------------------------------------------------------------------|
  | 0x01      | Read Key Installed | Notifies the peer that the read key is installed                     |
  | 0x02      | Rekey Request      | Requests the client key manager to initiate a rekeying TLS handshake |
{: #control-message-types title="Control Message Types"}

A control message that is not exactly one byte long, or that carries
the reserved value 0x00 or an unassigned Ctrl Type, MUST be silently
discarded.  A Read Key Installed control message that the key manager
is not expecting, and a Rekey Request received by the server key
manager or before the initial establishment ({{initial-establishment}})
has completed, MUST also be silently discarded.

### Read Key Installed {#read-key-installed}

The Read Key Installed control message (Ctrl Type = 0x01) is sent by
a key manager to its peer to indicate that it has installed its read
(receive) key.  On receiving it, the peer installs its write (send)
key and, during rekeying, switches sending to the new Primary DKC.
The message is used in the same way during initial establishment and
rekeying.

### Rekey Request {#rekey-request}

The Rekey Request control message (Ctrl Type = 0x02) is sent by the
server key manager to the client key manager to request that the client
key manager initiate a rekeying TLS handshake (see {{rekeying}}).

Because only the client key manager initiates a TLS handshake, the
server key manager uses this message when it determines that rekeying is
needed (per its own criteria in {{rekey-strategy}}).  This division of
roles allows an endpoint holding only the client role to implement a TLS
client only, and an endpoint holding only the server role to implement a
TLS server only.

Upon receiving a Rekey Request, the client key manager initiates a
rekeying TLS handshake as described in {{rekeying}}, unless a rekeying
procedure is already in progress, in which case the Rekey Request MUST
be ignored (see {{sim-rekeying}}).


# Key Derivation {#dtls-key-derivation}

## Role Determination {#role-determination}

Role determination and method selection follow the procedure defined
in {{Section 5.1 of
I-D.ietf-tsvwg-sctp-dtls-chunk}}.  After
the SCTP association is established, the key management function
retrieves from the SCTP stack's DTLS chunk API the assigned role
(Client or Server), the selected DTLS Key
Management Method, and the downgrade prevention data (both endpoints'
DTLS Key Management Parameters) used as input to key derivation.

## Exporter Context {#exporter-context}

DTLS Key Contexts are derived using the TLS Exporter as defined in
{{Section 7.5 of RFC9846}}.  The exporter context is constructed as
the concatenation of the following fields:

| Field | Length | Value |
|-------|--------|-------|
| Direction | 1 byte | 0x00 = Client-to-Server, 0x01 = Server-to-Client |
| Key role | 1 byte | 0x00 = primary, 0x01 = restart |
| Key type | 1 byte | 0x00 = Key, 0x01 = SN_KEY, 0x02 = IV |
| Client KM Param | variable | DTLS Key Management Parameter sent by the endpoint designated as Client |
| Server KM Param | variable | DTLS Key Management Parameter sent by the endpoint designated as Server |
{: #exporter-context-table title="Exporter Context" cols="l c l "}

Each DTLS Key Management Parameter ({{Section 4.1 of
I-D.ietf-tsvwg-sctp-dtls-chunk}}) is included as the
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

The endpoint with the Client role installs exports with
Direction=Client-to-Server as its write keys and
Direction=Server-to-Client as its read keys.  The
endpoint with Server role does the reverse.

The length of each exported value is determined by the negotiated
cipher suite.  The Key and SN_KEY values have the key length of the
AEAD algorithm ({{Section 7.3 of RFC9846}} and {{Section 4.2.3 of
RFC9147}}), and the IV value has the length of the per-record nonce
(iv_length in {{Section 7.3 of RFC9846}}).

## DKC Installation {#dkc-installation}

Each successful TLS handshake produces one or two (if restart is
supported) DKCs:

* A Primary DKC for regular SCTP traffic.
* A Restart DKC for the SCTP restart procedure.

The first DKC established for any SCTP association MUST use DTLS
epoch 3.  Each subsequent Primary DKC, together with its paired
Restart DKC, uses the next consecutive epoch.  After an SCTP restart,
the epoch resets to 3.

If SCTP restart is supported, the endpoint MUST generate a Restart DKC
for each epoch where a Primary DKC is generated.  The Restart DKC uses
the same DTLS epoch as the Primary DKC generated alongside it; the two
are distinguished not by epoch but by the restart indicator (the R bit
in the DTLS chunk, see {{I-D.ietf-tsvwg-sctp-dtls-chunk}}) and by the
Key role field of the exporter context ({{exporter-context}}).  The
Restart DKC MUST be maintained in a well-defined state (initialized but
never used for regular traffic) so that both endpoints have a consistent
view of sequence numbers and replay window. The Restart DKC requirements
in {{sctp-restart-prereq}} apply. If SCTP restart is supported, the
Restart DKC is committed to persistent secure storage, replacing the
previously stored one, as soon as it has been exported.

# Requirements on the DTLS Chunk {#dtls-chunk-requirements}

Note to the RFC Editor: This section is expected to be specified in
{{I-D.ietf-tsvwg-sctp-dtls-chunk}}.  Once it is, this section will be
replaced by a reference to that document.

## First Use Indication {#first-use-indication}

The key management method defined in this document relies on the DTLS
Chunk indicating the first successful use of each DKC in the receive
direction ("Indication of First use of Key Epoch", {{Section 7.11 of
I-D.ietf-tsvwg-sctp-dtls-chunk}}).  The server key manager uses this
indication to start the drain timer during rekeying (step 9 of
{{rekey-procedure}}) and after an SCTP restart (step 13 of
{{restart-procedure}}).

A DTLS Chunk implementation used with this
key management method MUST provide this
indication for every DKC, Primary and Restart, as follows:

* Each DKC has an associated flag, "already used".

* When the receive key material of a DKC is installed ("Establish
  Receive Key Material", {{Section 7.5 of
  I-D.ietf-tsvwg-sctp-dtls-chunk}}), the "already used" flag of that
  DKC is set to false.  This applies to every installation, including
  a Restart DKC installed from persistent storage.

* When a received DTLS chunk has been successfully decrypted and
  authenticated with a DKC, has passed replay protection, and the
  "already used" flag of that DKC is false, the DTLS Chunk sets the
  flag to true and indicates the first use of the DKC to the key
  manager.  The indication contains the SCTP association, the restart
  indicator (R bit), and the DTLS epoch of the DKC.

* DTLS chunks that fail decryption, authentication, or replay
  protection MUST NOT change the flag and MUST NOT trigger the
  indication.

As a result, the indication is given exactly once for each installed
DKC, independently of the epoch values used by other DKCs.  In
particular, it is given for the epoch 3 Primary DKC installed after an
SCTP restart, even though DKCs with an equal or higher epoch were used
before the restart.
The key manager ignores indications that it is not waiting for.

~~~~~~~~~~~ pseudocode
install_receive_keys(DKC):
    DKC.already_used = false

on_receive(chunk):
    DKC = lookup(association, chunk.R, chunk.epoch)
    if DKC exists and decrypt(chunk, DKC) == SUCCESS
       and replay_check(chunk, DKC) == PASS:
        if DKC.already_used == false:
            DKC.already_used = true
            indicate_first_use(association, chunk.R, chunk.epoch)
        process the SCTP packet
~~~~~~~~~~~
{: #first-use-pseudocode title="First Use Indication"}

When the socket API of {{Section 8 of I-D.ietf-tsvwg-sctp-dtls-chunk}}
is used, the indication is delivered as an SCTP_DTLS_NEW_EPOCH_EVENT
notification ({{Section 8.1.2 of I-D.ietf-tsvwg-sctp-dtls-chunk}})
with sdne_epoch set to the epoch of the DKC.  Since this notification
does not carry the restart indicator, the key manager associates it
with the DKC it is waiting for: the Primary DKC whose receive key
material was most recently installed and for which no indication has
been received yet.

# Procedures {#procedures}

Steps that involve the Restart DKC apply only if SCTP restart is
supported by both endpoints ({{sctp-restart-prereq}}).

## Initial Establishment {#initial-establishment}

~~~~~~~~~~~ aasvg

 Initiator                                       Responder
     |                                               |
  1. +-----------[INIT]----------------------------->|
     |<----------[INIT ACK]--------------------------+
     +-----------[COOKIE ECHO]---------------------->| 2.
  3. |<----------[COOKIE ACK]------------------------+
     |                                               |
     | TLS  client KM                 server KM  TLS |
     |  |    |                               |    |  |
  4. |  |<---+ Start TLS cli   Start TLS srv +--->|  |    5.
     |  |    |                               |    |  |
     |  |<---+-------------------------------+--->|  |    6.
     |  |    |                               |    |  |
     |  |    |                               |    |  |
  7. |  +--->| READ installed                |    |  |
     |  |    +-------------[RKI]------------>|    |  |    8b.
     |  |    |                               |<---+  |    8a.
     |  |    | (wait for done + client RKI)  |    |  |
     |  |    |   (install RX+TX, enforce)    |    |  |
     |  |    |<------------[RKI]-------------+    |  |
  9. |  |    |   READ installed, enforce     |    |  |
     |  |    |                               |    |  |
     |                                               | -.
 10. +-----------[protected APP DATA]--------------->|  | APP
     +<----------[protected APP DATA]----------------+  +---
     |                    ...                        |  |

~~~~~~~~~~~
{: #initial-establishment-diagram title="Initial Establishment" artwork-align="center"}

Legend: TLS = local TLS engine; client KM/server KM = client/server key
manager; RKI = Read Key Installed control message; RX+TX = read and
write keys.

  {{initial-establishment-diagram}} shows the case where the SCTP
  Initiator takes the client key manager role; the opposite case is the
  same with the key manager roles exchanged.  In the following,
  Initiator and Responder refer to the SCTP roles, and Client and Server
  refer to the key manager roles, which are also the TLS roles. The key
  managers drive TLS solely through the TLS user API.

  The procedure is as follows:

  1. The Initiator sends INIT containing the DTLS Key Management
     Parameter ({{Section 4.1 of I-D.ietf-tsvwg-sctp-dtls-chunk}}) with
     this method's identifier (see {{sec-iana-psi}}) in its
     preference-ordered list.

  2. The Responder enters ESTABLISHED state. It retrieves the agreed
     DTLS Key Management Method and role from the SCTP stack (e.g.,
     using the "Get Agreed DTLS Key Management Method and Role" API
     defined in {{Section 7.2 of I-D.ietf-tsvwg-sctp-dtls-chunk}}) and
     verifies that the selected method matches the one defined in this
     document (see {{sec-iana-psi}}) and gets the assigned role as key
     manager client or server (in the example depicted in
     {{initial-establishment-diagram}} it is server).

  3. The Initiator enters ESTABLISHED state. It performs the same
     retrieval and verification as the Responder, confirming the
     assigned role as key manager client or server (in the example
     depicted in {{initial-establishment-diagram}} it is client).

  4. The client key manager starts a TLS 1.3 handshake, limiting offered
     cipher suites to those supported by the DTLS Chunk Protection
     Operator, and relays the resulting flight of TLS records to the
     server key manager per {{tls-user-message}}.

  5. The server key manager starts a TLS 1.3 server and is ready to
     relay any received TLS records to the TLS server and to forward any
     TLS records produced by the TLS server to the client key manager
     per {{tls-user-message}}.

  6. The TLS endpoints continue to exchange TLS messages to complete the
     TLS handshake.

  7. The client key manager's TLS handshake completes. It exports the
     Primary and Restart DKCs, installs the Server-to-Client key
     material of the Primary DKC as its read (receive) key, and sends a
     Read Key Installed control message ({{read-key-installed}}) to the
     server key manager. The Restart DKC is installed in step 9.

  8. The server key manager proceeds only after both its own TLS
     handshake has completed (8a) and it has received the client key
     manager's Read Key Installed control message (8b). Once both
     conditions are met, it exports the Primary and Restart DKCs and
     installs their read (receive) and write (send) keys, calls
     Require Protected SCTP Packets to enforce DTLS chunk protection for
     all future packets, informs the ULP that the association is
     protected, and sends a Read Key Installed control message
     ({{read-key-installed}}) to the client key manager.

  9. The client key manager receives the Read Key Installed control
     message, installs the Client-to-Server key material of the Primary
     DKC as its write (send) key, installs the read and write keys of
     the Restart DKC, calls Require Protected SCTP Packets to enforce
     DTLS chunk protection for all future packets, and informs the ULP
     that the association is protected.

  10. Protected application traffic can begin.

  If the TLS handshake fails, the key managers proceed as described in
  {{error-handling}}.

  After key installation, the TLS connection SHOULD be closed promptly
  following the procedure in {{session-resumption}}.


## Rekeying {#rekeying}

### Triggering Criteria

Rekeying is triggered by implementation-configurable criteria,
including:

* Time elapsed since last peer authentication.
* Volume of data transferred since last rekeying.
* Approaching the cipher suite's AEAD usage limits
  ({{Section 4.5.3 of RFC9147}}).

### Procedure {#rekey-procedure}

  The client key manager and the server key manager keep the same key
  management roles for the entire lifetime of the SCTP association: the
  client key manager is always the TLS client and the server key manager
  is always the TLS server, for the initial handshake and for every
  rekeying procedure.  Consequently, only the client key manager ever
  initiates a TLS handshake.  This allows an endpoint holding only the
  client role to implement only a TLS client, and an endpoint holding
  only the server role to implement only a TLS server.

  Either endpoint may need to rekey.  This is handled as follows:

  * If the client key manager needs to rekey, it directly initiates a
    rekeying TLS handshake.

  * If the server key manager needs to rekey, it cannot initiate a TLS
    handshake itself.  Instead it sends a Rekey Request control message
    ({{rekey-request}}) to the client key manager, which then initiates
    the rekeying TLS handshake.

  Because the key management messages are carried on SCTP stream 0 with
  reliable, in-order delivery, the key managers do not need their own
  retransmission mechanism: SCTP retransmits lost messages, and if it
  cannot deliver them, the association fails ({{Section 8.1 of
  RFC9260}}).

~~~~~~~~~~~ aasvg

 Initiator                                       Responder
     |    (traffic continues using epoch N DKC)      |
     |                                               |
     | TLS  client KM                 server KM  TLS |
     |  |    |                               |    |  |
     |  |    |    (server needs to rekey:)   |    |  |
  2. |  |    |<----------[Rekey Req]---------+    |  |  1.
     |  |    |                               |    |  |
     |  |    |      (client rekeys,          |    |  |
     |  |    |       or got Rekey Req:)      |    |  |
  3. |  |<---+ Start TLS cli   Start TLS srv +--->|  |  4.
     |  |    |                               |    |  |
     |  |<---+-------------------------------+--->|  |  5.
     |  |    |                               |    |  |
  6. |  +--->| READ (epoch N+1)              |    |  |
     |  |    +-----------[RKI]-------------->|    |  |  7b.
     |  |    |                               |<---+  |  7a.
     |  |    |  (wait own done + cli RKI)    |    |  |
     |  |    |  (install RX+TX Primary N+1,  |    |  |
     |  |    |    TX->N+1)                   |    |  |
     |  |    |<----------[RKI]---------------+    |  |
  8. |  |    |   WRITE N+1, drain, TX->N+1   |    |  |
     |  |    |                               |    |  |
     |  |    |  (first use N+1,              |    |  |  9.
     |  |    |   start drain)                |    |  |
     |  |    |                               |    |  |
     |    (traffic transitions to epoch N+1 DKC)     |
     |    (after draining:  remove Primary N,        |
     |     replace Restart N by N+1)                 |

~~~~~~~~~~~
{: #rekey-diagram title="Rekeying Procedure" artwork-align="center"}

Legend: TLS = local TLS engine; client KM/server KM = client/server key
manager; Rekey Req = Rekey Request control message; RKI = Read Key
Installed control message; RX+TX = read and write keys; N, N+1 = old and
new epoch; TX->N+1 = switch sending to the epoch N+1 DKC.

  {{rekey-diagram}} shows both triggers.  Steps 1 and 2 (the Rekey
  Request) are present only when the server key manager is the one that
  needs to rekey; when the client key manager needs to rekey, it starts
  directly at step 3.  As in initial establishment, the key managers
  drive TLS solely through the TLS user API.

  The procedure is as follows:

  1. If the server key manager needs to rekey, it sends a Rekey Request
     control message ({{rekey-request}}) to the client key manager.  The
     server key manager does not start a TLS handshake.

  2. The client key manager receives the Rekey Request.  If it is not
     already rekeying, it proceeds to initiate a rekeying TLS handshake
     (step 3).  If it is already rekeying, it SHOULD ignore the Rekey
     Request (see {{sim-rekeying}}).

  3. The client key manager starts a rekeying TLS handshake to derive
     the epoch N+1 DKCs and relays the resulting flight of TLS records
     to the server key manager per {{tls-user-message}}.

  4. The server key manager acts as the TLS server for the rekeying
     handshake, as in step 5 of {{initial-establishment}}.

  5. The TLS endpoints continue to exchange TLS messages to complete the
     TLS handshake. The authenticated peer identity MUST remain the same
     as in the previous TLS connections (see {{tls-auth}}).

  6. The client key manager's TLS handshake completes.  It exports the
     epoch N+1 Primary and Restart DKCs, installs the Server-to-Client
     key material of the Primary DKC as its read (receive) key, and
     sends a Read Key Installed control message ({{read-key-installed}})
     to the server key manager.  The Restart DKC is kept for later
     installation ({{sctp-restart-prereq}}).

  7. The server key manager proceeds only after both its own TLS
     handshake has completed (7a) and it has received the client key
     manager's Read Key Installed control message (7b).  It then exports
     the epoch N+1 Primary and Restart DKCs and installs the read
     (receive) and write (send) keys of the Primary DKC.  The Restart
     DKC is kept for later installation ({{sctp-restart-prereq}}).  The
     server key manager switches sending to the epoch N+1 Primary DKC
     ("Set Key to Use", {{Section 7.8 of
     I-D.ietf-tsvwg-sctp-dtls-chunk}}) and then sends a Read Key
     Installed control message ({{read-key-installed}}) to the client
     key manager.

  8. The client key manager receives the Read Key Installed control
     message ({{read-key-installed}}), installs the Client-to-Server
     key material of the epoch N+1 Primary DKC as its write (send)
     key, switches sending to the epoch N+1 Primary DKC, and starts
     the drain timer ({{drain-timer}}).  When the drain timer
     expires, the epoch N Primary DKC is removed and the epoch N+1
     Restart DKC replaces the epoch N Restart DKC
     ({{sctp-restart-prereq}}).

  9. When the server key manager is notified of the first use of the
     epoch N+1 Primary DKC in a received DTLS chunk
     ({{first-use-indication}}), it starts the drain timer
     ({{drain-timer}}).  When the drain timer expires, the epoch N
     Primary DKC is removed and the epoch N+1 Restart DKC replaces
     the epoch N Restart DKC ({{sctp-restart-prereq}}).

  If the TLS handshake fails, the key managers proceed as described in
  {{error-handling}}.

  The new DKCs use epoch N+1 (where N is the current epoch when
  initiating rekeying). The epoch N and epoch N+1 Primary DKCs
  coexist until the drain timer expires. The epoch N Restart
  DKC remains installed until it is replaced ({{sctp-restart-prereq}}).

  A client key manager that does not switch sending to the epoch N+1
  Primary DKC after receiving the server's Read Key Installed control
  message violates this specification.  The server key manager can
  detect this, for example because no first use indication is given
  while the AEAD decryption counter of the epoch N Primary DKC
  ({{Section 7.13 of I-D.ietf-tsvwg-sctp-dtls-chunk}}) increases
  above implementation-specific values or for an implementation-specific
  time.
  In that case, the server key manager SHOULD abort the SCTP
  association with a Protocol Violation error cause
  ({{Section 3.3.10.13 of RFC9260}}).

  If the association is idle, the first use indication, and with it
  the installation of the new Restart DKC, is delayed until the next
  SCTP packet from the client.

  Rekeying always uses ephemeral key exchange ({{tls-config}}).

  After rekeying, the TLS connection SHOULD be closed promptly
  following the procedure in {{session-resumption}}.

### Drain Timer Considerations {#drain-timer}

The drain timer determines how long the old (epoch N) Primary DKC is
retained after the new (epoch N+1) Primary DKC has been activated.
Its purpose is to allow in-flight packets protected with the old keys
to be received and processed before those keys are removed.  When the
drain timer expires, the old Primary DKC is removed and the new
Restart DKC replaces the old one ({{sctp-restart-prereq}}).

When the client key manager receives the Read Key Installed control
message in step 8, it knows that the server key manager is already
sending with the new Primary DKC.  It can therefore use a short drain
timer, long enough for reordered or delayed packets protected with
the old keys to be received.

The server key manager starts its drain timer when it is notified
that the client key manager has started to use the new Primary DKC
({{first-use-indication}}).  For the same reason, a short drain timer
is sufficient.

The RECOMMENDED default value of the drain timer is 3 seconds for
both key managers.

### Concurrent Rekey Requests {#sim-rekeying}

Because only the client key manager ever initiates a TLS handshake,
two ClientHellos cannot cross and no tie-breaker is needed.

The only concurrency to resolve is when the server key manager sends a
Rekey Request while the client key manager has already initiated (or is
about to initiate) a rekeying procedure to epoch N+1.  Since both the
direct client trigger and the server's Rekey Request lead to the same
outcome (a single client-initiated rekeying procedure to epoch N+1), the
client key manager treats an incoming Rekey Request as redundant and
SHOULD ignore it while a rekeying procedure is already in progress.

Similarly, if the server key manager needs to rekey while a
client-initiated rekeying procedure is already in progress, it does not
send a Rekey Request, as the in-progress rekeying procedure already
satisfies the need.

## SCTP Association Restart {#sctp-restart}

### Restart DKC Requirements {#sctp-restart-prereq}

The following applies when both endpoints have indicated restart
support (R bit) in the DTLS Key Management Parameter:

* Both endpoints MUST have a valid Restart DKC.
* The Restart DKC MUST be stored securely and persistently to survive
  crash events ({{Section 10.4 of I-D.ietf-tsvwg-sctp-dtls-chunk}}).
* At most one Restart DKC is installed at any time.  A Restart DKC
  derived during rekeying ({{rekey-procedure}}) or after an SCTP
  restart ({{restart-procedure}}) is not installed when it is exported.
  Instead, when the drain timer ({{drain-timer}}) expires, the new
  Restart DKC replaces the installed Restart DKC.
  The Restart DKC derived during initial establishment
  ({{initial-establishment}}) is installed together with the Primary
  DKC, as no Restart DKC exists yet.
* The Restart DKC is stored and installed as the read and write keys
  of the endpoint, as derived in the association instance in which it
  was exported.  A change of the key management roles during an SCTP
  restart does not affect it; the new roles apply only to the DKCs
  derived after the restart.

### Restart Procedure {#restart-procedure}

~~~~~~~~~~~ aasvg

 Initiator                                Responder
     |                                        |
  1. | (install Restart DKC from storage)     |
     |                                        | -.
  2. +---------[INIT]------------------------>|   | Plain
  3. |<--------[INIT ACK]---------------------+   +------
     |                                        | -'
     |                                        | -.
  4. +------[DTLS(COOKIE ECHO)]-------------->|   | Protected
  5. |<-----[DTLS(COOKIE ACK)]----------------+   +----------
  6. |                                        | -'
     | (TLS handshake for new keys,           |
     |     steps 7-13, as in rekey case)      |
     |                                        |
 14. +------[DTLS(protected APP DATA)]------->|  APP DATA
     |<-----[DTLS(protected APP DATA)]--------+
     |                  ...                   |


~~~~~~~~~~~
{: #restart-diagram title="SCTP Restart Procedure" artwork-align="center"}

Steps 1 to 5 follow {{Section 5.4 of I-D.ietf-tsvwg-sctp-dtls-chunk}}.

1. The restarting endpoint (Initiator) retrieves the Restart DKC from
   persistent secure storage and installs it for both the send and
   receive directions.  It then removes the Restart DKC from persistent
   storage, because its sequence numbers will be used.

2. The Initiator sends an INIT chunk with a Verification Tag of 0.  It
   includes the DTLS Key Management Parameter with the same method
   list and the same role and restart settings as before, but with a
   new random Tie Breaker.

3. The Responder, which did not restart, replies with an INIT ACK
   chunk in plaintext as specified in {{RFC9260}}.  It includes the
   DTLS Key Management Parameter with the same method list and the
   same role and restart settings, but with a new random Tie Breaker.

4. The Initiator sends COOKIE ECHO in a DTLS chunk protected with
   the Restart DKC (R bit set).

5. The Responder replies COOKIE ACK in a DTLS chunk protected with
   the Restart DKC (R bit set); it then removes the restart key
   material from persistent storage, as its sequence
   numbers will be used.

6. Both endpoints have a restarted association, whose state is
   as described in {{Section 5.2.4 of RFC9260}}, with two differences
   specific to this document: DTLS chunk protection is enforced using
   the Restart DKC (the COOKIE ECHO and COOKIE ACK were exchanged
   protected with it), and the DTLS Key Management Client and Server
   roles may differ from those of the previous instance of the
   association, since the new INIT handshake re-runs role determination.
   The authenticated peer identity, however, MUST remain the same as in
   the previous instance of the association (see {{tls-auth}}).
   The ULP MAY be informed that the association is restarted at this
   point.  ULP traffic MAY begin immediately using the Restart DKC.

7. Steps 7 to 13 perform the TLS 1.3 handshake and key installation,
   and correspond one-to-one to steps 3 to 9 of the rekeying procedure
   ({{rekey-procedure}}).  In these steps, "epoch N" refers to the
   Restart DKC used to restart the association, and "epoch N+1" refers
   to the new Primary and Restart DKCs, which use epoch 3
   ({{Section 5.4 of I-D.ietf-tsvwg-sctp-dtls-chunk}}).  The steps
   differ from the rekeying procedure only as follows:

   * At the key transition (steps 11 and 12), each endpoint switches
     the active DTLS key context from the Restart DKC to the new Primary
     DKC after the Read Key Installed exchange completes (the Read Key
     Installed messages themselves are protected with the Restart DKC),
     rather than enforcing protection for the first time; the ULP was
     already informed at step 6.

   * At step 13, the server key manager starts the drain timer when it
     is notified of the first use of the epoch 3 Primary DKC in a
     received DTLS chunk ({{first-use-indication}}).

{:start="14"}
14. Protected application traffic uses the new Primary DKC.

After restart, the new Primary DKC is required to use epoch 3 (the epoch
resets), see {{Section 5.4 of I-D.ietf-tsvwg-sctp-dtls-chunk}}.

The Responder MUST NOT change the Restart DKC during the restart
procedure. The Restart DKC used for the restart is replaced by the
new Restart DKC when the drain timer expires.

It is RECOMMENDED to complete the TLS handshake and install new DKCs
as soon as possible after restart, to minimize the window during
which no Restart DKC is available for a subsequent restart.

Note: There may be a short time gap after association establishment
where no Restart DKC is yet installed.  If an SCTP restart is
initiated during that time, it will fail.  However, this is unlikely
as the restarting endpoint sends INIT multiple times with exponential
back-off. After a crash during the drain, the restarting endpoint uses
the new Restart DKC, while its peer still has the old one installed
until its own drain timer expires (about 3 seconds). The restart fails
until then and succeeds on a later INIT retransmission.
The resulting delay is bounded by the drain timer.

# Error Handling {#error-handling}

TLS reports errors with TLS alert messages.  A TLS alert is sent in an
SCTP user message with PPID 4242, like any other TLS record
({{tls-user-message}}).

If a TLS handshake fails during initial establishment and the
implementation determines that it can address the cause of the error
(for example, by retrying with different parameters, as long as doing
so does not compromise security), it SHOULD retry
the TLS handshake.  Otherwise, the SCTP association MUST be aborted.

If a TLS handshake fails during rekeying, and the current DKC has not
yet reached its usage limits, the implementation SHOULD retry the
handshake. If retry is not possible, or if the current DKC has reached
its usage limits, the association MUST be aborted.

A TLS handshake that fails because the peer could not be
authenticated, including a change of the authenticated peer identity
({{tls-auth}}), MUST NOT be retried; the SCTP association MUST be
aborted.

If a TLS handshake fails after an SCTP restart ({{restart-procedure}}),
the rules for rekeying apply, with the Restart DKC as the current DKC.

# Security Considerations {#security-considerations}

## General

The security considerations given in {{RFC9846}}, {{RFC9147}}, and
{{RFC9260}} also apply to this document.  BCP 195 {{RFC9325}}
{{RFC8996}} provides recommendations and requirements for improving
the security of deployed services that use TLS.  BCP 195 MUST be
followed.

## Privacy Considerations

Although DTLS in SCTP provides confidentiality for user messages and
almost all SCTP chunks, the SCTP common header, DTLS chunk header,
and DTLS record header are not confidentiality protected.  An
attacker can correlate TLS connections over the same SCTP association
using the SCTP common header.

To provide identity protection, it is RECOMMENDED to use
certificate-based authentication in TLS 1.3 and to not reuse tickets.
TLS 1.3 with external PSK authentication does not provide identity
protection.

By mandating ephemeral key exchange and cipher suites with
confidentiality, DTLS in SCTP effectively mitigates many
forms of passive pervasive monitoring.  Frequent rekeying forces
attackers to perform dynamic key exfiltration and limits the amount
of compromised data due to key compromise.

Implementations of this key management method SHOULD NOT allow the ULP
to exchange any data, other than the key management messages defined in
this document, until the peer is authenticated and the local endpoint
and the remote endpoint have both installed read and write keys and
enforced protection. This is to avoid any information leakage from the
ULP to unintended parties.

## Unprotected Key Management Messages

Until DTLS chunk protection is enforced during initial establishment
({{initial-establishment}}), key management messages are carried in
unprotected SCTP packets.  An on-path attacker can inject, modify, or
drop them.  Modified or injected TLS records cause the TLS handshake to
fail, which results in the association being aborted; this is a denial
of service that an on-path attacker can also cause by dropping
packets.  An injected Read Key Installed control message can cause the
server key manager to start sending with the new keys before the
client key manager has installed its read key; SCTP recovers the
resulting packet loss by retransmission.  An injected Rekey Request is
discarded ({{control-messages}}).  Once protection is enforced, all key
management messages are protected by the DTLS chunk.

# IANA Considerations {#iana-considerations}

Note to the RFC Editor: Please replace "RFC-To-Be" in this section with
the RFC number assigned to this document, and remove this note.

## DTLS Key Management Method Identifier {#sec-iana-psi}

IANA is requested to assign one DTLS Key Management Method Identifier
in the "DTLS Key Management Method" registry defined by
{{I-D.ietf-tsvwg-sctp-dtls-chunk}} to identify the
key management method defined in this document.

| Identifier | Key Management Method Name                   | Reference | Contact       |
|------------|----------------------------------------------|-----------|---------------|
| 192        | TLS-Based Key Management for SCTP DTLS Chunk | RFC-To-Be | Draft Authors |
{: #iana-psi title="DTLS Key Management Method Identifier" cols="r l l l"}

## TLS Exporter Labels {#iana-export-label}

IANA is requested to register the following value in the "TLS
Exporter Labels" registry within the "Transport Layer Security (TLS)
Parameters" registry group ({{RFC5705}}, {{RFC8447}}), with Reference
RFC-To-Be and an empty Comment.  The label is used only with TLS, not
with DTLS, so DTLS-OK is set to "N".

| Value                         | DTLS-OK | Recommended |
|-------------------------------|---------|-------------|
| EXPORTER_TLS_FOR_DTLS_IN_SCTP | N       | N           |
{: #iana-tls-exporter title="TLS Exporter Label" cols="l l l"}

## DTLS Chunk Key Management Control Message Types {#sec-iana-ctrl-types}

IANA is requested to create a new registry called "DTLS Chunk
Key Management Control Message Types" in the Stream Control
Transmission Protocol (SCTP) Parameters group.  This registry
governs the values of the Ctrl Type field of the control messages
defined in {{control-messages}}.

The registration policy for this registry is IETF Review as defined in
{{RFC8126}}.

Each entry in the registry MUST contain the following fields:

* Ctrl Type: an 8-bit unsigned integer value (0x00-0xFF).
* Name: a short, human-readable name for the control message type.
* Reference: a reference to the document defining the control message
  type.

The initial contents of the registry are shown in
{{iana-ctrl-types}}.  The value 0x00 is reserved.

| Ctrl Type   | Name               | Reference |
|-------------|--------------------|-----------|
| 0x00        | Reserved           | RFC-To-Be |
| 0x01        | Read Key Installed | RFC-To-Be |
| 0x02        | Rekey Request      | RFC-To-Be |
| 0x03-0xFF   | Unassigned         |           |
{: #iana-ctrl-types title="DTLS Chunk Key Management Control Message Types" cols="l l l"}

## SCTP Payload Protocol Identifier {#sec-iana-ppid}

In the "Payload Protocol Identifiers" registry of the "Stream Control
Transmission Protocol (SCTP) Parameters" registry group, PPID 4242 is
currently registered as "DTLS Chunk Key-Management Messages" with the
reference "\[draft-westerlund-tsvwg-sctp-dtls-chunk-01\]".  IANA is
requested to update the name and the reference of this entry as shown
in {{iana-payload-protection-id}}.

IANA is also requested to register PPID 4243 in the same registry, as
shown in {{iana-payload-protection-id}}.

| ID Value | SCTP Payload Protocol Identifier              | Reference |
|----------|-----------------------------------------------|-----------|
| 4242     | TLS records for DTLS Chunk Key Management      | RFC-To-Be |
| 4243     | Control Messages for DTLS Chunk Key Management | RFC-To-Be |
{: #iana-payload-protection-id title="Payload Protocol Identifier" cols="r l l"}
