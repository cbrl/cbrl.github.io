---
title: Extract and Preserve Metadata in Videos From Sony Alpha Cameras
date: 2025-05-03 15:46:28 +/-0500
categories: [Photography, Metadata]
tags: [sony, sony alpha, camera, video, metadata]
---

## Overview

Sony Alpha[^1] cameras embed detailed metadata in their video files. However, much of it is stored
using non-standard methods that most tools and services fail to recognize. Outside the standard
QuickTime tags, which contain minimal information, there are two primary sources of metadata:

[^1]: This information may also apply to some models outside the Alpha series.

1. **Embedded XML Document**: This resides in the root MP4 container and contains static metadata
about the video and the camera. This information is also typically saved as a sidecar XML file
alongside the video.

2. **Timestamped Data Stream**: This is stored in a separate data track (identified with the
handler type `rtmd` for "Realtime Metadata") within the video file. It contains a stream of
timestamped dynamic data points recorded during the video capture, such as:
	- F-stop, exposure, and ISO settings
	- GPS position (if available)
	- Camera orientation and acceleration (certain models)

As an example of where this fails, popular image services such as Google Photos or Immich will not
read the GPS position from these videos, meaning they won't show up on a map view or location-based
searches.

Additionally, transcoding tools may also drop the metadata entirely. On top of likely losing the
GPS information in the root XML document, this usually also eliminates the metadata stream, which
can be very useful for applications such as [Gyroflow](https://gyroflow.xyz/) if your specific
model of camera records motion data.

## Sample Video

This video will be used in the examples below. It was recorded with an a6400 camera.

{%
	include embed/video.html
	src='/assets/posts/2025-05-03-Sony-Alpha-Video-Metadata/C0140.MP4'
	title='C0140.MP4'
%}

The format of the video is detailed below. The `rtmd` stream (`Stream #0:2[0x3]`) contains the
timestamped dynamic metadata.

```console
user@machine:~$ ffprobe C0140.MP4

Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'C0140.MP4':
  Metadata:
    compatible_brands: XAVCmp42iso2
    major_brand     : XAVC
    minor_version   : 16785407
    creation_time   : 2023-04-09T06:35:03.000000Z
    encoder         : Lavf60.16.100
  Duration: 00:00:21.03, start: 0.000000, bitrate: 22087 kb/s
  Stream #0:0[0x1](und): Video: hevc (Main) (hev1 / 0x31766568), yuv420p(tv, bt709/bt709/iec61966-2-4, progressive), 1920x1080 [SAR 1:1 DAR 16:9], 20055 kb/s, 59.94 fps, 59.94 tbr, 60k tbn (default)
    Metadata:
      creation_time   : 2023-04-09T06:35:03.000000Z
      handler_name    : Video Media Handler
      vendor_id       : [0][0][0][0]
      encoder         : AVC Coding
  Stream #0:1[0x2](und): Audio: pcm_s16be (ipcm / 0x6D637069), 48000 Hz, stereo, s16, 1536 kb/s (default)
    Metadata:
      creation_time   : 2023-04-09T06:35:03.000000Z
      handler_name    : Sound Media Handler
      vendor_id       : [0][0][0][0]
  Stream #0:2[0x3](und): Data: none (rtmd / 0x646D7472), 491 kb/s (default)
    Metadata:
      creation_time   : 2023-04-09T06:35:03.000000Z
      handler_name    : Timed Metadata Media Handler
      timecode        : 03:37:00:34
```

## Metadata Preservation

Many popular media tools, such as FFmpeg, will not recognize the metadata stream shown above, and
will simply drop it when transcoding. In the example below, the `rtmd` stream does not exist in the
transcoded video.

```console
user@machine:~$ ffmpeg -i C0140.MP4 -c:v libx265 -b:v 20M -c:a copy transcoded.MP4

Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'transcoded.MP4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2mp41
    encoder         : Lavf60.16.100
  Duration: 00:00:21.03, start: 0.000000, bitrate: 21104 kb/s
  Stream #0:0[0x1](und): Video: hevc (Main) (hev1 / 0x31766568), yuv420p(tv, bt709/bt709/iec61966-2-4, progressive), 1920x1080 [SAR 1:1 DAR 16:9], 19558 kb/s, 59.94 fps, 59.94 tbr, 60k tbn (default)
    Metadata:
      handler_name    : Video Media Handler
      vendor_id       : [0][0][0][0]
      encoder         : Lavc60.31.102 libx265
  Stream #0:1[0x2](und): Audio: pcm_s16be (ipcm / 0x6D637069), 48000 Hz, stereo, s16, 1536 kb/s (default)
    Metadata:
      handler_name    : Sound Media Handler
      vendor_id       : [0][0][0][0]
```

If you attempt to force FFmpeg to copy the stream using `-map 0`, you will receive an error about
an unknown format.

```console
user@machine:~$ ffmpeg -i C0140.MP4 -map 0 -c:v libx265 -b:v 20M -c:a copy transcoded.MP4

[mp4 @ 0x55c1d03f1440] Could not find tag for codec none in stream #2, codec not currently supported in container
[out#0/mp4 @ 0x55c1d03b5cc0] Could not write header (incorrect codec parameters ?): Invalid argument
Error while filtering: Invalid argument
[out#0/mp4 @ 0x55c1d03b5cc0] Nothing was written into output file, because at least one of its streams received no packets.
```

Using a tool like [MP4Box](https://wiki.gpac.io/MP4Box/MP4Box/), which operates at a lower level on
the MP4 container, we can copy the a meta box or data stream from the original file without knowing
the actual format of the data.

```bash
# Copy the timed metadata to the transcoded video
MP4Box -add C0140.MP4#3 transcoded.MP4

# Copy the metadata atom to the transcoded video
MP4Box -dump-xml /tmp/C0140.xml C0140.MP4
MP4Box -set-meta META -set-xml /tmp/C0140.xml transcoded.MP4
```

## Copying GPS Position to Standard Tags

Because the GPS position is not written to standard QuickTime tags, image storage services such as
Google Photos or Immich will not recognize it.

[ExifTool](https://exiftool.org/) is capable of reading many proprietary formats, including those
found in video from Sony cameras. Using the `-ee`/`-extractEmbedded` flag, it will extract the GPS
position from the embedded metadata, which will be exposed through the composite `GPSLatitude`,
`GPSLongitude`, and `GPSPosition` tags.

Immich actually uses ExifTool to read metadata, but will miss this information because it doesn't
use the required flags.

```console
user@machine:~$ exiftool -extractEmbedded --duplicates -location:all C0140.MP4

GPS Version ID                  : 2.2.0.0
GPS Latitude Ref                : North
GPS Longitude Ref               : East
GPS Status                      : Measurement Active
GPS Measure Mode                : 2-Dimensional Measurement
GPS Map Datum                   : WGS-84
GPS Latitude                    : 34 deg 41' 12.43" N
GPS Longitude                   : 135 deg 31' 32.02" E
GPS Position                    : 34 deg 41' 12.43" N, 135 deg 31' 32.02" E
```

Using ExifTool again, we can write this extracted information into standard tags that most tools
and services will recognize.

```bash
exiftool -extractEmbedded --duplicates '-GPSLatitude<$GPSLatitude' '-GPSLongitude<$GPSLongitude' '-UserData:GPSCoordinates<$GPSPosition' C0140.MP4
```

## Utility Scripts

[This repository](https://github.com/cbrl/sony-alpha-video-scripts) contains a set of scripts that
can be used to work with videos from Sony cameras.

> These scripts require FFmpeg, MP4Box, and ExifTool to be installed.
{: .prompt-info }

`video_convert.sh`{: .filepath }
: This script uses FFmpeg and MP4Box to transcode a video while preserving all metadata.

`copy_video_metadata.sh`{: .filepath }
: This script will copy metadata from an original source file to an already existing transcoded
file. This can be used for videos which were previously transcoded or if you don't want to use as
in the script above.

`video_gps_standardize.sh`{: .filepath }
: This will extract GPS coordinates from the embedded XML data and write them into more standard
GPS metadata tags, making it compatible with platforms like Google Photos or Immich.

`xml_to_xmp.py`{: .filepath }
: This can be used to convert Sony XML metadata to the XMP format. The XML data can be extracted
using the MP4Box command seen above, but is also usually found as a sidecar file alongside the
original video.

## Additional Notes

Currently, Gyroflow will not be able to load a transcoded video that had its metadata copied via
MP4Box. This is due to a bug in `mp4parse-rust`, as noted in
[this GitHub issue](https://github.com/mozilla/mp4parse-rust/issues/414). The `telemetry-parser`
project which uses `mp4parse-rust` has removed the problematic `meta-xml` feature as of
[this commit](https://github.com/AdrianEddy/telemetry-parser/commit/21eddb1ba3bf84c255041b3b7970474339eacbbb),
so some future version of Gyroflow should work correctly.
