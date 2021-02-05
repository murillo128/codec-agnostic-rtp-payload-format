--
docname: draft-codec-agnostic-rtp-payload-format-00
title: Codec agnostic RTP payload format for video
category: std
ipr: trust200902
area: ART
workgroup: AVTCORE
stand_alone: yes

pi: [toc, sortrefs, symrefs]

author:
-
  ins: S. Garcia Murillo
  name: Sergio Garcia Murillo
  org: CoSMo
  email: sergio.garcia.murillo@cosmosoftware.io

-
  ins: Y. Fablet
  name: Youenn Fablet
  org: Apple Inc.
  email: youenn@apple.com 
  
-
  ins: A. Gouaillard
  name: Alex Gouaillard
  org: CoSMo
  email: alex.gouaillard@cosmosoftware.io



normative:
  RFC2119:
  RFC3550:
  RFC3551:
  RFC3711:
  RFC4566:
  RFC5285:
  RFC7656:
  RFC8285:

informative:
  RFC2198:
  RFC4588:
  RFC5109:
  RFC6464:
  RFC6465:
  RFC6904:
  RFC6184:
  SFrame:
    target: https://tools.ietf.org/html/draft-omara-sframe
    title: Secure Frame (SFrame)
  WebRTCInsertableStreams:
    target: https://w3c.github.io/webrtc-insertable-streams
    title: WebRTC Insertable Media using Streams

--- abstract

RTP Media Chains usually rely on piping encoder output directly to packetizers. Media packetization formats often support a specific codec format and optimize RTP packets generation accordingly.
With the development of Selective Forward Unit (SFU) solutions, RTP Media Chains used in WebRTC solutions are increasingly relying on application-specific transforms that seat between encoder and packetizer on one end and between depacketizer and decoder on the other end. These transforms are typically encrypting media content so that the media content is not readable from the SFU, for instance using {{SFrame}} or {{WebRTCInsertableStreams}}.
In that context, RTP packetizers can no longer expect to use packetization formats that mandate media content to be in a specific codec format.
This document provides a solution to that problem by describing a generic RTP packetization format that can be used on any media content, and how to negotiate use of this format.
This document also describes a solution to allow SFUs to continue performing packet routing on top of this generic RTP packetization format.

--- middle

Introduction
============

As per Figure 1 of {{RFC7656}}, a Media Packetizer transforms a single Encoded Stream into one or several RTP packets.
The Encoded Stream is coming straight from the Media Encoder and is expected to follow the format produced by the Media Encoder.
A number of Media Packetizer formats have been designed to process a specific format produced by Media Encoder.
For instance {{6184}} is dedicated to the processing of content produced by H.264 Media Encoders, and generates packets following NALUs organization.

WebRTC applications are increasingly deploying end-to-end encryption solutions on top of RTP Media Chains.
End-to-end encryption is implemented by inserting application-specific Media Transformers between Media Encoder and Media Packetizer on the sending side, and between Media Depacketizer and Media Decoder on the receiving side, as described in Figure 1 and Figure 2.
To support end-to-end encryption, Media Transformers can use the {{SFrame}} format.
In browsers, Media Transformers are implemented using {{WebRTCInsertableStreams}}, for instance by injecting Javacript code provided by web pages.

```
                Physical Stimulus
                      |
                      V
           +----------------------+
           |     Media Capture    |
           +----------------------+
                      |
                 Raw Stream
                      V
           +----------------------+
           |     Media Source     |<-- Synchronization Timing
           +----------------------+
                      |
                Source Stream
                      V
           +----------------------+
           |    Media Encoder     |
           +----------------------+
                      |
                Encoded Stream 
                      V
           +----------------------+
           |   Media Transformer  |<-- NEW: application-specific transform
           +----------------------+         (e.g. SFrame Encryption)
                      |
              Transformed Stream    +------------+
                      V             |            V
           +----------------------+ | +----------------------+
           |   Media Packetizer   | | | RTP-Based Redundancy |
           +----------------------+ | +----------------------+
                      |             |            |
                      +-------------+  Redundancy RTP Stream
               Source RTP Stream                 |
                      V                          V
           +----------------------+   +----------------------+
           |  RTP-Based Security  |   |  RTP-Based Security  |
           +----------------------+   +----------------------+
                      |                          |
              Secured RTP Stream   Secured Redundancy RTP Stream
                      V                          V
           +----------------------+   +----------------------+
           |   Media Transport    |   |   Media Transport    |
           +----------------------+   +----------------------+
```
             Figure 1: Sender Side Concepts in the Media Chain
             With Application-level Media Transform

These RTP packets are sent over the wire to a receiver media chain matching the sender side, reaching the Media Depacketizer that will reconstruct the Encoded Stream before passing it to the Media Decoder.
```
          +----------------------+   +----------------------+
          |   Media Transport    |   |   Media Transport    |
          +----------------------+   +----------------------+
            Received |                 Received | Secured
            Secured RTP Stream       Redundancy RTP Stream
                     V                          V
          +----------------------+   +----------------------+
          | RTP-Based Validation |   | RTP-Based Validation |
          +----------------------+   +----------------------+
                     |                          |
            Received RTP Stream   Received Redundancy RTP Stream
                     |                          |
                     |     +--------------------+
                     V     V
          +----------------------+
          |   RTP-Based Repair   |
          +----------------------+
                     |
            Repaired RTP Stream
                     V
          +----------------------+
          |  Media Depacketizer  |
          +----------------------+
                     |
         Received Transformed Stream
                     V
          +----------------------+
          |   Media Transformer  |<-- NEW: application-specific transform
          +----------------------+         (e.g. SFrame Decryption)
                     |
           Received Encoded Stream
                     V
          +----------------------+
          |    Media Decoder     |
          +----------------------+
                     |
           Received Source Stream
                     V
          +----------------------+
          |      Media Sink      |--> Synchronization Information
          +----------------------+
                     |
            Received Raw Stream
                     V
          +----------------------+
          |     Media Render     |
          +----------------------+
                     |
                     V
             Physical Stimulus
```
             Figure 2: Receiver Side Concepts in the Media Chain
             With Application-level Media Transform
 
This generic packetization does not change how the mapping between one or several encoded or dependant streams are mapped to the RTP streams or how the synchronization sources(s) (SSRC) are assigned. 

Given the use of post-encoder application-specific transforms, the whole Media Chain needs to be made aware of it.
This includes the sender post-transform Media Chain, Media Transport intermediaries (SFUs typically) and receiver pre-transform Media Chain.

As these transforms can alter Encoded Streams in any possible way, the use of codec-specific Media Packetizers like {{RFC6184}} on Transformed Stream may be suboptimal on sender side.
It may also be problematic on the receiving side in case codec-specific processing is done prior the Media Transformer.
Media Transport intermediaries are often looking at the Media Content itself to fuel their packet selection algorithms.

Goals
=====

The objective of this document is to support inserting any application-specific transform between encoders and packetizers in the Media Chain.
For that purpose, this document will:
1. Provide a generic packetization format that supports any media content (compressed audio, compressed video, encrypted content...) that allows reuse of existing RTP mechanisms in place in WebRTC applications such as RTX, RED or FEC.
2. Provide a way to negotiate use of the generic packetization format between sender and receiver, with minimum impact on existing negotiation approaches.
3. Provide a side-channel information so that network intermediaries (SFU in particular) can do their existing packet routing strategies without inspecting the media content.

RTP Packetization
=================

A generic packetizer, by design, is not expected to understand the format of the media to transmit. The unit used by the packetizer to do processing is called a frame in the remainder of the document.

It is the responsibility of the application using the packetizer to group media content in meaningful frames. In the common case of a video codec, the packetizer frame is the frame in byte format (h264 annex b for example) generated by the encoder. 

If the application wants to transform encoded content, the application needs to split the encoded content into frames prior the transform.
Each frame is then transformed independently, for instance encrypted using {{SFrame}}.
The content of each transformed frame is then processed by the packetizer.

In the case of a video codec supporting spatial scalability, each spatial layer MUST be split in its own frame by the application before passing it to the packetizer. 

When the packetizer receives a frame from the application, it MUST fragment the frame content in multiple RTP packets to ensure packets do not exceed the network maximum transmission unit. The content of the frame will be treated as a binary blob by the packetizer, so the decision about the boundaries of each fragment is decided arbitrarily by the packetizer. The packetizer or any relying server MUST NOT modify the frame content and concatenating the RTP payload of the RTP packets for each frame MUST produce the exact binary content of the input frame content.

The marker bit of each RTP packet in a frame MUST be set according to the audio and video profiles specified in [[rfc3551]].

The spatial layer frames are sent in ascending order, with the same RTP timestamp, and only the last RTP packet of the last spatial layer frame will have the marker bit set to 1.

Payload Multiplexing
====================

In order to reduce the number of payload type in the SDP exchange, a single payload type code for the generic packetization can be used for all negotiated media formats.
That requires to identify the original payload type code of the frame negotiated media format, called the associated payload type (APT) hereunder.
The APT value is the payload type code of the associated format passed to the rtp generic packetizer before any transformation is applied.

The APT value is sent in a dedicated header extension.
The payload of this header extension can be encoded using either the one-byte or two-byte header defined in [[RFC5285]].
Figures 3 and 4 show examples with each one of these examples.

```
                    0                   1
                    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |  ID   | len=0 |S|     APT     |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
Figure 3: Frame Associated Payload Type Encoding Using the One-Byte Header Format

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |      ID       |     len=1     |S|     APT     |    0 (pad)    |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
Figure 4: Frame Associated Payload Type Encoding Using the Two-Byte Header Format

The APT value is the associated payload type value.
The S bit indicates if the media stream can be forwarded safely starting from this RTP packet.
Typically, it will be set to 1 on the first rtp packet of an I-Frame video frame and in all rtp audio packets.

Receivers MUST be ready to receive RTP packets with different associated payload types in the same way it would receive different payload type codes on the RTP packets.

The URI for declaring this header extension in an extmap attribute is "urn:ietf:params:rtp-hdrext:associated-payload-type".

SDP Negotiation
===============

A payload type for the negotiated format, the generic payload type and the associated payload type header extension MUST be negotiated in the SDP O/A for each media type in order to be able to use the RTP generic packetization.
Only the payload types negotiated are allowed to be used as associated payload types.
Figure 5 illustrates a SDP that negotiates video using either VP8 or VP9 codecs with the possibility to use the generic packetization.
RTX and FEC are also negotiated and will be applied normally.

```
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=setup:actpass
a=mid:1
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=extmap:4 urn:ietf:params:rtp-hdrext:associated-payload-type
a=sendrecv
a=rtpmap:96 vp9/90000
a=rtpmap:97 vp8/90000
a=rtpmap:98 generic/90000
a=rtpmap:99 rtx/90000
a=fmtp:99 apt=96
a=rtpmap:100 rtx/90000
a=fmtp:100 apt=97
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=98
```
Figure 5: SDP example negotiating the generic payload type and related header extension

Q: What frequency should we use for audio? One per the different frequencies of each audio codec?

Layer Selection
===============

SFUs need to have a basic understanding of each frame they receive so they can decide to forward it or not and to which endpoint.
They might need similar information to support media content recording.
This information is either generic to a group of frame (called a stream hereafter) or specific to each frame.

The information is transmitted as a RTP header extension as the RTP packet payload should be treated as opaque by the SFU.
This is especially necessary if the payload is transformed by {{SFrame}} to support end-to-end encryption.
The amount of information should be limited to what is strictly necessary to the SFU task since it is not always as trusted as individual peers.

For audio, configuration information such as Opus TOC might be useful.
For video, the following configuration information might include:
- Stream configuration information: resolution, quality, frame rate...
- Codec specific configuration information: codec profile like profile_idc...
- Frame specific information: whether the stream is decodable when starting from this frame, whether the frame is skippable...

For video content, this information can be sent using a Dependency Descriptor header extension.
In that case, the first RTP packet of the frame will have its start_of_frame equal to 1 and the last packet will have its end_of_frame equal to 1.

SFU Packet Selection
====================

SFUs need to have a basic understanding of each frame they receive so they can decide to forward it or not and to which endpoint.
They might need similar information to support media content recording.
This information is either generic to a group of frame (called a stream hereafter) or specific to each frame.

The information is transmitted as a RTP header extension as the RTP packet payload should be treated as opaque by the SFU.
This is especially necessary if the payload is end-to-end encrypted.
The amount of information should be limited to what is strictly necessary to the SFU task since it is not always as trusted as individual peers.

For audio, configuration information such as Opus TOC might be useful.
For video, the following configuration information might include:
- Stream configuration information: resolution, quality, frame rate...
- Codec specific configuration information: codec profile like profile_idc...
- Frame specific information: whether the stream is decodable when starting from this frame, whether the frame is skippable...

For video content, this information can be sent using a Dependency Descriptor header extension.
In that case, the first RTP packet of the frame will have its start_of_frame equal to 1 and the last packet will have its end_of_frame equal to 1.

Redundancy Technniques Considerations
=====================================

The solution described in this document is expected to integrate well with the existing RTP ecosystem.
This section describes how the generic packetizer can be used jointly with existing techniques that allow to mitigate unreliable transports.

Retransmission Techniques
-------------------------

{{RFC4588}} defines a retransmission payload format (RTX) that can be used in case of packet loss.
As defined in {{RFC4588}}, RTX is able to handle any payload format, including the format described in this document.
Given RTX preserves both RTP packet payload and headers, the receiver will be able to identify the payload type of the recovered packet and whether generic packetization is used.
RTX will also allow recovering RTP header extensions that convey information on the media content itself.

Forward Error Correction (FEC) Techniques
-----------------------------------------

FEC is another technique used in RTP Media Chains to protect media content against packet loss.
{{RFC5109}} defines such a payload format used to transmit FEC for specific packets protection.

FEC may protect some parts of the media content more than others. For instance, intra frame encoded data or important network abstraction layer units (NALUs) like SPS/PPS may be more protected.
With a post-encoder transform and the use of a generic packetization, the granularity of the recovery mechanism is no longer at the NALU level but at the level of the frame generated by the post-encoder transform.
In case a SVC codec is used, each spatial layer will be processed as an independent frame. In that case, base layers can be protected more heavily than higher resolution layers.

Redundand Audio Data Techniques
-------------------------------

As defined in {{RFC7656}} RTP-based redundancy is defined here as a transformation that generates redundant or repair packets sent out as a Redundancy RTP Stream to mitigate Network Transport impairments, like packet loss and delay. 
 
{{RFC2198}} defines a payload format for sending the same audio data encoded multiple times at different quality levels.
This allows to use a lower quality encoding of the audio data, should the higher quality encoding of the audio data is lost during the transmission.

If a Media Transformation is in use, both the primary and redundant encoding must be transformed independently and the redundant packet created normally. As the RTP headers present in the redundant packet are only applicable to the primary encoding, if the payload type for a redundant encoding block is mapped to the generic packetizer, the value of the associated payload type for the primary encoding is applied to the redundant encoding block as well.

Security Considerations
=======================

IANA Considerations
===================

Two new media subtypes have been registered with IANA, as described in this section.  This registration is done using the registration template {{REF}} and following [[RFC3555]].

## Registration of audio/generic

   Type name: audio

   Subtype name: generic

   Required parameters: none

   Optional parameters: none

   Encoding considerations: This format is framed (see Section 4.8 in the template document [3]) and contains binary data.

   Security considerations: TBD.

   Interoperability considerations: TBD

   Published specification: TBD.

   Applications that use this media type: TBD.
   
   Additional information: none

   Intended usage: COMMON

   Restrictions on usage: TBD

   Author:

   Change controller:
      
# Registration of video/generic

   Type name: video

   Subtype name: generic

   Required parameters: none

   Optional parameters: none

   Encoding considerations: This format is framed (see Section 4.8 in the template document [3]) and contains binary data.

   Security considerations: TBD.

   Interoperability considerations: TBD

   Published specification: TBD.

   Applications that use this media type: TBD.
   
   Additional information: none

   Intended usage: COMMON

   Restrictions on usage: TBD

   Author:

   Change controller:


--- back
