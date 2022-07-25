---
docname: draft-murillo-whep-00
title: WebRTC-HTTP egress protocol (WHEP)
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

--- abstract

This document describes a simple HTTP-based protocol that will allow WebRTC-based viewers to watch content from streaming services and/or CDNs(Content Delivery Network)/WTNs(WebRTC Transmission Network).

--- middle

# Introduction


# Terminology

{::boilerplate bcp14-tagged}

- WHEP player: WebRTC media player that acts as a client of the WHEP protocol by receiving and decoding the media from a remote Media Server.
- WHEP endpoint: Egress server receiving the initial WHEP request.
- WHEP endpoint URL: URL of the WHEP endpoint that will create the WHEP resource.
- Media Server: WebRTC Media Server or consumer that establishes the media session with the WHEP player and delivers the media to it.
- WHEP resource: Allocated resource by the WHEP endpoint for an ongoing egress session that the WHEP player can send requests for altering the session (ICE operations or termination, for example).
- WHEP resource URL: URL allocated to a specific media session by the WHEP endpoint which can be used to perform operations such as terminating the session or ICE restarts.


# Overview

The WebRTC-HTTP Egress Protocol (WHEP) uses an HTTP POST request to perform a single-shot SDP offer/answer so an ICE/DTLS session can be established between the WHEP player and the streaming service endpoint (Media Server).

Once the ICE/DTLS session is set up, the media will flow unidirectionally from Media Server to the WHEP player. In order to reduce complexity, no SDP renegotiation is supported, so no tracks or streams can be added or removed once the initial SDP offer/answer over HTTP is completed.

~~~~~
                                                                               
 +-------------+    +---------------+ +--------------+ +---------------+
 | WHEP player |    | WHEP endpoint | | Media Server | | WHEP Resource |
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


Alternatively, there are cases in which the WHEP player may wish the service to provide the SDP offer (for example to avoid setting up an audio and video session when only audio is supported), so in this case the initial HTTP POST request will not contain a body and the response will contain the SDP offer from the service instead. The WHEP player will have to provide the SDP answer in a subsequent HTTP PATCH request to the WHEP resource.
~~~~~
                                                                               
 +-------------+    +---------------+ +--------------+ +---------------+
 | WHEP player |    | WHEP endpoint | | Media Server | | WHEP Resource |
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
In order to set up a streaming session, the WHEP player will generate an SDP offer according to the JSEP rules and perform an HTTP POST request to the configured WHEP endpoint URL.

The HTTP POST request will have a content type of "application/sdp" and contain the SDP offer as the body. The WHEP endpoint will generate an SDP answer and return a "201 Created" response with a content type of "application/sdp", the SDP answer as the body, and a Location header field pointing to the newly created resource.

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
ETag: "38sdf4fdsf54:EsAw"
Content-Type: application/sdp
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

If the WHEP player prefers the WHEP Endpoint to generate the SDP offer, the WHEP player will send a POST request without HTTP BODY and an Accept HTTP header of "application/sdp" to the configured WHEP endpoint URL. The HTTP POST request MUST have a content type of "application/sdp". 

The WHEP endpoint will generate an SDP offer according to the JSEP rules and return a "201 Created" response with a content type of "application/sdp", the SDP offer as the body, a Location header field pointing to the newly created resource and an Expire header indicating the maximum time that the WHEP player is allowed to send the SDP answer to the WHEP resource.

The WHEP player MUST generate an SDP answer to SDP offer provided by the WHEP endpoint and send an HTTP PATCH request to the URL provided in the Location header for the WHEP resource. The HTTP PATCH request will have a content type of "application/sdp" and contain the SDP answer as the body. If the SDP offer is not accepted by the WHEP player, it MUST perform an HTTP DELETE operation for terminating the session to the WHEP resource URL.

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
ETag: "38sdf4fdsf54:EsAw"
~~~~~
{: title="HTTP POST and PATCH doing SDP O/A example"}

If the WHEP resource does not receive an HTTP PATCH request before the time indicated in the Expire header HTTP POST response, it SHOULD delete the resource and respond with a 404 Not Found response to any request on the WHEP resource URL received afterwards.



## Common procedures

Once a session is set up, ICE consent freshness {{!RFC7675}} will be used to detect abrupt disconnection and DTLS teardown for session termination by either side.

To explicitly terminate a session, the WHEP player MUST perform an HTTP DELETE request to the resource URL returned in the Location header field of the initial HTTP POST. Upon receiving the HTTP DELETE request, the WHEP resource will be removed and the resources freed on the Media Server, terminating the ICE and DTLS sessions.

A Media Server terminating a session MUST follow the procedures in {{!RFC7675}} section 5.2 for immediate revocation of consent.

The WHEP endpoints MUST return an HTTP 405 response for any HTTP GET, HEAD or PUT requests on the resource URL in order to reserve its usage for future versions of this protocol specification.

The WHEP resources MUST return an HTTP 405 response for any HTTP GET, HEAD, POST or PUT requests on the resource URL in order to reserve its usage for future versions of this protocol specification.

## ICE and NAT support

The SDP provided by the WHEP player MAY be sent after the full ICE gathering is complete with the full list of ICE candidates, or it MAY only contain local candidates (or even an empty list of candidates).

In order to simplify the protocol, there is no support for exchanging gathered trickle candidates from Media Server ICE candidates once the SDP answer is sent. The  WHEP Endpoint SHALL gather all the ICE candidates for the Media Server before responding to the client request and the SDP answer SHALL contain the full list of ICE candidates of the Media Server. The Media Server MAY use ICE lite, while the WHEP player MUST implement full ICE.

The WHEP player MAY perform trickle ICE or ICE restarts {{!RFC8863}} by sending an HTTP PATCH request to the WHEP resource URL with a body containing a SDP fragment with MIME type "application/trickle-ice-sdpfrag" as specified in {{!RFC8840}}. When used for trickle ICE, the body of this PATCH message will contain the new ICE candidate; when used for ICE restarts, it will contain a new ICE ufrag/pwd pair.

The WHEP player MUST NOT send any ICE trickle or restart until the SDP O/A is completed. So, if the WHEP player is not acting as offerer in the SDP O/A, it MUST NOT send any HTTP PATCH request for ICE trickle or restart until the 200 OK response to the HTTP PATCH request containing the SDP answer has been received.

Trickle ICE and ICE restart support is OPTIONAL for a WHEP resource. If both Trickle ICE or ICE restarts are not supported by the WHEP resource, it MUST return a 405 Method Not Allowed response for any HTTP PATCH request. If the WHEP resource supports either Trickle ICE or ICE restarts, but not both, it MUST return a 501 Not Implemented for the HTTP PATCH requests that are not supported.

As the HTTP PATCH request sent by a WHEP player may be received out-of-order by the WHEP resource, the WHEP resource MUST generate a
unique strong entity-tag identifying the ICE session as per {{!RFC7232}} section 2.3. The initial value of the entity-tag identifying the initial ICE session MUST be returned in an ETag header field in the 201 response to the initial POST request to the WHEP endpoint if the WHEP player is acting as SDP offerer, or in the HTTP PATCH response containing the SDP answer otherwise. It MUST also be returned in the 200 OK of any PATCH request that triggers an ICE restart.

A WHEP player sending a PATCH request for performing trickle ICE MUST include an "If-Match" header field with the latest known entity-tag as per {{!RFC7232}} section 3.1. When the PATCH request is received by the WHEP resource, it MUST compare the indicated entity-tag value with the current entity-tag of the resource as per {{!RFC7232}} section 3.1 and return a "412 Precondition Failed" response if they do not match. 

WHEP players SHOULD NOT use entity-tag validation when matching a specific ICE session is not required, such as when initiating a DELETE request to terminate a session.

A WHEP resource receiving a PATCH request with new ICE candidates, but which does not perform an ICE restart, MUST return a "204 No Content" response without body. If the Media Server does not support a candidate transport or is not able to resolve the connection address, it MUST accept the HTTP request with the 204 response and silently discard the candidate.

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

A WHEP player sending a PATCH request for performing ICE restart MUST contain an "If-Match" header field with a field-value "*" as per {{!RFC7232}} section 3.1. 

If the HTTP PATCH request results in an ICE restart, the WHEP resource SHALL return a "200 OK" with an "application/trickle-ice-sdpfrag" body containing the new ICE username fragment and password. The response may optionally contain the new set of ICE candidates for the Media Server and the new entity-tag correspond to the new ICE session in an ETag response header field.

If the ICE request cannot be satisfied by the WHEP resource, the resource MUST return an appropriate HTTP error code and MUST NOT terminate the session immediately. The WHEP player MAY retry performing a new ICE restart or terminate the session by issuing an HTTP DELETE request instead. In either case, the session MUST be terminated if the ICE consent expires as a consequence of the failed ICE restart as per {{!RFC7675}} section 5.1. 

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

Because the WHEP player needs to know the entity-tag associated with the ICE session in order to send new ICE candidates, it MUST buffer any gathered candidates before it receives the HTTP response to the initial POST request or the PATCH request with the new entity-tag value. Once it knows the entity-tag value, the WHEP player SHOULD send a single aggregated HTTP PATCH request with all the ICE candidates it has buffered so far.

In case of unstable network conditions, the ICE restart HTTP PATCH requests and responses might be received out of order. In order to mitigate this scenario, when the client performs an ICE restart, it MUST discard any previous ice username/pwd frags and ignore any further HTTP PATCH response received from a pending HTTP PATCH request. Clients MUST apply only the ICE information received in the response to the last sent request. If there is a mismatch between the ICE information at the client and at the server (because of an out-of-order request), the STUN requests will contain invalid ICE information and will be rejected by the server. When this situation is detected by the WHEP player, it SHOULD send a new ICE restart request to the server.

## WebRTC constraints

In the specific case of media consumption from a streaming service, some assumptions can be made about the server-side which simplifies the WebRTC compliance burden, as detailed in WebRTC-gateway document {{?draft-ietf-rtcweb-gateways}}.

In order to reduce the complexity of implementing WHEP in both clients and Media Servers, WHEP imposes the following restrictions regarding WebRTC usage:

Both the WHEP player and the WHEP endpoint SHALL use SDP bundle {{!RFC8843}}. Each "m=" section MUST be part of a single BUNDLE group. Hence, when a WHEP player sends an SDP offer, it MUST include a "bundle-only" attribute in each bundled "m=" section. The WHEP player and the Media Server MUST support multiplexed media associated with the BUNDLE group as per {{!RFC8843}} section 9. In addition, per {{!RFC8843}} the WHEP player and Media Server will use RTP/RTCP multiplexing for all bundled media. The WHEP player and Media Server SHOULD include the "rtcp-mux-only" attribute in each bundled "m=" section.

As the codecs for a given stream may not be known by the Media Server when the WHEP player starts watching a stream, if the WHEP endpoint is acting as SDP answerer, it MUST include all the offered codecs that it supports in the SDP answer and not make any assumption about which will be the codec that will be actually sent.

Trickle ICE and ICE restarts support is OPTIONAL for both the WHEP players and Media Servers as explained in section 4.1.

## Load balancing and redirections

WHEP endpoints and Media Servers might not be co-located on the same server, so it is possible to load balance incoming requests to different Media Servers. WHEP players SHALL support HTTP redirection via the "307 Temporary Redirect response code" as described in {{!RFC7231}} section 6.4.7. The WHEP resource URL MUST be a final one, and redirections are not required to be supported for the PATCH and DELETE requests sent to it.

In case of high load, the WHEP endpoints MAY return a 503 (Service Unavailable) status code indicating that the server is currently unable to handle the request due to a temporary overload or scheduled maintenance, which will likely be alleviated after some delay. The WHEP endpoint might send a Retry-After header field indicating the minimum time that the user agent ought to wait before making a follow-up request.

## STUN/TURN server configuration

The WHEP endpoint MAY return STUN/TURN server configuration URLs and credentials usable by the client in the "201 Created" response to the HTTP POST request to the WHEP endpoint URL.

Each STUN/TURN server will be returned using the "Link" header field {{!RFC8288}} with a "rel"" attribute value of "ice-server". The Link target URI is the server URL as defined in {{!RFC7064}} and {{!RFC7065}}. The credentials are encoded in the Link target attributes as follows:

- username: If the Link header field represents a TURN server, and credential-type is "password", then this attribute specifies the username to use with that TURN server.
- credential: If the "credential-type" attribute is missing or has a "password" value, the credential attribute represents a long-term authentication password, as described in {{!RFC8489}}, Section 10.2.
- credential-type: If the Link header field represents a TURN server, then this attribute specifies how the credential attribute value should be used when that TURN server requests authorization. The default value if the attribute is not present is "password".

~~~~~
     Link: <stun:stun.example.net>; rel="ice-server"
     Link: <turn:turn.example.net?transport=udp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
     Link: <turn:turn.example.net?transport=tcp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
     Link: <turns:turn.example.net?transport=tcp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
~~~~~
{: title="Example ICE server configuration"}

NOTE: The naming of both the "rel" attribute value of "ice-server" and the target attributes follows the one used on the W3C WebRTC recommendation {{?W3C.REC-webrtc-20210126} RTCConfiguration dictionary in section 4.2.1. "rel" attribute value of "ice-server" is not prepended with the "urn:ietf:params:whep:" so it can be reused by other specifications which may use this mechanism to configure the usage of STUN/TURN servers.

There are some WebRTC implementations that do not support updating the STUN/TURN server configuration after the local offer has been created as specified in {{!RFC8829}} section 4.1.18. In order to support these clients, the WHEP endpoint MAY also include the STUN/TURN server configuration on the responses to OPTIONS request sent to the WHEP endpoint URL before the POST request is sent.

The generation of the TURN server credentials may require performing a request to an external provider, which can both add latency to the OPTION request processing and increase the processing required to handle that request. In order to prevent this, the WHEP Endpoint SHOULD NOT return the STUN/TURN server configuration if the OPTION request is a preflight request for CORS, that is, if the OPTION request does not contain an Access-Control-Request-Method with "POST" value and the Access-Control-Request-Headers HTTP header does not contain the "Link" value. 

It might be also possible to configure the STUN/TURN server URLs with long-term credentials provided by either the broadcasting service or an external TURN provider on the WHEP player, overriding the values provided by the WHEP endpoint.

## Authentication and authorization

WHEP endpoints and resources MAY require the HTTP request to be authenticated using an HTTP Authorization header field with a Bearer token as specified in {{!RFC6750}} section 2.1. WHEP players MUST implement this authentication and authorization mechanism and send the HTTP Authorization header field in all HTTP requests sent to either the WHEP endpoint or resource except the preflight OPTIONS requests for CORS.

The nature, syntax, and semantics of the bearer token, as well as how to distribute it to the client, is outside the scope of this document. Some examples of the kind of tokens that could be used are, but are not limited to, JWT tokens as per {{!RFC6750}} and {{!RFC8725}} or a shared secret stored on a database. The tokens are typically made available to the end user alongside the WHEP endpoint URL and configured on the WHEP players (similar to the way RTMP URLs and Stream Keys are distributed).

WHEP endpoints and resources could perform the authentication and authorization by encoding an authentication token within the URLs for the WHEP endpoints or resources instead. In case the WHEP player is not configured to use a bearer token, the HTTP Authorization header field must not be sent in any request.

## Protocol extensions

In order to support future extensions to be defined for the WHEP protocol, a common procedure for registering and announcing the new extensions is defined.

Protocol extensions supported by the WHEP server MUST be advertised to the WHEP player in the "201 Created" response to the initial HTTP POST request sent to the WHEP endpoint. The WHEP endpoint MUST return one "Link" header field for each extension, with the extension "rel" type attribute and the URI for the HTTP resource that will be available for receiving requests related to that extension.

Protocol extensions are optional for both WHEP players and servers. WHEP players MUST ignore any Link attribute with an unknown "rel" attribute value and WHEP servers MUST NOT require the usage of any of the extensions.

Each protocol extension MUST register a unique "rel" attribute value at IANA starting with the prefix: "urn:ietf:params:whep:".

For example, considering a potential extension of server-to-client communication using server-sent events as specified in https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events, the URL for connecting to the server side event resource for the published stream could be returned in the initial HTTP "201 Created" response with a "Link" header field and a "rel" attribute of "urn:ietf:params:whep:server-sent-events". (This document does not specify such an extension, and uses it only as an example.)

In this theoretical case, the HTTP 201 response to the HTTP POST request would look like:

~~~~~
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: https://whep.example.org/resource/id
Link: <https://whep.ietf.org/publications/213786HF/sse>;rel="urn:ietf:params:whep:server-side-events"
~~~~~


# Security Considerations

HTTPS SHALL be used in order to preserve the WebRTC security model.


# IANA Considerations

The link relation type below has been registered by IANA per Section 4.2 of {{!RFC8288}}.

## Link Relation Type: ice-server

Relation Name:  ice-server

Description:  For the WHEP protocol, conveys the STUN and TURN servers that can be used by an ICE Agent to establish a connection with a peer.

Reference:  TBD

# Acknowledgements

--- back


