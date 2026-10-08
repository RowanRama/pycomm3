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
Ordinary tag reads succeed through that connection. Therefore payload decoding
is verified, but standalone live retrieval on this controller remains unresolved.

The successful captured session contains earlier binary exchanges using class
``0x64`` and services ``0x4B``/``0x4C``. Those exchanges may explain the privilege
difference, but their authentication role and format have not been established.
They were not replayed. This implementation does not authenticate a session or
change controller permissions. The capture contains three such exchange pairs
(request/reply packets 275/277 and 279/281, 293/295 and 297/299, and 401/404 and
407/409). Their request payload lengths are 285/22, 281/22 and 281/22 bytes.
Class 0x64 is named ``program_name`` in the existing CIP object library; that
name alone does not establish these vendor services' meaning.

All live tests used read requests and connection lifecycle operations. No tag
values, clock, controller mode, project or permissions were changed.

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
properties" option. A Gateway integration is a separate option if standalone
CIP session access cannot be established.
