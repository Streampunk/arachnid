# Arachnid 
## DRAFT!

Streampunk Media's draft specification of HTTP(s)-based transport of NMOS grains and flows over the world-wide-web. This specification is a prototype and is not yet complete. It is being developed in parallel with a Node.js implementation as `spm-http-in` and `spm-http-out` nodes in [dynamorse](/Streampunk/dynamorse). 

The aim of the specification is to efficiently transport NMOS grains, which are PTP timestamped video frames, chunks of audio samples or data/event media data, using parallel, overlapped HTTP requests. The specification supports push and pull modes, with a choice of unsecure HTTP or secure HTTPS protocols. The headers are common to both all methods. Transport takes place as fast as the network can carry the underlying packets and may be faster than real time. Senders may slow down the speed of their response to real time or slower. In this way, back pressure can be applied between processing endpoints.

As a consequene of this approach, grains of a media stream do not necessarily travel in the order that they are presented and the packets on the wire will be an interleaved mix of different grains from the same flow. Clients may need to have a small buffer to re-order a few frames. This approach relies on a strong identity and timing model, as described in the JT-NM reference architecture. The primary benefit of this approach is that streams will scale to fill avaiable network pipes with unmodified operating system kernels. This allows a user a runtime choice to trade a few grains of latency for more reliability, better bandwidth utilisation, back pressure, security and defacto Internet-backed routing.

## Headers

### Grain-specific headers

Mapping of the grain logical model to HTTP headers.

* `X-Arachnid-OriginTimeStamp` - PTP timestamp in `<secs>:<nanos>` notation.
* `X-Arachnid-SyncTimeStamp` - PTP timestamp in `<secs>:<nanos>` notation.
* `X-Arachnid-Timecode` - SMPTE 12M Timecode formatted in `HH:MM:SS[:;]FF` notation.
* `X-Arachnid-FlowID` - UUID formatted in `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` notation
* `X-Arachnid-SourceID` - UUID formatted in `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` notation
* `X-Arachnid-GrainType` - Enumeration of `video`, `audio`, `data`
* `X-Arachnid-GrainDuration` - Rational number in `<numerator>/<denominator>` format - fraction of a second.
* `X-Arachnid-FourCC` - (optional) FourCC code representing the packing of uncompressed samples into the stream.

`Content-Type` shall be set to the registered MIME type, e.g. `video/raw` or `audio/L16`. Additional parameters shall be specified according to the defining RFC, for example:

`Content-Type: video/raw; sampling=YCbCr-4:2:2; width=1920; height=1080; depth=10; colorimetry=BT709-2; interlace=1 sampling=YCbCr-4:2:2; width=1920; height=1080; depth=10; colorimetry=BT709-2; interlace=1`

`Content-Type: audio/L16; rate=48000; channels=2`

`Content-Length` must be provided so that a receiver can pre-allocate suitable buffers. However, it is expected that larger grains will be delivered as defined lumps.

### Arachnid protocol headers

Headers that deal with parallel threads, present in requests and responses unless otherwise specified.

* `X-Arachnid-NextByThread` - Response only. Given the number of threads being used to transport the media, the timestamp of the next grain that this thread should request. For example, with 5 threads, the value of `X-Arachnid-NextByThread` will be `X-Arachnid-OriginTimeStamp` plus 5 times `X-Arachnid-GrainDuration`.
* `X-Arachnid-ThreadNumber` - Thread number for this request - zero-based.
* `X-Arachnid-TotalConcurrent` - Number of threads making requests for this content.
* `X-Arachnid-ClientID` - An identifier for the client making requests, used so that a server can provided consistent timestamps across multiple threads at the start of a session.

In the absence of these headers, transport is assumed to be single-threaded or the concurrency does not require the assistance of the server.

PTP timestamps shall always contain nine digits to specify the nanosecond value and shall be zero-padded if the value could be written with less digits.

#### Maximum number of parallel streams 

In common with the established limitation for web browsers, no more than 6 parallel requests can be made for content for the same flow. Many firewalls and servers are configured to treat more than six concurrent requests for content between client and server as a denial of service attack.

## URLs

URLs take the form of a base path and either the grain's origin time stamp or relative time to now. The URL should contain the flow identifier or label, but this is not mandatory. The base path must be specified as the `manifest_href` path of the associated NMOS registration and dicovery API sender object. For example, an explicit grain reference:

`http://clips.newsroom.glasgow.spm.co.uk/flows/4223aa8d-9e3f-4a08-b0ba-863f26268b6f/1466371328:891000000`

Relative references take the form of a positive or negative integer specifying how the grain to be returned relates to the current time at the server and is measured in grain counts. For example, `0` is the most recently emitted grain of a flow from a sender, `1` is the next grain to be emitted and `-1` is the previous grain. For example:

`http://clips.newsroom.glasgow.spm.co.uk/flows/4223aa8d-9e3f-4a08-b0ba-863f26268b6f/-1`

## Pull

In pull mode, a receiver is a client that makes HTTP GET requests of a sender that an HTTP server. 

Senders may cache every grain of a flow or may have a limited cache of, say, 10-30 grains. This is entirely configurable by use case. If a receiver happens to know the grain timestamps of a flow, it could start to make explicit requests for grains. It may get a cache hit or miss, depending on the size of the cache. The assumption is that most of the time, a receiver wants to get the grains that most recently flowed, to this protocol supports a startup phase followed by requests for explicit grains.

The receiver starts by making a number of concurrent relative get requests of the sender as it does not yet know the timestamps in the stream, for example with 4 threads the receiver requests grains `.../-3`, `.../-2`, `.../-1` and `.../0`. The receiver sets the `X-Arachnid-ThreadNumber`, `X-Arachnid-TotalConcurrent` and `X-Arachnid-ClientID` headers to inform the server that it would like a consistent set of timestamps across the responses. The server responds with grains relative to the last grain that was emitted with relative path `0` and explicit path of _t_. The value of _t_ is returned in the response to the request for path `.../0` in header `X-Arachnid-OriginTimeStamp`. For request `.../-1` this header is _t_ - _d_ where _d_ is grain duration which is `40:080000000`, for '.../-2' it is _t_ - 2 * _d_ which is `40:040000000` etc.. 

The responses also contain `X-Arachnid-NextByThread` headers indicating this next grain timestamp that should be requested on that thread. If the number of threads is _c_, request `.../0` has `X-Arachnid-NextByThread` set to _t_ + _c_ * _d_, which is `40:280000000`. This is the server informing the client that the next grain it should request after fully completing its request has the given explicit timestamp, that the receiver should switch to making explicit requests. In this way, the grain rate of a stream may change mid flow. A server must reply with consistent information across all threads with the same client identifier for a period of 10 seconds from receiving the first request. 

To complete the example, here are the example request and response headers (with the first fairly complete):

```
GET http://clips.newsroom.glasgow.spm.co.uk/flows/4223aa8d-9e3f-4a08-b0ba-863f26268b6f/-3 HTTP/1.1
...
X-Arachnid-ThreadNumber: 0
X-Arachnid-TotalConcurrent: 4
X-Arachnid-ClientID: transform7_inverness
...

HTTP/1.1 200 OK
...
X-Arachnid-ThreadNumber: 0
X-Arachnid-TotalConcurrent: 4
X-Arachnid-OriginTimeStamp: 40:000000000
X-Arachnid-SyncTimeStamp: 40:000000000
X-Arachnid-Timecode: 10:00:00:00
X-Arachnid-FlowID: 4223aa8d-9e3f-4a08-b0ba-863f26268b6f
X-Arachnod-SourceID: 26bb72a1-0112-495d-81ab-f5160ca69015
X-Arachnid-NextByThread: 40:160000000
X-Arachnid-GrainType: video
X-Arachnid-GrainDuration: 1/25
Content-Type: video/raw; sampling=YCbCr-4:2:2; width=1920; height=1080; depth=10; colorimetry=BT709-2; interlace=1 sampling=YCbCr-4:2:2; width=1920; height=1080; depth=10; colorimetry=BT709-2; interlace=1
Content-Length: 5184000
...
```

```
GET http://clips.newsroom.glasgow.spm.co.uk/flows/4223aa8d-9e3f-4a08-b0ba-863f26268b6f/-2 HTTP/1/1
...
X-Arachnid-ThreadNumber: 1
X-Arachnid-TotalConcurrent: 4
X-Arachnid-ClientID: transform7_inverness
...

HTTP/1.1 200 OK
...
X-Arachnid-ThreadNumber: 1
X-Arachnid-TotalConcurrent: 4
X-Arachnid-OriginTimeStamp: 40:040000000
X-Arachnid-NextByThread: 40:200000000
X-Arachnid-GrainDuration: 1/25
...
```

```
GET http://clips.newsroom.glasgow.spm.co.uk/flows/4223aa8d-9e3f-4a08-b0ba-863f26268b6f/-1 HTTP/1/1
...
X-Arachnid-ThreadNumber: 2
X-Arachnid-TotalConcurrent: 4
X-Arachnid-ClientID: transform7_inverness
...

HTTP/1.1 200 OK
...
X-Arachnid-ThreadNumber: 2
X-Arachnid-TotalConcurrent: 4
X-Arachnid-OriginTimeStamp: 40:080000000
X-Arachnid-NextByThread: 40:240000000
X-Arachnid-GrainDuration: 1/25
...
```

```
GET http://clips.newsroom.glasgow.spm.co.uk/flows/4223aa8d-9e3f-4a08-b0ba-863f26268b6f/0 HTTP/1/1
...
X-Arachnid-ThreadNumber: 3
X-Arachnid-TotalConcurrent: 4
X-Arachnid-ClientID: transform7_inverness
...

HTTP/1.1 200 OK
...
X-Arachnid-ThreadNumber: 3
X-Arachnid-TotalConcurrent: 4
X-Arachnid-OriginTimeStamp: 40:120000000
X-Arachnid-NextByThread: 40:2800000000
X-Arachnid-GrainDuration: 1/25
...
```

Subsequent requests by thread are made to `.../40:160000000`, `.../40:200000000`, `.../40:240000000` and `.../40:280000000`, and not to `.../1`, `.../2`, `.../3` etc.. 

Servers should be capable of answering requests for relative grains as follows:

* Servers should answer requests for relative frames within a range of plus or minus ten grains, or the HTTP timeout for the platform, whichever is the shorter time. Note that long-GOP grains may have a grain duration of some seconds.
* Requests for grains that were cached but that are no longer available (larger negative minus number) should produce a `410 Gone` HTTP response - the resource has gone from this base path and will not be available again.
* Requests for grains that are too far in the future should be answered with a `404 Not Found` response code as the grains may become available if requested in the future.

## Push 

In push mode, a sender is a client that makes HTTP POST requests to a receiver that is an HTTP server.

_Details to follow_

## Protocol - HTTP and HTTPS

Use HTTPS instead of HTTP. A server will require certificates that must be set up in the standard way for TLS/SSL security. Content will be encrypted between endpoints.

_Details to follow_

## Comparison to other protocols

Why not use MPEG-DASH, HLS or similar? The design goal of this protocol is to avoid the use of pre-allocation and manifests and to encourage the use of parallel transport as a good alternative to RTP in a data centre. In particular, the application of this protocol is intended for optimal high-bitrate, low latency transport of uncompressed and lightly compressed streams at one quaility level whereas MPEG-DASH and HLS support lower-latency, lower-bandwidth transport of adaptive bitrate streams - arguably a different use case.

However, this is a journey and it is entirely possible that this protocol will morph into something like MPEG-DASH. Watch this space!

## Next steps and future

Arachnid is labelled as _Streampunk Media arachnid_ to make it clear that this is not an AMWA or NMOS RFC or specification at this time. This is intended as an open specification and will be offered to the AMWA as an RFC as soon as it is sufficiently mature.

Implementation is ongoing. Further details to follow. For more information or if you have questions, contact the author Richard Cartwright (spark@streampunk.media).

## License

This specification is released under the Apache 2.0 license. Copyright 2016 Streampunk Media Ltd.
