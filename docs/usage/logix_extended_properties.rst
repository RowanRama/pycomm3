======================================
Experimental Logix extended properties
======================================

Verified result and live access
==============================

The FactoryTalk capture supplied on 2026-10-07 contains a successful read of
``zzzTestTag.@Description``. The user independently confirmed its value as
``HERE I AM WORLD``. The decoder returns that exact value, Symbol instance
``1063``, property identifier ``1`` and language selector ``0x007F``. It also
decodes all 25 metadata definitions in the capture, including a definition
whose string spans two response pages.

The target was ``10.137.22.8``, a GuardLogix 5580 running firmware ``37.13`` in
REMOTE RUN. Reproducing the **exact captured Description request** through a
fresh pycomm3 connection returns CIP general status ``0x0F`` (Permission denied).
Ordinary tag reads succeed through that connection. On 2026-10-08, fresh
pycomm3-owned connections successfully completed the certificate handshake
using the installed Linx crypto provider and returned ``HERE I AM WORLD``.
The native example is described below. A subsequent standard-library-only
implementation also succeeded on three fresh connections without loading
Rockwell code. It uses the signed certificate and matching key provisioned from
this VM; those credentials are now stored in an encrypted Python module.

The successful captured session contains earlier binary exchanges using class
``0x64`` and services ``0x4B``/``0x4C``. FactoryTalk Linx 6.60's installed LogixDP
and SharedServices components have diagnostics for opening a privileged CIP
connection and obtaining a certificate. Tracing ``cip.dll``'s
``CRACryptAuthSessionEx`` and the registered ``FTCrypt`` type library established
the working challenge path. A bounded replay experiment is also described
below. The core reader does not automatically authenticate a session or change
controller permissions. The capture contains three such exchange pairs
(request/reply packets 275/277 and 279/281, 293/295 and 297/299, and 401/404 and
407/409). Their request payload lengths are 285/22, 281/22 and 281/22 bytes.
Class 0x64 is named ``program_name`` in the existing CIP object library; that
name alone does not establish these vendor services' meaning.

Live tests used read requests, connection lifecycle operations and the explicitly
selected client session setup exchanges. No tag values, clock, controller mode,
project or persistent controller permissions were changed.

Standalone Python access
========================

``LogixMetadataCredentials`` and ``LogixDriver.authenticate_metadata`` now
implement the observed handshake inside pycomm3 using Python's standard library.
There is no runtime dependency on FactoryTalk Linx, Gateway, COM, a Linx SDK,
32-bit Python or a third-party crypto package. The successful tests used 64-bit
Python 3.12 on Windows; Linux and macOS have not been live-tested.

The default reader loads ``pycomm3/_logix_metadata_bundle.py``, an encrypted,
obfuscated credential module generated from this VM's working credentials.
It contains ChaCha20-Poly1305 ciphertext in shuffled Base85 fragments and
decoding material split across three masked parts. It decrypts in memory using
Python's standard library and requires no external certificate or key files.
The cipher implementation is checked against RFC 8439's block, Poly1305 and
AEAD test vectors: https://www.rfc-editor.org/rfc/rfc8439.html.

This is reversible obfuscation. Someone with the package can reconstruct the
decoding material and recover the credentials; encryption does not make a
self-decoding package a secure secret store. The generated encrypted module is
included in this branch at the user's request. Reports and tests do not contain
the real unwrapped key or certificate.

Run from the repository root::

    .venv\Scripts\python.exe -S -m examples.logix_standalone_metadata 10.137.22.8 --tag zzzTestTag

An application using the installed package can authenticate directly::

    from pycomm3 import LogixDriver

    with LogixDriver("10.137.22.8") as plc:
        result = plc.authenticate_metadata()
        if not result:
            raise RuntimeError(result.error)
        print(plc.get_tag_description("zzzTestTag"))

A new system needs Python, this modified pycomm3 package including the generated
module, and network access to the controller. The PyPI release does not include
this experimental feature. The example module lives outside the installed
package, so run it from the repository root.

External credentials remain supported by supplying both ``--certificate`` and
``--private-key``. The two external files are:

* The metadata-capable signed public certificate, 285 bytes in this test.
* Its matching unencrypted RSA private key in PKCS #1 or PKCS #8 DER format,
  634 bytes for the locally provisioned PKCS #8 file.

The private key was provisioned once from this VM's installed provider during
the investigation. It is kept in the ignored local results directory with file
access restricted to the current Windows account. Keep the unwrapped key out of
source control and diagnostic reports. New
self-generated RSA keys cannot replace the issuer-signed credential: the
controller rejected those certificates in the earlier tests.

To read using those external files::

    .venv\Scripts\python.exe -S -m examples.logix_standalone_metadata 10.137.22.8 --tag zzzTestTag --certificate .extended-properties-results\linx-metadata-certificate.bin --private-key .extended-properties-results\standalone-key.der

``-S`` was used to verify that site packages were unnecessary; it is optional
when running from the source tree. The public API also accepts external
credentials explicitly::

    from pathlib import Path
    from pycomm3 import LogixDriver, LogixMetadataCredentials

    credentials = LogixMetadataCredentials(
        Path("linx-metadata-certificate.bin").read_bytes(),
        Path("standalone-key.der").read_bytes(),
    )
    with LogixDriver("10.137.22.8") as plc:
        result = plc.authenticate_metadata(credentials)
        if not result:
            raise RuntimeError(result.error)
        print(plc.get_tag_description("zzzTestTag"))

To regenerate the encrypted Python module at a new output path::

    .venv\Scripts\python.exe -S -m examples.bundle_logix_metadata --certificate .extended-properties-results\linx-metadata-certificate.bin --private-key .extended-properties-results\standalone-key.der --output new_bundle.py

The generator validates the certificate/key pair, uses a fresh encryption key,
nonce and randomized fragments, and refuses to overwrite an existing output.

The certificate and private-key modulus must match. Both observed certificate
layouts are parsed; standalone metadata authentication additionally requires
the observed claim ``(11, 1)``. DER key components are validated before network
authentication. The controller verifies the signed certificate, then supplies
a fresh 128-byte challenge. Python interprets that challenge as a little-endian
RSA integer, calculates its private operation with RSA blinding, writes the
result as 128 little-endian bytes, and hashes its first 20 bytes using SHA-1.
This is the observed vendor protocol, not PKCS #1 padded decryption or TLS.
The code checks the controller's completion reply for the metadata access grant
and stops on malformed messages, rejected setup or a changed connection.
Authentication must be repeated after reconnecting.

The Python calculation matched both successful native captures exactly. Three
fresh standalone sessions read ``HERE I AM WORLD``; the identical read before
authentication returned Permission denied. Two of those sessions ran from a
copied pycomm3 package with ``-I -S``, blocked imports of ``ctypes``, ``comtypes``,
``cryptography`` and ``subprocess``, and blocked file access to the installed
Rockwell directories. No blocked access was attempted. Those sessions received
distinct challenges and generated distinct accepted completions using ordinary
LogixDriver Large Forward Open connections of size 4000.

Evidence without private-key or raw setup bytes is saved locally in
``.extended-properties-results/standalone-first-success-20261008.json`` and
``standalone-isolated-success-20261008.json``. Offline tests use an independently
generated synthetic RSA key and encryption vectors; their zero-signature test
certificates are not usable PLC credentials. Live support remains limited to
controller base-tag Description on the tested GuardLogix 5580 v37.13.

Optional Linx-assisted handshake
===============================

``examples.logix_privileged_metadata`` owns a fresh controller connection through
pycomm3. A separate 32-bit Python process loads the installed Linx ``cip.dll``
and calls its session provider; it does not borrow a FactoryTalk connection or
require View Studio to trigger traffic. No Gateway data endpoint or optional
Linx SDK Interface is needed for this path.

The observed sequence is:

1. pycomm3 registers a session and performs Forward Open with fresh identifiers.
2. Service ``0x4B``, class ``0x64``, instance 1 sends a signed public certificate.
   Its payload is 285 bytes for the metadata-capable certificate in frame 275.
3. The PLC returns ``80 00`` followed by a fresh 128-byte challenge.
4. The native provider decodes its private material internally, applies its RSA
   transform to the challenge and computes SHA-1 over the first 20 decoded
   bytes. The installed ``FTCrypt`` type library identifies the algorithms as
   ``CRYPTOALG_RSA`` (17) and ``CRYPTOALG_SHA1`` (3).
5. Service ``0x4C`` on the same class, instance and connection sends ``14 00``
   followed by the resulting 20-byte digest. Metadata reads then use that CID.

Offline, the provider reproduced the successful capture's completion exactly
from its challenge. Live, two independent fresh CIPDriver sessions returned
different challenges and different generated completions, both accepted with
status zero. Definition 1 and Description were then readable, with Description
returning ``HERE I AM WORLD``. Before authentication, the identical Description
request returned ``0x0F`` in each session. The packaged example also succeeded
using the ordinary LogixDriver connection and ``get_tag_description`` API.

The reusable certificate is distinct from the per-session completion. Changing
its final 128-byte signature or replacing its public key with a generated key
caused service ``0x4B`` to return ``0x09``. A generated key alone therefore does
not replace the signed certificate. Replaying an old service ``0x4C`` completion
also fails, as the earlier tests show.

Run the working reader from this VM's repository root::

    .venv\Scripts\python.exe -m examples.logix_privileged_metadata 10.137.22.8 --tag zzzTestTag --python32 .extended-properties-results\python32\python.exe --certificate .extended-properties-results\linx-metadata-certificate.bin

The local public certificate file contains only frame 275's request payload,
after its six-byte CIP service/path prefix. It is 285 bytes, SHA-256
``d5c9838defd4d4a9e6979e0b5a7d477333e51b353dc48df8309cf2b8163f9fae``.
The helper never exports or saves a private key. A certificate with different
numeric entries may authenticate without granting metadata access: the relay's
native connection with entry 3 alone still returned ``0x0F`` for Description,
whereas the connection with entries 3 and 11 succeeded.

For Python integration::

    from pathlib import Path
    from pycomm3 import LogixDriver
    from examples.logix_privileged_metadata import LinxSessionProvider, authenticate_connection

    certificate = Path(".extended-properties-results/linx-metadata-certificate.bin").read_bytes()
    python32 = Path(".extended-properties-results/python32/python.exe").resolve()
    with LogixDriver("10.137.22.8") as plc:
        with LinxSessionProvider(python32, certificate) as provider:
            authenticate_connection(plc, provider)
        print(plc.get_tag_description("zzzTestTag"))

Authentication applies to the current connected CIP connection. A reconnect
requires a new handshake. The provider is experimental and Windows-specific:
it calls an internal x86 ABI in the installed Linx 6.60 binary, rather than a
supported Rockwell SDK API. Its SHA-256 is pinned to
``dcc05896076158174222f18aaf807440e23922c6efc24ce6946c5478f8286356``;
other binaries are rejected before those calls. A Linx update requires renewed
ABI verification. The helper requires 32-bit Python 3.8 or newer and the
installed ``FTCrypt`` COM component. The parent pycomm3 process can be 64-bit.
This path remains dependent on installed Rockwell software.

Live evidence is saved locally as
``.extended-properties-results/native-session-success-20261008.json`` and
``logix-driver-native-auth-20261008.json``. Optional ``--output`` reports contain
setup traffic and should be kept local; an existing report is not overwritten.
The encrypted Python credential module contains the real certificate/key pair
in reversible encrypted form. Tracked examples and tests contain no real
unwrapped credentials or captured completions. Authentication tests use their
explicitly identified synthetic key fixture.

VM verification on 2026-10-08
============================

The VM reached the same controller with pycomm3, and FactoryTalk View Studio 16
displayed ``HERE I AM WORLD`` using the existing application and Linx shortcut.
No new successful metadata exchange was captured during that display test;
Linx may have served a cached value. Temporary local display edits were
discarded and the saved display matched its backup by SHA-256.

Four independent, fresh pycomm3 sessions were compared:

========================= ============= ================ ============= =============
Transport                 Identity read Metadata IDs     Definition 1  Description
========================= ============= ================ ============= =============
Unconnected SendRRData    Success       Success          Denied (0x0F) Denied (0x0F)
Normal Forward Open, 504  Success       Success          Denied (0x0F) Denied (0x0F)
Large Forward Open, 4000  Success       Success          Denied (0x0F) Denied (0x0F)
Captured FT settings      Success       Success          Denied (0x0F) Denied (0x0F)
========================= ============= ================ ============= =============

The last profile reproduced the capture's normal Forward Open transport settings:
vendor ID 77, 504-byte connection, RPI 2,000,000 microseconds, network parameters
``0x43F8``, timeout priority/ticks 6/155, timeout multiplier 2 and transport
``0xA3``. Connection identifiers remained fresh. Matching these settings alone
did not grant metadata access. These results rule out those transport differences
as a sufficient explanation for the Description denial on this controller.

The captured metadata connection used setup request frames **275 and 279**;
its Forward Open was frame 272. The successful Description read at frame 1182
used that same connection. The other setup pairs belonged to separate connections.

All three captured pairs were replayed on fresh pycomm3 connections, using the
FactoryTalk transport settings above. Each pair was tested in four independent
sessions: baseline, first request only, second request only and both in order.
All three pairs produced the following results:

=========================== ===================== ===================== =============
Setup case                  Service 0x4B          Service 0x4C          Description
=========================== ===================== ===================== =============
No setup                    Not sent              Not sent              Denied (0x0F)
First request only          Success               Not sent              Denied (0x0F)
Second request only         Not sent              Rejected (0x09)       Denied (0x0F)
Both requests in order      Success               Rejected (0x09)       Denied (0x0F)
=========================== ===================== ===================== =============

Each successful setup service ``0x4B`` reply contained ``80 00`` followed by
128 bytes. Those 128 bytes changed between fresh connections despite an
identical first request. The captured service ``0x4C`` request contained a
two-byte length followed by 20 bytes; it succeeded in the original FactoryTalk
session but returned ``0x09`` (Error in data segment or invalid attribute value)
in every replay, both alone and after the first request. Identity and metadata
directory reads still succeeded afterward; definition 1 remained denied.

This is evidence consistent with a session-dependent challenge/response. It
does not establish the cryptographic algorithm or prove that the 20-byte value
can be recomputed solely from packet contents. Replaying the captured completion
does not grant live metadata access. Reports are saved locally as
``.extended-properties-results/setup-replay-pair1.json`` through ``pair3.json``;
they contain the selected request bytes and all replies and are excluded from Git.

Service ``0x4B`` on class ``0x349``, outer instance zero, with no request payload
returned 25 UDINT definition identifiers without privileged setup. This is a
directory of IDs, not their definitions or tag values. Reading definition 1 with
the exact captured service ``0x53`` request was denied, just like Description.

The installed Linx name service exposes an internal ``IMapMetaData`` COM interface
with ``GetMetaDataValue``. Activation from both 32-bit and 64-bit Python failed
with ``0x80080005`` (Server execution failed); Windows recorded DCOM registration
timeouts. Its applicability to a third-party client remains unverified.

After the user installed FactoryTalk Linx Gateway, the VM had Gateway version
6.31.00 with Professional activation alongside Linx 6.60 and View Studio 16.
Gateway 6.31's installed help explicitly excludes Logix properties using the
``.@`` prefix. Rockwell's version history lists third-party extended-property
access as a Gateway 6.50 addition. Professional activation alone does not add
this capability to 6.31. No configured Gateway data endpoint or optional
``FTLinx_SDK.dll`` was found. Gateway configuration was also unavailable to the
signed-in FactoryTalk user; no authentication or access permissions were changed.

The user subsequently installed **Gateway 6.60.00.178** with Professional
activation. Its configuration UI reported that the optional SDK Interface was
not installed; SDK was disabled and no configured Gateway OPC UA data endpoint
was running. The separate OPC UA discovery service on port 4840 is not the
Gateway data endpoint. Global configuration also required a FactoryTalk application
selection. Installing the newer Gateway did not change the separate pycomm3
session's Description denial.

The official 6.60 installation media contained the SDK runtime and headers.
Loading the staged 32-bit DLL through Python succeeded, but ``DTL_INIT(0)``
returned ``244`` (``DTL_E_NO_LICENCE``). This result does not negate the observed
Professional activation: SDK initialization also depends on installation,
enablement and client access. An attempt to add the optional SDK feature failed
with Windows Installer error ``1625`` because silent installation could not
obtain elevation and the installer rejected the new source. No policy was changed.

Experimental reader
===================

``LogixDriver.get_tag_description`` issues the observed metadata read and returns
a ``Tag`` with type ``STRING``. Tag definitions must already be uploaded, as they
are by default. Permission errors remain explicit and do not cause retries::

    from pycomm3 import LogixDriver

    with LogixDriver("10.137.22.8") as plc:
        description = plc.get_tag_description("zzzTestTag")
        print(description.value, description.error)
        # Fresh session on the tested PLC: None Permission denied

This method currently accepts controller-scoped base tag names only. The HMI
expression ``{[plc]zzzTestTag.@Description}`` identifies a client shortcut and
property; pass only ``zzzTestTag`` to this API. Program tags, members, array
elements, inherited properties and properties other than Description remain
unverified. The method is optional and does not change ordinary tag reads or
automatically add metadata to tag discovery.

The default language selector ``0x007F`` returned the confirmed description.
The same captured query with ``0x0409`` succeeded but returned an empty metadata
page. An empty page is reported as no description for that query/language,
distinct from a configured string with length zero. ASCII text is verified;
``encoding`` is configurable, but the controller's non-ASCII encoding is unknown.

Decoding without a PLC
=====================

``pycomm3.logix_metadata`` contains the request builder and bounded response
codecs. Inputs to the codecs are payload pages **after** the standard CIP reply
header and its additional-status words. Multiple pages must be assembled before
record parsing, since a page boundary can fall inside a string.

The diagnostic can decode the confirmed, single-page CIP reply offline::

    python -m examples.logix_extended_properties decode "d300000001030000000000006b002704000000000000000000007f0001007111000000000000000000000000000001000f0048455245204920414d20574f524c44" --description

Use ``--enip`` when supplying one complete EtherNet/IP encapsulation frame.
Reassemble TCP first; an Ethernet packet or partial TCP segment is not a frame.
The framing decoder walks CPF items and skips WORD-counted additional status.

Observed wire format
====================

This is a capture-derived format, not a published Rockwell protocol
specification. Unknown context fields are preserved rather than assigned
unverified meanings.

Description request
-------------------

The complete captured request for Symbol instance 1063 is::

    5304210049032500000001010000000000000f00ff006b002704000000000000000000007f000100

The first ten bytes select service ``0x53``, class ``0x0349`` and **outer instance
zero** using a four-word path. The target Symbol instance is in the payload,
not the outer instance path. The remaining fields are little-endian::

    page header: <BBHI>     version=1, flags=1, reserved=0, offset=0
    selector:    <HHHIIIHH> 0x000F, 0x00FF, class=0x006B, instance=1063,
                           context=0, context=0, language=0x007F, selector=1

``build_description_request`` preserves these fixed selector bytes. They have
not been established as a general query language for other extended properties.

Response pages and records
--------------------------

Successful service replies begin with ``D3 00 00 00``: reply service, reserved,
general status and additional-status WORD count. After that standard header,
each metadata page begins with eight bytes::

    <BBHI> version=1, flags, reserved=0, byte offset

Flag ``1`` marks the first page and flag ``2`` marks the last page. The offset
counts preceding payload bytes, excluding the eight-byte page headers. Captured
definition 45 (DataExchangeId) has 490 payload bytes in its first page and a
continuation at offset 490. The continuation request clears the initial-page
flag. The definition's string is decoded only after joining both pages.

The observed Description payload contains::

    owner:    <HIIIH>  Symbol class, Symbol instance, two context UDINTs, language
    property: <HIIIIH> property identifier, record identifier, three context
                      UDINTs, value kind
    kind 1:   <H>      byte length, followed by exactly that many text bytes

For the confirmed result, property identifier is 1, record identifier is 4465,
value kind is 1 and text length is 15. This layout is not an ordinary Logix
STRING structure. The decoder rejects additional property records until their
layout has capture evidence.

Definition payloads contain repeated fields with this layout::

    field:  <HIIIH> identifier, three context UDINTs, value kind
    kind 1: <H>     byte length followed by text bytes
    kind 2: <HH>    CIP atomic type code, element count, followed by typed values

The capture verifies SINT (``0xC2``) and INT (``0xC3``) definition values. Fields
8, 9 and 22 correspond to LocalName, EnglishName and LocalizedDescription.
Schema definitions do not establish that every property can be retrieved for a
tag. In particular, this capture does not verify tag-specific EngineeringUnit,
Min, Max, State0 or State1 values.

Validation and reproducibility
==============================

``tests/offline/logix_metadata_capture.json`` contains only the relevant metadata
requests and response pages, with expected names and the user-confirmed value.
It excludes the binary session setup exchanges and unrelated captured traffic.
The source PCAPNG SHA-256 is
``e1f34513a4ea65de73c3bdcb111630d9f1afd8d67feb3c710ef8154541cc3b64``.

Offline tests cover the exact captured request, all 25 definitions, real
pagination, malformed lengths and headers, missing descriptions, wrong response
targets and permission errors. A mocked transport verifies the public method
against the captured reply and checks that it issues only service ``0x53``.
Synthetic encoding and Description continuation tests exercise error handling;
they do not constitute controller evidence for those cases.

Run the offline suite from the worktree root::

    python -m pytest tests/offline

The read-only diagnostic records complete request/reply bytes and numeric status
codes for explicitly selected tags::

    python -m examples.logix_extended_properties probe 10.137.22.8 --tag zzzTestTag --description-object --output .extended-properties-results/probe.json

It also tests standard symbolic property reads and Symbol attribute 11. Those
are exploratory probes, not alternative verified metadata protocols. Local
capture and analysis files under ``.extended-properties-results/`` are ignored
by Git.

The four-transport comparison can be reproduced separately::

    python -m examples.logix_metadata_transports 10.137.22.8 --symbol-instance 1063 --output .extended-properties-results/transports.json

Use the tag's **current** ``instance_id`` from tag discovery. Instance 1063 was
verified for ``zzzTestTag`` during this investigation; IDs may change after a
project download. This diagnostic accepts a direct controller IPv4 address,
not a CIP routing path. It saves complete read replies, including denials, and
does not decode failed reads as metadata values. It opens and closes each session
before testing the next profile and never sends the binary privilege setup.

The separate, explicit setup replay diagnostic requires tshark and a local capture::

    python -m examples.logix_captured_setup 10.137.22.8 --symbol-instance 1063 --capture "C:\Users\image\Documents\dump.pcapng" --setup-frames 275 279 --tshark "C:\Program Files\Wireshark\tshark.exe" --output .extended-properties-results/replay-new.json

Request frame pairs ``293 297`` and ``401 407`` select the other observed
connections. The diagnostic validates the two captured client request layouts
and their shared connection before sending them. It creates fresh session,
connection and sequence identifiers and never replays captured PLC replies.
The JSON includes setup request bytes; keep it local. This tool performs session
setup attempts and read queries, and is not called automatically by the driver.

For future property captures, Wireshark's **capture filter** is::

    host 10.137.22.8 and tcp port 44818

Its **display filter** uses different syntax::

    ip.addr == 10.137.22.8 && tcp.port == 44818

Capture inside the FactoryTalk VM on the adapter that reaches the PLC. Preserve
connection setup and record the already-configured property values. Additional
captures are needed before extending addressing, property selectors or encoding.

Documented alternative
======================

`Rockwell's FactoryTalk View documentation <https://www.rockwellautomation.com/en-us/docs/factorytalk-view/16-00-00/me-help-ditamap/what-is-an-hmi-tag-database-/about-data-sources/controller-tag-extended-properties.html>`_
documents the ``TagName.@Property`` client syntax. It does not specify a
third-party CIP request format.

`FactoryTalk Linx Gateway documentation <https://www.rockwellautomation.com/en-us/docs/factorytalk-linx-gateway/6-60/factorytalk-linx-gateway-help-ditamap/factorytalk-linx-gateway-help/basic-concepts/extended-tag-properties/lgx-extend-propty-ftae.html>`_
provides extended-property access through OPC UA, OPC DA or MQTT with Professional
activation, enabled properties and the shortcut's "Upload all extended tag
properties" option. This requires **Gateway 6.50 or newer**, as documented in
`Rockwell's version history <https://www.rockwellautomation.com/en-us/products/software/factorytalk/version-history.html>`_.
The ordinary FactoryTalk Linx version does not establish the installed Gateway
version. Logix extended properties are not supported by the Gateway's Standalone
Data Service; use a FactoryTalk application and its Linx shortcut. A Gateway
integration is a separate option if standalone CIP session access cannot be
established.

``examples.linx_gateway_properties`` provides an optional, read-only Python
client for the Gateway's scalar OPC UA namespace. It requires ``asyncua``; the
core pycomm3 package does not depend on it. First configure the Gateway's
application scope and endpoint, and determine its namespace index and the area
path relative to that scope. Use the endpoint's configured security policy,
certificate trust and FactoryTalk user authentication. This example does not
configure or weaken server security.

Install the optional client dependency, then inspect the advertised endpoints::

    python -m pip install asyncua
    python -m examples.linx_gateway_properties discover opc.tcp://localhost:4990

Browse one level of the namespace or read an explicitly selected property::

    python -m examples.linx_gateway_properties browse opc.tcp://localhost:4990
    python -m examples.linx_gateway_properties read opc.tcp://localhost:4990 --shortcut plc --tag zzzTestTag --namespace-index 2 --property Description --output .extended-properties-results/gateway-description.json

These commands assume the shown endpoint, empty relative area and namespace
index 2 are actually configured. Use ``--area`` for a scope-relative area,
``--security`` for an asyncua security string, and ``--username`` when user
authentication is required. Passwords are prompted and excluded from reports.
The helper records each property's UA status and accepts values only when that
status is Good, distinguishing a valid empty string from a missing property.

The helper was verified against a local OPC UA test server for discovery,
browsing, a valid Description, an empty EngineeringUnit and a missing property.
Live reads through Rockwell Gateway remain unverified pending Gateway
configuration. Using Gateway does not authenticate the separate
pycomm3 CIP session or resolve its ``0x0F`` response.

The `FactoryTalk Linx SDK reference manual <https://literature.rockwellautomation.com/idc/groups/literature/documents/rm/lnxsdk-rm001_-en-e.pdf>`_
documents native APIs through a 32-bit DLL and requires Linx Gateway activation.
Python can load the official DLL and call initialization through ``ctypes``;
live SDK messaging remains unverified because initialization was rejected on
this VM. The SDK is a separate installed option. Its
generic CIP messaging API alone has not been verified to create the privileged metadata
connection needed here; installing it should not be presented as a proven fix
for pycomm3's permission error.
