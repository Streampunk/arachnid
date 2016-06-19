# Arachnid

A specification of HTTP(s)-based transport of NMOS grains and flows over the world-wide-web. This specification is a prototype and is not yet complete. It is being developed in parallel with a Node.js implementation as `spm-http-in` and `spm-http-out` nodes in [dynamorse](/Streampunk/dynamorse). 

The aim of the specification is to efficiently transport NMOS grains, which are PTP timestamped video frames, chunks of audio samples or data/event media data, using parallel, overlapped HTTP requests. The specification supports push and pull modes, with a choice of unsecure HTTP or secure HTTPS protocols. The headers are common to both all methods. Transport takes place at the speed of the network, or faster if 

## Headers

### Grain-specific headers

Mapping of the grain logical model to HTTP headers.

* `X-Arachnid-OriginTimeStamp` - PTP timestamp in '<secs>:<nanos>' notation.
* `X-Arachnid-SyncTimeStamp` - PTP timestamp in '<secs>:<nanos>' notation.
* `X-Arachnid-Timecode` - SMPTE 12M Timecode formatted in HH:MM:SS[:;]FF notation.
* `X-Arachnid-FlowID` - UUID formatted in xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx notation
* `X-Arachnid-SourceID` - UUID formatted in xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx notation
* `X-Arachnid-GrainType` - Enumeration of `video`, `audio`, `data`
* `X-Arachnid-GrainDuration` - Rational number in `<numerator>/<denominator>` format - fraction of a second.

`Content-Type` should be set to the same MIME type as defined by a mapping of the same prototcol to an SDP file, e.g. `video/raw` or `audio/L16`. Additional protocol parameters should be specified, such as width and height, for example:

`Content-Type: video/raw; sampling=YCbCr-4:2:2; width=1920; height=1080; depth=10; colorimetry=BT709-2; interlace=1 sampling=YCbCr-4:2:2; width=1920; height=1080; depth=10; colorimetry=BT709-2; interlace=1`

### Arachnid protocol headers

Headers that deal with parallel threads. 

* `X-Arachnid-NextByThread`

`Content-Length` must be provided.

Details to follow.

## Next steps and future

Arachnid is labelled as _Streampunk Media arachnid_ to make it clear that this is not an AMWA or NMOS RFC or specification at this time. This is intended as an open specification and will be offered to the AMWA as an RFC as soon as it is sufficiently mature.
