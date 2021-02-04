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
  RFC7656:
  RFC8285:

informative:
  RFC2198:
  RFC4588:
  RFC5109:
  RFC6464:
  RFC6465:
  RFC6904:
  SFrame:

--- abstract



--- middle

Introduction
============

The objective of this spec is to create a generic RTP packetization format that can be used with any audio or video codec or encrypted content and that allows SFUs to perform routing and layer selection without requiring access to the codec payload.

Media packetization and depacketization
=======================

As per {{RFC7656}} the generic packetizer will define a Media Packetizer that transforms a single Encoded Stream into one or several RTP packets.

  
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
           |     Media Source     |<- Synchronization Timing
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
           |    Media Encryptor   |
           +----------------------+
                      |
                Encrypted Stream    +------------+
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

             Figure 1: Sender Side Concepts in the Media Chain with Application-level Media Encryption 
             
```

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
           Received Encrypted Stream
                     V
          +----------------------+
          |    Media Decrypter   |          
          +----------------------+
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

            Figure 2: Receiver Side Concepts of the Media Chain with Application-level Media Encryption 

```
 
This generic packetization does not change how the mapping between one or several encoded or dependant streams are mapped to the RTP streams or how the synchronization sources(s) (SSRC) are assigned. 

The generic packetizer only supports Single RTP stream on a Single media Transport (SRST) when Scalale Video Coding (SVC) is in use.

The other elements on the Media Chain, like RTP-Based Redundancy, are not affected by the usage of the generic packetizer. 

RTP Packetization
=================

A generic packetizer, by design, is not expected to understand the format of the media to transmit. The unit used by the packetizer to do processing is called a frame in the remainder of the document.

It is the responsibility of the application using the packetizer to group media content in meaningful frames. In the common case of a video codec, the packetizer frame is the frame in byte format (h264 annex b for example) generated by the encoder. 

If the application wants to encrypt content, the application needs to split media content into frames before encryption as explained above and, using {{SFrame}} for example, encrypt each frame content independently. The content of each encrypted frame will then be fed to the packetizer.

In the case of a video codec supporting spatial scalability, each spatial layer MUST be split in its own frame by the application before passing it to the packetizer. 

When the packetizer receives a frame from the application, it MUST fragment the frame content in multiple RTP packets to ensure packets do not exceed the network maximum transmission unit. The content of the frame will be treated as a binary blob by the packetizer, so the decision about the boundaries of each fragment is decided arbitrarily by the packetizer. The packetizer or any relying server MUST NOT modify the frame content and concatenating the RTP payload of the RTP packets for each frame MUST produce the exact binary content of the input frame content.

The marker bit of each RTP packet in a frame MUST be set according to the audio and video profiles specified in [[rfc3551]].

The spatial layer frames are sent in ascending order, with the same RTP timestamp, and only the last RTP packet of the last spatial layer frame will have the marker bit set to 1.

Payload Multiplexing
====================

In order to reduce the number of payload type codes on the SDP exchange, a single payload type code for the generic packetization can be used for each media type. That requires to identify the original payload type code for the negotiated media format that the frame belongs to.

The associated payload type will be sent in a header extension. The payload of associated payload header extension element can be encoded using either the one-byte or two-byte header defined in [[RFC5285]].

Figures 1 and 2 show sample encoding with each of these header formats.

```
                    0                   1
                    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                   |  ID   | len=0 |S|     APT     |
                   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
Figure 1: Sample Associated Payload Type Encoding Using the One-Byte Header Format

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |      ID       |     len=1     |S|     APT     |    0 (pad)    |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
Figure 2: Sample Associated Payload Type Encoding Using the Two-Byte Header Format

The APT value is the value is the payload type code of the associated format passed to the rtp generic packetizer before any transformation is applied. The S bit indicates if the media stream can be forwarded safely starting from this RTP packet. Typically, it will be set to 1 on the first rtp packet of an I-Frame video frame and in all rtp audio packets.

Receivers MUST be ready to receive RTP packets with different associated payload types in the same way it would receive different payload type codes on the RTP packets.

The URI for declaring this header extension in an extmap attribute is "urn:ietf:params:rtp-hdrext:associated-payload-type".
   
Layer Selection
===============

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

SDP Negotiation
===============

A payload type format and the payload type codec and the associated payload type header extension MUST be negotiated in the SDP O/A for each media type in order to be able to use the RTP generic packetization. Only the payload types negotiated are allowed to be used as associated payload types.

RTX and FEC procedures applies normally.

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

Q: What frequency should we use for audio? One per the different frequencies of each audio codec?

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

{{RFC2198}} defines a payload format for sending the same audio data encoded multiple times at different quality levels.
This allows to use a lower quality encoding of the audio data, should the higher quality encoding of the audio data is lost during the transmission.

RTP Media Chains can integrate RED as a codec and apply transforms after the RED encoder and before the RED decoder.
In that case, given the payload type of RED blocks will be transformed, a SFU may not be able to modify it before sending it to other recipients.
It is important in that case for the SFU to negotiate the same payload type used as the RED block payload type between sender and all receivers.

RTP Media Chains can also be designed so that RED sits after the post-encoder transform and before the pre-decoder transform.
In that case, the SFU is able to update RED block payload types.

If RTP header extensions are used to provide information on the media content itself, these RTP header extensions can be sent as RTP header extensions of the RED packet.
These RTP header extensions are expected to be applicable to any of the audio data encodings transmitted as part of the RED packet.

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
