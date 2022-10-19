---
docname: draft-murillo-whep-01
title: WebRTC-HTTP Egress Protocol (WHEP)
abbrev: whep
category: info

ipr: trust200902
area: Security
keyword: Internet-Draft WebRTC

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:

 -
    ins: S. Murillo
    name: Sergio Garcia Murillo
    organization: Millicast
    email: sergio.garcia.murillo@cosmosoftware.io
    
 -
    ins: C. Chen
    name: Cheng Chen
    organization: ByteDance
    email: webrtc@bytedance.com
    
normative:
  FETCH:
    author:
      org: WHATWG
    title: Fetch - Living Standard
    target: https://fetch.spec.whatwg.org
      
--- abstract

This document describes a simple HTTP-based protocol that will allow WebRTC-based viewers to watch content from streaming services and/or Content Delivery Networks (CDNs) or WebRTC Transmission Network (WTNs).

--- middle

# Introduction

The IETF RTCWEB working group standardized JSEP ({{!RFC8829}}), a mechanism used to control the setup, management, and teardown of a multimedia session. It also describes how to negotiate media flows using the Offer/Answer Model with the Session Description Protocol (SDP) {{!RFC3264}} as well as the formats for data sent over the wire (e.g., media types, codec parameters, and encryption). WebRTC intentionally does not specify a signaling transport protocol at application level. This flexibility has allowed the implementation of a wide range of services. However, those services are typically standalone silos which don't require interoperability with other services or leverage the existence of tools that can communicate with them.

While some standard signaling protocols are available that can be integrated with WebRTC, like SIP {{?RFC3261}} or XMPP {{?RFC6120}}, they are not designed to be used in broadcasting/streaming services, and there also is no sign of adoption in that industry. RTSP {{?RFC7826}}, which is based on RTP and may be the closest in terms of features to WebRTC, is not compatible with the SDP offer/answer model {{!RFC3264}}.

So, currently, there is no standard protocol designed for consuming media from streaming service using WebRTC.


There are many situations in which the lack of a standard protocol for consuming media from streaming service using WebRTC has become a problem:
  
- Interoperability between WebRTC services and products.
- Reusing player software which can be integrated easily.
- Integration with Dynamic Adaptive Streaming over HTTP (DASH) for offering live streams via WebRTC while offering a time-shifted version via DASH.
- Playing WebRTC streams on devices that don't support custom javascript to be run (like TVs).

This document mimics what has been done  the WebRTC HTTP Ingest Protocol (WHIP) {{?I-D.draft-ietf-wish-whip}} for ingestion and specifies a simple HTTP-based protocol that can be used for consuming media from a streaming service using WebRTC.

# Terminology

{::boilerplate bcp14-tagged}

- WHEP Player: WebRTC media player that acts as a client of the WHEP protocol by receiving and decoding the media from a remote Media Server.
- WHEP Endpoint: Egress server receiving the initial WHEP request.
- WHEP Endpoint URL: URL of the WHEP endpoint that will create the WHEP resource.
- Media Server: WebRTC Media Server or consumer that establishes the media session with the WHEP player and delivers the media to it.
- WHEP Resource: Allocated resource by the WHEP endpoint for an ongoing egress session that the WHEP player can send requests for altering the session (ICE operations or termination, for example).
- WHEP Resource URL: URL allocated to a specific media session by the WHEP endpoint which can be used to perform operations such as terminating the session or ICE restarts.


# Overview

The WebRTC-HTTP Egress Protocol (WHEP) uses an HTTP POST request to perform a single-shot SDP offer/answer so an ICE/DTLS session can be established between the WHEP Player and the streaming service endpoint (Media Server).

Once the ICE/DTLS session is set up, the media will flow unidirectionally from Media Server to the WHEP Player. In order to reduce complexity, no SDP renegotiation is supported, so no tracks or streams can be added or removed once the initial SDP offer/answer over HTTP is completed.

~~~~~
                                                                               
 +-------------+    +---------------+ +--------------+ +---------------+
 | WHEP Player |    | WHEP Endpoint | | Media Server | | WHEP Resource |
 +--+----------+    +---------+-----+ +------+-------+ +--------|------+
    |                         |              |                  |       
    |                         |              |                  |       
    |HTTP POST (SDP Offer)    |              |                  |       
    +------------------------>+              |                  |       
    |201 Created (SDP answer) |              |                  |       
    +<------------------------+              |                  |       
    |          ICE REQUEST                   |                  |       
    +--------------------------------------->+                  |       
    |          ICE RESPONSE                  |                  |       
    |<---------------------------------------+                  |       
    |          DTLS SETUP                    |                  |       
    |<======================================>|                  |       
    |          RTP/RTCP FLOW                 |                  |       
    +<-------------------------------------->+                  |       
    | HTTP DELETE                                               |       
    +---------------------------------------------------------->+       
    | 200 OK                                                    |       
    <-----------------------------------------------------------x       
                                                                               
~~~~~
{: title="WHEP session setup and teardown"}


Alternatively, there are cases in which the WHEP Player may wish the service to provide the SDP offer (for example to avoid setting up an audio and video session when only audio is supported), so in this case the initial HTTP POST request will not contain a body and the response will contain the SDP offer from the service instead. The WHEP Player will have to provide the SDP answer in a subsequent HTTP PATCH request to the WHEP resource.

~~~~~
                                                                               
 +-------------+    +---------------+ +--------------+ +---------------+
 | WHEP Player |    | WHEP Endpoint | | Media Server | | WHEP Resource |
 +--+----------+    +---------+-----+ +------+-------+ +--------|------+
    |                         |              |                  |       
    |                         |              |                  |       
    |HTTP POST (empty)        |              |                  |       
    +------------------------>+              |                  |       
    |201 Created (SDP offer)  |              |                  |       
    +<------------------------+              |                  | 
    | HTTP PATCH (SDP answer)                |                  |       
    +---------------------------------------------------------->+       
    | 200 OK                                 |                  |       
    <-----------------------------------------------------------x           
    |          ICE REQUEST                   |                  |       
    +--------------------------------------->+                  |       
    |          ICE RESPONSE                  |                  |       
    |<---------------------------------------+                  |       
    |          DTLS SETUP                    |                  |       
    |<======================================>|                  |       
    |          RTP/RTCP FLOW                 |                  |       
    +<-------------------------------------->+                  |       
    | HTTP DELETE                                               |       
    +---------------------------------------------------------->+       
    | 200 OK                                                    |       
    <-----------------------------------------------------------x       
                                                                               
~~~~~
{: title="WHEP session setup and teardown"}

# Protocol Operation

## SDP offer generated by the WHEP player
In order to set up a streaming session, the WHEP Player will generate an SDP offer according to the JSEP rules and perform an HTTP POST request to the configured WHEP Endpoint URL.

The HTTP POST request will have a content type of "application/sdp" and contain the SDP offer as the body. The WHEP Endpoint will generate an SDP answer and return a "201 Created" response with a content type of "application/sdp", the SDP answer as the body, and a Location header field pointing to the newly created resource.

The SDP offer SHOULD use the "recvonly" attribute and the SDP answer MUST use "sendonly" the attribute.

~~~~~
POST /whep/endpoint HTTP/1.1
Host: whep.example.com
Content-Type: application/sdp
Content-Length: 1326

v=0
o=- 5228595038118931041 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=msid-semantic: WMS
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=recvonly
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

HTTP/1.1 201 Created
ETag: "xyzzy"
Content-Type: application/sdp
Accept-Patch: application/trickle-ice-sdpfrag
Content-Length: 1400
Location: https://whep.example.org/resource/id

v=0
o=- 1657793490019 1 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-lite
a=msid-semantic: WMS *
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=sendonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
~~~~~
{: title="HTTP POST and PATCH doing SDP O/A example"}

## SDP offer generated by the WHEP endpoint

If the WHEP player prefers the WHEP Endpoint to generate the SDP offer, the WHEP Player will send a POST request without HTTP BODY and an Accept HTTP header of "application/sdp" to the configured WHEP endpoint URL.

The WHEP Endpoint will generate an SDP offer according to the JSEP rules and return a "201 Created" response with a content type of "application/sdp", the SDP offer as the body, a Location header field pointing to the newly created resource and an Expire header indicating the maximum time that the WHEP player is allowed to send the SDP answer to the WHEP Resource.

The WHEP Player MUST generate an SDP answer to SDP offer provided by the WHEP Endpoint and send an HTTP PATCH request to the URL provided in the Location header for the WHEP Resource. The HTTP PATCH request will have a content type of "application/sdp" and contain the SDP answer as the body. If the SDP offer is not accepted by the WHEP player, it MUST perform an HTTP DELETE operation for terminating the session to the WHEP Resource URL.

The SDP offer SHOULD use the "sendonly" attribute and the SDP answer MUST use "recvonly" attribute in this case.

~~~~~
POST /whep/endpoint HTTP/1.1
Host: whep.example.com
Content-Length: 0
Accept: application/sdp

HTTP/1.1 201 Created
Content-Type: application/sdp
Content-Length: 1400
Location: https://whep.example.com/resource/id
Expires: Wed, 27 Jul 2022 07:28:00 GMT

v=0
o=- 5228595038118931041 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=msid-semantic: WMS
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=sendonly
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendonly
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

PATCH /resource/id HTTP/1.1
Host: whep.example.com
Content-Type: application/sdp
Content-Length: 1326

v=0
o=- 1657793490019 1 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-lite
a=msid-semantic: WMS *
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

HTTP/1.1 204 No Content
ETag: "xyzzy"
~~~~~
{: title="HTTP POST and PATCH doing SDP O/A example"}

If the WHEP Resource does not receive an HTTP PATCH request before the time indicated in the Expire header HTTP POST response, it SHOULD delete the resource and respond with a 404 Not Found response to any request on the WHEP Resource URL received afterwards.


## Common procedures

The WHEP Resource COULD require a live publishing to be happening in order to allow a WHEP Players to start viewing a stream.
In that case, the WHEP Resource SHALL return a "409 Conflict" response to the POST request issued by the WHEP Client with a Retry-After header indicating the number of seconds before sending a new request.
WHEP Players MAY periodically try to connect to the WHEP Resource with exponential backoff period with an initial value of the Retry-After header value in the "409 Conflict" response.

Once a session is set up, ICE consent freshness {{!RFC7675}} will be used to detect abrupt disconnection and DTLS teardown for session termination by either side.

To explicitly terminate a session, the WHEP Player MUST perform an HTTP DELETE request to the resource URL returned in the Location header field of the initial HTTP POST. Upon receiving the HTTP DELETE request, the WHEP resource will be removed and the resources freed on the Media Server, terminating the ICE and DTLS sessions.

A Media Server terminating a session MUST follow the procedures in {{!RFC7675}} section 5.2 for immediate revocation of consent.

The WHEP Endpoints MUST return an "405 Method Not Allowed" response for any HTTP GET, HEAD or PUT requests on the endpoint URL in order to reserve its usage for future versions of this protocol specification.

The WHEP Endpoints MUST support OPTIONS requests for Cross-Origin Resource Sharing (CORS) as defined in [FETCH] and it SHOULD include an "Accept-Post" header with a mime type value of "application/sdp" on the "200 OK" response to any OPTIONS request recevied as per {{!W3C.REC-ldp-20150226}}.

The WHEP Resources MUST return an"405 Method Not Allowed" for any HTTP GET, HEAD, POST or PUT requests on the resource URL in order to reserve its usage for future versions of this protocol specification.

## ICE and NAT support

The SDP provided by the WHEP Player MAY be sent after the full ICE gathering is complete with the full list of ICE candidates, or it MAY only contain local candidates (or even an empty list of candidates) as per {{!RFC8863}}.

In order to simplify the protocol, there is no support for exchanging gathered trickle candidates from Media Server ICE candidates once the SDP answer is sent. The WHEP Endpoint SHALL gather all the ICE candidates for the Media Server before responding to the client request and the SDP answer SHALL contain the full list of ICE candidates of the Media Server. The Media Server MAY use ICE lite, while the WHEP player MUST implement full ICE.

Trickle ICE and ICE restart support is OPTIONAL for a WHEP resource.

If the WHEP resource supports either Trickle ICE or ICE restarts, the WHEP player MUST include an "Accept-Patch" header with a mime type value of "application/trickle-ice-sdpfrag" in the "201 Created" of the POST request that creates the WHEP resource as per {{!RFC5789}} section 3.1.

If the WHEP resource supports either Trickle ICE or ICE restarts, but not both, it MUST return a "405 Not Implemented" response for the HTTP PATCH requests that are not supported.

If the WHEP resource does not support the PATCH method for any purpose, it MUST return a "501 Not Implemented" response, as described in {{!RFC9110}} section 6.6.2.

As the HTTP PATCH request sent by a WHEP player may be received out-of-order by the WHEP Resource, the WHEP Resource MUST generate a
unique strong entity-tag identifying the ICE session as per {{!RFC9110}} section 2.3. The initial value of the entity-tag identifying the initial ICE session MUST be returned in an ETag header field in the "201 Created" response to the initial POST request to the WHEP Endpoint if the WHEP player is acting as SDP offerer, or in the HTTP PATCH response containing the SDP answer otherwise. It MUST also be returned in the "200 OK" of any PATCH request that triggers an ICE restart. Note that including the ETag in the original "201 Created" response is only REQUIRED if the WHEP resource supports ICE restarts and OPTIONAL otherwise.

A WHEP Player sending a PATCH request for performing trickle ICE MUST include an "If-Match" header field with the latest known entity-tag as per {{!RFC9110}} section 3.1. When the PATCH request is received by the WHEP resource, it MUST compare the indicated entity-tag value with the current entity-tag of the resource as per {{!RFC9110}} section 3.1 and return a "412 Precondition Failed" response if they do not match. 

WHEP Players SHOULD NOT use entity-tag validation when matching a specific ICE session is not required, such as when initiating a DELETE request to terminate a session.

A WHEP Resource receiving a PATCH request with new ICE candidates, but which does not perform an ICE restart, MUST return a "204 No Content" response without body. If the Media Server does not support a candidate transport or is not able to resolve the connection address, it MUST accept the HTTP request with the 204 response and silently discard the candidate.

~~~~~
PATCH /resource/id HTTP/1.1
Host: whep.example.com
If-Match: "38sdf4fdsf54:EsAw"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 548

a=ice-ufrag:EsAw
a=ice-pwd:P2uYro0UCOQ4zxjKXaWCBui1
m=audio RTP/AVP 0
a=mid:0
a=candidate:1387637174 1 udp 2122260223 192.0.2.1 61764 typ host generation 0 ufrag EsAw network-id 1
a=candidate:3471623853 1 udp 2122194687 198.51.100.1 61765 typ host generation 0 ufrag EsAw network-id 2
a=candidate:473322822 1 tcp 1518280447 192.0.2.1 9 typ host tcptype active generation 0 ufrag EsAw network-id 1
a=candidate:2154773085 1 tcp 1518214911 198.51.100.2 9 typ host tcptype active generation 0 ufrag EsAw network-id 2
a=end-of-candidates

HTTP/1.1 204 No Content
~~~~~
{: title="Trickle ICE request"}

A WHEP Player sending a PATCH request for performing ICE restart MUST contain an "If-Match" header field with a field-value "*" as per {{!RFC9110}} section 3.1. 

If the HTTP PATCH request results in an ICE restart, the WHEP resource SHALL return a "200 OK" with an "application/trickle-ice-sdpfrag" body containing the new ICE username fragment and password. The response may optionally contain the new set of ICE candidates for the Media Server and the new entity-tag correspond to the new ICE session in an ETag response header field.

If the ICE request cannot be satisfied by the WHEP Resource, the WHEP resource SHALL return a "200 OK" with an "application/trickle-ice-sdpfrag" body containing the new ICE username fragment and password. Also, the "200 OK" response for a successful ICE restart MUST contain the new entity-tag corresponding to the new ICE session in an ETag response header field and MAY contain a new set of ICE candidates for the Media Server.

~~~~~
PATCH /resource/id HTTP/1.1
Host: whep.example.com
If-Match: "*"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 54

a=ice-ufrag:ysXw
a=ice-pwd:vw5LmwG4y/e6dPP/zAP9Gp5k

HTTP/1.1 200 OK
ETag: "289b31b754eaa438:ysXw"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 102

a=ice-lite
a=ice-ufrag:289b31b754eaa438
a=ice-pwd:0b66f472495ef0ccac7bda653ab6be49ea13114472a5d10a
~~~~~
{: title="ICE restart request"}

Because the WHEP Player needs to know the entity-tag associated with the ICE session in order to send new ICE candidates, it MUST buffer any gathered candidates before it receives the HTTP response to the initial POST request or the PATCH request with the new entity-tag value. Once it knows the entity-tag value, the WHEP Player SHOULD send a single aggregated HTTP PATCH request with all the ICE candidates it has buffered so far.

In case of unstable network conditions, the ICE restart HTTP PATCH requests and responses might be received out of order. In order to mitigate this scenario, when the client performs an ICE restart, it MUST discard any previous ice username/pwd frags and ignore any further HTTP PATCH response received from a pending HTTP PATCH request. Clients MUST apply only the ICE information received in the response to the last sent request. If there is a mismatch between the ICE information at the client and at the server (because of an out-of-order request), the STUN requests will contain invalid ICE information and will be rejected by the server. When this situation is detected by the WHEP Player, it SHOULD send a new ICE restart request to the server.

## WebRTC constraints

In the specific case of media consumption from a streaming service, some assumptions can be made about the server-side which simplifies the WebRTC compliance burden, as detailed in WebRTC-gateway document {{?I-D.draft-ietf-rtcweb-gateways}}.

In order to reduce the complexity of implementing WHEP in both players and Media Servers, WHEP imposes the following restrictions regarding WebRTC usage:

Both the WHEP Player and the WHEP Endpoint SHALL use SDP bundle {{!RFC9143}}. Each "m=" section MUST be part of a single BUNDLE group. Hence, when a WHEP Player or a  WHEP Endpoints sends an SDP offer, it MUST include a "bundle-only" attribute in each bundled "m=" section. The WHEP player and the Media Server MUST support multiplexed media associated with the BUNDLE group as per {{!RFC9143}} section 9. In addition, per {{!RFC9143}} the WHEP Player and Media Server will use RTP/RTCP multiplexing for all bundled media. The WHEP Player and Media Server SHOULD include the "rtcp-mux-only" attribute in each bundled "m=" section as per {{!RFC8858}}.

As the codecs for a given stream may not be known by the Media Server when the WHEP Player starts watching a stream, if the WHEP Endpoint is acting as SDP answerer, it MUST include all the offered codecs that it supports in the SDP answer and not make any assumption about which will be the codec that will be actually sent.

Trickle ICE and ICE restarts support is OPTIONAL for both the WHEP Players and Media Servers as explained in section 4.1.

## Load balancing and redirections

WHEP Endpoints and Media Servers might not be co-located on the same server, so it is possible to load balance incoming requests to different Media Servers. WHEP Players SHALL support HTTP redirection via the "307 Temporary Redirect" response as described in {{!RFC9110}} section 6.4.7. The WHEP Resource URL MUST be a final one, and redirections are not required to be supported for the PATCH and DELETE requests sent to it.

In case of high load, the WHEP endpoints MAY return a "503 Service Unavailable" response indicating that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance, which will likely be alleviated after some delay. The WHEP Endpoint might send a Retry-After header field indicating the minimum time that the user agent ought to wait before making a follow-up request.

## STUN/TURN server configuration

The WHEP Endpoint MAY return STUN/TURN server configuration URLs and credentials usable by the client in the "201 Created" response to the HTTP POST request to the WHEP Endpoint URL.

Each STUN/TURN server will be returned using the "Link" header field {{!RFC8288}} with a "rel" attribute value of "ice-server" as specified in {{I-D.draft-ietf-wish-whip}}

It might be also possible to configure the STUN/TURN server URLs with long-term credentials provided by either the broadcasting service or an external TURN provider on the WHEP Player, overriding the values provided by the WHEP Endpoint.

## Authentication and authorization

WHEP Endpoints and Resources MAY require the HTTP request to be authenticated using an HTTP Authorization header field with a Bearer token as specified in {{!RFC6750}} section 2.1. WHEP players MUST implement this authentication and authorization mechanism and send the HTTP Authorization header field in all HTTP requests sent to either the WHEP endpoint or resource except the preflight OPTIONS requests for CORS.

The nature, syntax, and semantics of the bearer token, as well as how to distribute it to the client, is outside the scope of this document. Some examples of the kind of tokens that could be used are, but are not limited to, JWT tokens as per {{!RFC6750}} and {{!RFC8725}} or a shared secret stored on a database. 

WHEP Endpoints and Resources could perform the authentication and authorization by encoding an authentication token within the URLs for the WHEP Endpoints or Resources instead. In case the WHEP Player is not configured to use a bearer token, the HTTP Authorization header field must not be sent in any request.

## Protocol extensions

In order to support future extensions to be defined for the WHEP protocol, a common procedure for registering and announcing the new extensions is defined.

Protocol extensions supported by the WHEP server MUST be advertised to the WHEP Player in the "201 Created" response to the initial HTTP POST request sent to the WHEP Endpoint. The WHEP Endpoint MUST return one "Link" header field for each extension, with the extension "rel" type attribute and the URI for the HTTP resource that will be available for receiving requests related to that extension.

Protocol extensions are optional for both WHEP Players and WHEP Endpoints and Resources. WHEP Players MUST ignore any Link attribute with an unknown "rel" attribute value and WHEP Endpoints and Resources MUST NOT require the usage of any of the extensions.

Each protocol extension MUST register a unique "rel" attribute value at IANA starting with the prefix: "urn:ietf:params:whep:ext" as specified in {{urn-whep-subspace}}.

For example, considering a potential extension of server-to-client communication using server-sent events as specified in https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events, the URL for connecting to the server side event resource for the published stream could be returned in the initial HTTP "201 Created" response with a "Link" header field and a "rel" attribute of "urn:ietf:params:whep:ext:example:server-sent-events". (This document does not specify such an extension, and uses it only as an example.)

In this theoretical case, the "201 Created" response to the HTTP POST request would look like:

~~~~~
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: https://whep.example.org/resource/id
Link: <https://whep.ietf.org/publications/213786HF/sse>;
      rel="urn:ietf:params:whep:ext:example:server-side-events"
~~~~~


# Security Considerations

HTTPS SHALL be used in order to preserve the WebRTC security model.


# IANA Considerations

This specification adds a registry for URN sub-namespaces for WHEP protocol extensions.

## Registration of WHEP URN Sub-namespace and whep Registry

IANA has added an entry to the "IETF URN Sub-namespace for Registered Protocol Parameter Identifiers" registry and created a sub-namespace for the Registered Parameter Identifier as per {{!RFC3553}}: "urn:ietf:params:whep".

To manage this sub-namespace, IANA has created the "System for Cross-domain Identity Management (WHEP) Schema URIs" registry, which is used to manage entries within the "urn:ietf:params:whep" namespace.  The registry description is as follows:

   - Registry name: WHEP

   - Specification: this document (RFC TBD)

   - Repository: See Section {{urn-whep-subspace}}

   - Index value: See Section {{urn-whep-subspace}}

## URN Sub-namespace for whep {#urn-whep-subspace}

whep Endpoint utilize URIs to identify the supported whep protocol extensions on the "rel" attribute of the Link header as defined in {{protocol-extensions}}.
This section creates and registers an IETF URN Sub-namespace for use in the whep specifications and future extensions.

### Specification Template

Namespace ID:

      The Namespace ID "whep" has been assigned.

Registration Information:

      Version: 1

      Date: TBD

Declared registrant of the namespace:

      The Internet Engineering Task Force.

Designated contact:

       A designated expert will monitor the whep public mailing list, "wish@ietf.org".

Declaration of Syntactic Structure:

      The Namespace Specific String (NSS) of all URNs that use the "whep" Namespace ID shall have the following structure: urn:ietf:params:whep:{type}:{name}:{other}

      The keywords have the following meaning:

      - type: The entity type. This specification only defines the "ext" type.

      - name: A required US-ASCII string that conforms to the URN syntax requirements (see {{?RFC8141}}) and defines a major namespace of a whep protocol extension. The value MAY also be an industry name or organization name.

      - other: Any US-ASCII string that conforms to the URN syntax requirements (see {{?RFC8141}}) and defines the sub-namespace (which MAY be further broken down in namespaces delimited by colons) as needed to uniquely identify an whep protocol extension.

Relevant Ancillary Documentation:

      None

Identifier Uniqueness Considerations:

      The designated contact shall be responsible for reviewing and enforcing uniqueness.

Identifier Persistence Considerations:

      Once a name has been allocated, it MUST NOT be reallocated for a different purpose.
      The rules provided for assignments of values within a sub-namespace MUST be constructed so that the meanings of values cannot change.
      This registration mechanism is not appropriate for naming values whose meanings may change over time.

Process of Identifier Assignment:

      Namespace with type "ext" (e.g., "urn:ietf:params:whep:ext") is reserved for IETF-approved whep specifications.

Process of Identifier Resolution:

      None specified.

Rules for Lexical Equivalence:

      No special considerations; the rules for lexical equivalence specified in {{?RFC8141}} apply.

Conformance with URN Syntax:

      No special considerations.

Validation Mechanism:

      None specified.

Scope:

      Global.

# Acknowledgements

--- back


