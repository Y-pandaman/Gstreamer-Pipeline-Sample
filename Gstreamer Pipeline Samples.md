# Gstreamer Pipeline Samples

Table of Contents:

- [TOC]

## Tips for Debug

- Set general debug level

  ```
  export GST_DEBUG=6 #5
  export GST_DEBUG=GST_REGISTRY:6,GST_PLUGIN:6
  ```

- Discover video file

  ```
  gst-discoverer-1.0 -v fat_bunny.ogg 
  ```

- Inspect plugin

  ```
  gst-inspect-1.0 avdec_h264
  ```

## Video

### display test video

```
gst-launch-1.0 -vvv videotestsrc ! autovideosink

gst-launch-1.0 -vvv videotestsrc ! 'video/x-raw,width=1280,height=720,format=RGB,framerate=60/1' ! fpsdisplaysink

gst-launch-1.0 -vvv videotestsrc ! videoconvert ! fpsdisplaysink text-overlay=false
```

### record to file

```
gst-launch-1.0 -vv videotestsrc ! x264enc ! flvmux ! filesink location=xyz.flv

gst-launch-1.0 -vv videotestsrc ! x264enc ! qtmux ! filesink location=xyz.mp4 -e
```

### record and display at the same time(queue)

[GStreamer Recording and Viewing Stream Simultaneously](https://stackoverflow.com/questions/37444615/gstreamer-recording-and-viewing-stream-simultaneously)

```
gst-launch-1.0 -vvv videotestsrc
! tee name=t
t. ! queue ! x264enc ! mp4mux ! filesink location=xyz.mp4 -e
t. ! queue leaky=1 ! autovideosink sync=false
```

**tips:**

1. `-e (EOS signal)`: Pipelines for file saving require a reliable EOS(End of Stream) signal
2. `queue leaky=1 ! autovideosink sync=false`: prevent blocking
3. `drop=true`: drop frame if cannot read quickly enough

### record webcam to `*.mp4`(jetson nano)

```
gst-launch-1.0 nvarguscamerasrc ! fakesink

gst-launch-1.0 nvarguscamerasrc num-buffers=2000 ! 'video/x-raw(memory:NVMM),width=1920, height=1080, framerate=30/1, format=NV12' ! omxh264enc ! qtmux ! filesink location=test.mp4 -e

gst-launch-1.0 nvarguscamerasrc num-buffers=2000 ! 'video/x-raw(memory:NVMM),width=1920, height=1080, framerate=30/1, format=NV12' ! nvvidconv ! x264enc ! qtmux ! filesink location=test.mp4 -e
```

### fps test

```
gst-launch-1.0 -v videotestsrc ! videorate ! video/x-raw,framerate=30/1 ! videoconvert ! autovideosink

gst-launch-1.0 -v videotestsrc ! videorate ! video/x-raw,framerate=30/1 ! x264enc tune=zerolatency bitrate=16000000 speed-preset=superfast ! rtph264pay ! udpsink port=5000 host=$HOST

gst-launch-1.0 -v videotestsrc ! videorate ! video/x-raw,framerate=60/1 ! x264enc tune=zerolatency bitrate=16000000 speed-preset=superfast ! h264parse ! rtph264pay ! udpsink port=5000 host=$HOST

gst-launch-1.0 -v videotestsrc ! video/x-raw,framerate=30/1 ! videorate ! video/x-raw,framerate=60/1 ! x264enc tune=zerolatency bitrate=16000000 speed-preset=superfast ! rtph264pay ! udpsink port=5000 host=$HOST
```

## Audio

```
gst-launch-1.0 audiotestsrc ! autoaudiosink

# This creates a higher beep:
gst-launch-1.0 audiotestsrc freq=1000 ! autoaudiosink

# volume
gst-launch-1.0 audiotestsrc volume=0.1 ! autoaudiosink

# White noise
gst-launch-1.0 audiotestsrc wave="white-noise" ! autoaudiosink

# Silence
gst-launch-1.0 audiotestsrc wave="silence" ! autoaudiosink
```

## Mux Video and Audio

```
gst-launch-1.0 -v audiotestsrc ! autoaudiosink videotestsrc ! autovideosink

# save to `*.mp4` file
gst-launch-1.0 -v -e videotestsrc \
  ! x264enc \
  ! mp4mux name=mux \
  ! filesink location="bla.mp4" audiotestsrc \
  ! lamemp3enc \
  ! mux.
```

videotestsrc --> x264enc -----
\>---> mp4mux ---> filesink audiotestsrc --> lamemp3enc --/

## Media File

```
# information of file
gst-discoverer-1.0 -v /home/jetson/Videos/why_I_left_China_for_Good.mp4
```

### Play Media File

```
# play `mp4` file with audio
gst-launch-1.0 filesrc location=/Users/siyao/Movies/why_I_left_China_for_Good.mp4 \
  ! qtdemux name=demux  demux.audio_0 \
  ! decodebin ! audioconvert ! audioresample ! autoaudiosink

# play `mp4` file with video
gst-launch-1.0 filesrc location=/Users/siyao/Movies/why_I_left_China_for_Good.mp4 \
  ! qtdemux \
  ! decodebin ! videoconvert ! videoscale ! autovideosink

# play `mkv` file with video
gst-launch-1.0 -v filesrc location=/Users/siyao/Movies/jellyfish-10-mbps-hd-h264.mkv \
  ! matroskademux name=demux  demux.video_0\
  ! decodebin \
  ! videoconvert \
  ! autovideosink

# play `mp4` file with both audio and video
gst-launch-1.0 filesrc location=/Users/siyao/Movies/why_I_left_China_for_Good.mp4 \
  ! qtdemux name=demux  demux.audio_0 \
  ! queue ! decodebin ! audioconvert ! audioresample ! autoaudiosink  demux.video_0 \
  ! queue ! decodebin ! videoconvert ! videoscale ! autovideosink
```

### Transcode Media File

```
# `mkv` to `mp4` with video
gst-launch-1.0 -v filesrc location=/home/jetson/Videos/jellyfish-10-mbps-hd-h264.mkv \
  ! matroskademux \
  ! h264parse \
  ! mp4mux \
  ! filesink location=jellyfish.mp4

# `mp4` to `mkv` with video
gst-launch-1.0 -v filesrc location=/Users/siyao/Movies/why_I_left_China_for_Good.mp4 \
  ! qtdemux name=demux matroskamux name=mux \
  ! filesink location=fat_bunny.mkv  demux.video_0 \
  ! queue ! h264parse ! mux.

# `mp4` to `mkv` with audio
gst-launch-1.0 -v filesrc location=/Users/siyao/Movies/why_I_left_China_for_Good.mp4 \
  ! qtdemux name=demux matroskamux name=mux \
  ! filesink location=fat_bunny.mka  demux.audio_0 \
  ! queue ! decodebin ! audioconvert ! vorbisenc ! mux.

# `mp4`to `mkv` with both audio and video
gst-launch-1.0 -v filesrc location=/Users/siyao/Movies/why_I_left_China_for_Good.mp4 \
  ! qtdemux name=demux matroskamux name=mux \
  ! filesink location=fat_bunny.mkv  demux.audio_0 \
  ! queue ! decodebin ! audioconvert ! audioresample ! vorbisenc ! mux.  demux.video_0 \
  ! queue ! h264parse ! mux.
```

## Network streaming

HOST=192.168.31.175 [//]: # (destination receiving the stream)

### Video RTP Streaming

- sender

#### send a test video with h264 rtp stream

```
gst-launch-1.0 -v videotestsrc ! x264enc tune=zerolatency bitrate=500 speed-preset=superfast ! rtph264pay ! udpsink port=5000 host=$HOST

gst-launch-1.0 -v videotestsrc pattern=snow ! "video/x-raw,format=I420,width=1920,height=1080,framerate=60/1" ! x264enc tune=zerolatency bitrate=500 speed-preset=superfast ! rtph264pay ! udpsink port=5000 host=$HOST

# grayscale
gst-launch-1.0 -v videotestsrc ! videobalance saturation=0 ! x264enc ! video/x-h264, stream-format=byte-stream ! rtph264pay ! udpsink host=$HOST port=5000

# control fps
gst-launch-1.0 -v videotestsrc ! videorate ! "video/x-raw,framerate=60/1" ! x264enc tune=zerolatency bitrate=16000000 speed-preset=superfast ! h264parse ! rtph264pay pt=96 ! udpsink port=5000 host=$HOST

gst-launch-1.0 -v videotestsrc ! videorate ! "video/x-raw,framerate=60/1" ! omxh264enc insert-sps-pps=true bitrate=16000000 ! h264parse ! rtph264pay pt=96 ! udpsink port=5000 host=$HOST

gst-launch-1.0 -v videotestsrc ! videorate ! "video/x-raw,width=1280,height=720,framerate=60/1" ! omxh264enc insert-sps-pps=true bitrate=16000000 ! h264parse ! rtph264pay pt=96 ! udpsink port=5000 host=$HOST
```

#### send a screen capture with h264 rtp stream(linux)

```
gst-launch-1.0 -v ximagesrc ! video/x-raw,framerate=20/1 ! videoscale ! videoconvert ! x264enc tune=zerolatency bitrate=500 speed-preset=superfast ! rtph264pay ! udpsink host=127.0.0.1 port=5000
```

#### send a screen capture with h264 rtp stream(osx)

```
gst-launch-1.0 -v avfvideosrc capture-screen=true ! video/x-raw,framerate=20/1 ! videoscale ! videoconvert ! x264enc tune=zerolatency bitrate=500 speed-preset=superfast ! rtph264pay ! udpsink host=127.0.0.1 port=5000
```

#### send a `*.mp4` file with h264 rtp stream

```
gst-launch-1.0 -v filesrc location=test.mp4 ! qtdemux ! h264parse ! omxh264dec ! x264enc ! rtph264pay ! udpsink host=$HOST port=5000

gst-launch-1.0 -v filesrc location=test.mp4 ! qtdemux ! h264parse ! omxh264dec ! omxh264enc insert-sps-pps=true ! rtph264pay ! udpsink host=$HOST port=5000

gst-launch-1.0 -v filesrc location=test.mp4 ! qtdemux ! h264parse ! rtph264pay ! udpsink host=$HOST port=5000
```

#### send a webcam with h264 rtp stream(jetson nano)

```
!!slow
gst-launch-1.0 nvarguscamerasrc ! 'video/x-raw(memory:NVMM),width=1280, height=720, framerate=60/1, format=NV12' ! nvvidconv ! x264enc bitrate=16000000 speed-preset=superfast ! rtph264pay ! udpsink port=5000 host=$HOST

!!fast
gst-launch-1.0 nvarguscamerasrc ! 'video/x-raw(memory:NVMM),width=1920, height=1080, framerate=30/1, format=NV12' ! nvvidconv flip-method=2 ! nvv4l2h264enc insert-sps-pps=true bitrate=16000000 ! rtph264pay ! udpsink port=5000 host=$HOST

gst-launch-1.0 nvarguscamerasrc ! 'video/x-raw(memory:NVMM),width=1920, height=1080, framerate=30/1, format=NV12' ! omxh264enc insert-sps-pps=true bitrate=16000000 ! rtph264pay ! udpsink port=5000 host=$HOST

# avoid receiver using sync=false
```

#### send a webcam with h264 rtp stream while record (jetson nano)

```
gst-launch-1.0 nvarguscamerasrc sensor-id=0 ! 'video/x-raw(memory:NVMM),width=1920, height=1080, framerate=30/1, format=NV12' ! nvvidconv flip-method=2 ! omxh264enc insert-sps-pps=true bitrate=16000000 \
    ! tee name=t \
    t. ! queue ! rtph264pay ! udpsink port=5000 host=$OSX_HOST \
    t. ! queue ! qtmux ! filesink location=test.mp4 -e
```

- receiver

#### receive h264 rtp stream(desktop host)

```
# display
gst-launch-1.0 -v udpsrc port=5000 ! "application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" ! rtph264depay ! h264parse ! decodebin ! videoconvert ! autovideosink sync=false

gst-launch-1.0 -v udpsrc port=5000 ! "application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" ! rtph264depay ! h264parse ! avdec_h264 ! videoconvert ! autovideosink

# fps
gst-launch-1.0 -v udpsrc port=5000 ! "application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" ! rtph264depay ! h264parse ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false

# record and display
gst-launch-1.0 -v udpsrc port=5000 ! "application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" \
! rtph264depay ! h264parse \
! tee name=t \
t. ! queue ! mp4mux ! filesink location=xyz.mp4 -e \
t. ! queue leaky=1 ! decodebin ! videoconvert ! autovideosink sync=false
```

> **tips**:
>
> 1. `sync=false`: remove dropped frame rate, lag and error message(`There may be a timestamping problem, or this computer is too slow`). see the [Streaming RTP/RTSP: sync/timestamp problems](https://stackoverflow.com/questions/11397655/streaming-rtp-rtsp-sync-timestamp-problems) for details.
> 2. `udpsrc port=5000 cap='application/x-rtp'`: will set the remaining properties to default.

### Audio RTP Streaming

#### raw audio

```
# sender
gst-launch-1.0 audiotestsrc \
  ! audioconvert \
  ! audioresample \
  ! rtpL24pay \
  ! udpsink host=$HOST port=5001

# receiver
gst-launch-1.0 -v udpsrc port=5001 \
  ! 'application/x-rtp,media=audio,payload=96,clock-rate=44100,encoding-name=L24' \
  ! rtpL24depay \
  ! audioconvert \
  ! autoaudiosink sync=false
```

#### encoded/decoded audio

```
# sender
gst-launch-1.0 audiotestsrc \
  ! audioconvert \
  ! audioresample \
  ! alawenc \
  ! rtppcmapay \
  ! udpsink host=$HOST port=5001

# receiver
gst-launch-1.0 -v udpsrc port=5001 \
  ! 'application/x-rtp,media=audio,payload=8,clock-rate=8000,encoding-name=PCMA'  \
  ! rtppcmadepay \
  ! alawdec \
  ! audioconvert \
  ! autoaudiosink sync=false
```

### Audio + Video RTP Streaming

```
# sender
gst-launch-1.0 -v videotestsrc \
  ! 'video/x-raw,format=I420,width=1920,height=1080,framerate=60/1' \
  ! omxh264enc insert-sps-pps=true bitrate=16000000 \
  ! h264parse \
  ! rtph264pay pt=96 \
  ! udpsink port=5000 host=$HOST audiotestsrc \
  ! audioconvert \
  ! audioresample \
  ! rtpL24pay \
  ! udpsink host=$HOST port=5001


# receiver
gst-launch-1.0 -v udpsrc port=5000 \
  ! 'application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96' \
  ! rtph264depay \
  ! h264parse \
  ! decodebin \
  ! videoconvert \
  ! fpsdisplaysink text-overlay=false sync=false \
  udpsrc port=5001 \
  ! "application/x-rtp,channels=1,media=audio,payload=96,clock-rate=44100,encoding-name=L24" \
  ! rtpL24depay \
  ! audioconvert \
  ! autoaudiosink sync=false

#TODO: SDP file
v=0
c=IN IP4 <Receiver IP>
m=video 5000 RTP/AVP 96
a=recvonly
a=rtpmap:96 H264/90000
m=audio 5001 RTP/AVP 8
a=recvonly
a=rtpmap:8 PCMA/8000/1
```

### Media File RTP Streaming

#### demux and split video and audio(not best)

```
# sender `mp4` file
gst-launch-1.0 -v filesrc location=/home/jetson/Videos/why_I_left_China_for_Good.mp4 \
  ! qtdemux name=demux  \
  demux.video_0 ! queue ! h264parse ! decodebin ! omxh264enc insert-sps-pps=true bitrate=16000000 ! rtph264pay pt=96 ! udpsink host=$HOST port=5000  \
  demux.audio_0 ! queue ! decodebin ! audioconvert ! audioresample ! avenc_ac3 ! rtpac3pay ! udpsink host=$HOST port=5001

# receiver
gst-launch-1.0 -v \
  udpsrc port=5000 \
  ! 'application/x-rtp,media=(string)video,clock-rate=(int)90000,encoding-name=(string)H264,payload=(int)96' \
  ! rtph264depay ! h264parse ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false  \
  udpsrc port=5001 \
  ! 'application/x-rtp,media=(string)audio,clock-rate=(int)44100,encoding-name=(string)AC3,payload=(int)96' \
  ! rtpac3depay ! ac3parse ! decodebin ! audioconvert ! autoaudiosink sync=false

# receiver(short version)
gst-launch-1.0 -v \
  udpsrc port=5000 caps='application/x-rtp' \
  ! rtph264depay ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false  \
  udpsrc port=5001 caps='application/x-rtp' \
  ! rtpac3depay ! decodebin ! audioconvert ! autoaudiosink sync=false
```

#### remux to MPEGST(best)

It has several advantages like good re-synchronization after packet loss and that A/V always stays sync

##### mpegts sender

```
# sender `mp4` file
gst-launch-1.0 -v filesrc location=/home/jetson/Videos/why_I_left_China_for_Good.mp4 \
  ! qtdemux name=demux mpegtsmux name=mux alignment=7 \
  ! rtpmp2tpay \
  ! udpsink host=$HOST port=5000 demux. \
  ! queue ! h264parse ! decodebin ! omxh264enc insert-sps-pps=true bitrate=16000000 ! mux. demux. \
  ! queue ! aacparse ! mux.
```

##### mpegts receiver

```
# receiver
gst-launch-1.0 -v udpsrc port=5000 \
  ! 'application/x-rtp,clock-rate=(int)90000,media=(string)video,payload=(int)96,encoding-name=(string)MP2T' \
  ! rtpmp2tdepay ! tsparse ! tsdemux name=demux \
  demux. ! queue ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false \
  demux. ! queue ! decodebin ! audioconvert ! autoaudiosink sync=false

# receiver(short version)
gst-launch-1.0 -v udpsrc port=5000 caps="application/x-rtp" \
  ! rtpmp2tdepay ! tsparse ! tsdemux name=demux \
  demux. ! queue ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false \
  demux. ! queue ! decodebin ! audioconvert ! autoaudiosink sync=false

# receiver(`ts` file)
gst-launch-1.0 -v -e udpsrc port=5000 caps="application/x-rtp" \
  ! rtpmp2tdepay ! tsparse ! filesink location=foo.ts
```

> `tsdemux demux.video_0` not work, see the details belowing: You're using a name (video_0) that doesn't exist. The pad names of demuxers vary from demuxer to demuxer, and might sometimes also differ from version to version. In this case that first video pad is called video_0031 as you can see at the end. It's generally best not to specify pad names unless you have a good reason to do so.

### RTSP Server

#### install server(jetson nano)

```
# install gst-rtsp-server libs
sudo apt-get install libgstrtspserver-1.0 libgstreamer1.0-dev

# install gst-rtsp-server test server(1.14 is your gst version, checked by gst-lanch-1.0 --version)
wget https://raw.githubusercontent.com/GStreamer/gst-rtsp-server/1.14/examples/test-launch.c
gcc test-launch.c -o test-launch $(pkg-config --cflags --libs gstreamer-1.0 gstreamer-rtsp-server-1.0)
```

#### stream web-cam(jetson nano)

```
# server
./test-launchh "nvarguscamerasrc ! video/x-raw(memory:NVMM), format=NV12, width=1920, height=1080, framerate=30/1 ! nvvidconv ! video/x-raw, width=640, height=480, format=NV12, framerate=30/1 ! omxh265enc ! rtph265pay name=pay0 pt=96 config-interval=1"

# client
gst-launch-1.0 rtspsrc location=rtsp://jetson-desktop.local:8554/test  ! decodebin ! videoconvert ! autovideosink
```

#### stream media file(*.mp4)

1. video

   ```
   # server
   ./test-launch "filesrc location=/home/jetson/Videos/why_I_left_China_for_Good.mp4 \
     ! qtdemux \
     ! h264parse \
     ! decodebin \
     ! videoconvert \
     ! omxh264enc insert-sps-pps=true bitrate=16000000 \
     ! rtph264pay name=pay0"
   
   # client
   gst-launch-1.0 -v rtspsrc location=rtsp://jetson-desktop.local:8554/test \
     ! rtph264depay \
     ! h264parse \
     ! decodebin \
     ! videoconvert \
     ! autovideosink sync=false
   ```

2. audio

   ```
   # server
   ./test-launch "filesrc location=/home/jetson/Videos/why_I_left_China_for_Good.mp4 \
     ! qtdemux \
     ! aacparse \
     ! decodebin \
     ! audioconvert \
     ! avenc_aac \
     ! rtpmp4apay name=pay0"
   
   # client
   gst-launch-1.0 -v rtspsrc location=rtsp://jetson-desktop.local:8554/test \
     ! rtpmp4adepay \
     ! aacparse \
     ! decodebin \
     ! audioconvert \
     ! autoaudiosink sync=false
   ```

3. video + audio

   h264 + acc,

   ```
   # server
   ./test-launch "filesrc location=/home/jetson/Videos/why_I_left_China_for_Good.mp4 \
     ! qtdemux name=demux \
     demux. ! queue ! h264parse ! decodebin ! omxh264enc insert-sps-pps=true bitrate=16000000 ! rtph264pay name=pay0 \
     demux. ! queue ! aacparse ! decodebin ! audioconvert ! avenc_aac ! rtpmp4apay name=pay1"
   
   # client
   gst-launch-1.0 -v rtspsrc location=rtsp://jetson-desktop.local:8554/test name=src \
     src. ! rtph264depay ! h264parse ! decodebin ! videoconvert ! autovideosink sync=false \
     src. ! rtpmp4adepay ! aacparse ! decodebin ! audioconvert ! autoaudiosink sync=false
   
   # client(short)
   gst-launch-1.0 -v rtspsrc location=rtsp://jetson-desktop.local:8554/test name=src \
     src. ! decodebin ! videoconvert ! autovideosink sync=false \
     src. ! decodebin ! audioconvert ! autoaudiosink sync=false
   ```

   mpegts,

   ```
   # server
   ./test-launch "filesrc location=/home/jetson/Videos/why_I_left_China_for_Good.mp4 \
     ! qtdemux name=demux mpegtsmux name=mux alignment=7 \
     ! rtpmp2tpay name=pay0 demux. \
     ! queue ! h264parse ! decodebin ! omxh264enc insert-sps-pps=true bitrate=16000000 ! mux. demux. \
     ! queue ! aacparse ! decodebin ! audioconvert ! avenc_aac ! mux."
   
   # client
   gst-launch-1.0 -v rtspsrc location=rtsp://jetson-desktop.local:8554/test \
     ! rtpmp2tdepay ! tsparse ! tsdemux name=demux \
     demux. ! queue ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false \
     demux. ! queue leaky=1 ! decodebin ! audioconvert ! autoaudiosink sync=false
   
   # client(only video)
   gst-launch-1.0 -v rtspsrc location=rtsp://jetson-desktop.local:8554/test name=src \
     src. ! rtpmp2tdepay ! tsparse ! tsdemux name=demux ! h264parse ! avdec_h264 ! videoconvert ! autovideosink sync=false
   
   # client(only audio)
   gst-launch-1.0 -v rtspsrc location=rtsp://jetson-desktop.local:8554/test \
     ! rtpmp2tdepay ! tsparse ! tsdemux name=demux ! aacparse ! decodebin ! audioconvert ! autoaudiosink sync=false
   ```

## Mixing

### Mixing videos

```
# display
gst-launch-1.0 -e compositor name=comp sink_0::alpha=0.7 sink_1::alpha=0.5 \
    ! autovideosink \
    videotestsrc \
        ! video/x-raw, framerate=10/1, width=640, height=360 \
        ! comp.sink_0 \
    videotestsrc pattern="snow" \
        ! video/x-raw, framerate=10/1, width=200, height=150 \
        ! comp.sink_1
```

video + rtp/udp,

```
# sender
gst-launch-1.0 -e compositor name=comp sink_0::alpha=0.7 sink_1::alpha=0.5 \
    ! videoconvert ! omxh264enc insert-sps-pps=true bitrate=16000000 ! h264parse ! rtph264pay ! udpsink port=5000 host=$HOST \
    videotestsrc \
        ! video/x-raw, framerate=10/1, width=640, height=360 \
        ! comp.sink_0 \
    videotestsrc pattern="snow" \
        ! video/x-raw, framerate=10/1, width=200, height=150 \
        ! comp.sink_1
```

[receiver](https://gist.github.com/liviaerxin/bb34725037fd04afa76ef9252c2ee875#receive-h264-rtp-stream)

### Mixing video+audio

sender(mpegts + rtp/udp),

```
# sender(videotestsrc/audiotestsrc)
gst-launch-1.0 -v \
    mpegtsmux name=mux alignment=7 ! rtpmp2tpay \
    ! udpsink port=5000 host=$HOST sync=false \
    compositor name=videomix ! x264enc ! queue ! mux. \
    audiomixer name=audiomix ! audioconvert ! audioresample ! avenc_ac3 ! queue ! mux. \
    videotestsrc pattern=ball ! videomix. \
    videotestsrc pattern=pinwheel ! videoscale ! video/x-raw,width=100 ! videomix. \
    audiotestsrc freq=400 ! audiomix. \
    audiotestsrc freq=600 ! audiomix.

# sender(mp4 file)
SRC=/home/jetson/Videos/why_I_left_China_for_Good.mp4
gst-launch-1.0 -v \
    mpegtsmux name=mux alignment=7 ! rtpmp2tpay \
    ! udpsink port=5000 host=$HOST sync=false \
    compositor name=videomix ! omxh264enc bitrate=16000000 ! queue2 ! mux. \
    audiomixer name=audiomix ! audioconvert ! audioresample ! avenc_ac3 ! queue2 ! mux. \
    filesrc location=$SRC ! qtdemux name=demux \
    demux.video_0 ! queue2 ! h264parse ! omxh264dec ! videoconvert ! videomix. \
    demux.audio_0 ! queue2 ! decodebin ! audioconvert ! audioresample ! audiomix. \
    videotestsrc pattern=ball ! videoscale ! video/x-raw,width=100,height=100 ! videomix. \
    audiotestsrc freq=600 volume=0.1 ! audiomix.

# sender(mp4 file) `uridecodebin`
gst-launch-1.0 \
    mpegtsmux name=mux ! rtpmp2tpay \
    ! udpsink port=5000 host=$HOST sync=false \
    compositor name=videomix ! x264enc ! queue2 ! mux. \
    audiomixer name=audiomix ! audioconvert ! audioresample ! avenc_ac3 ! queue2 ! mux. \
    uridecodebin uri=file://$SRC name=dec \
    dec. ! queue2 ! audioconvert ! audioresample ! audiomix. \
    dec. ! queue2 ! nvvidconv ! videoconvert ! video/x-raw,width=640,height=360 ! videomix. \
    videotestsrc pattern=ball ! video/x-raw,width=100,height=100 ! videomix. \
    audiotestsrc freq=400 volume=0.2 ! audiomix.
```

tcp server,

```
# View this in VLC with tcp://localhost:7001
gst-launch-1.0 \
    mpegtsmux name=mux ! rtpmp2tpay ! \
    tcpserversink port=7001 host=0.0.0.0 recover-policy=keyframe sync-method=latest-keyframe sync=false \
    compositor name=videomix ! x264enc ! queue2 ! mux. \
    audiomixer name=audiomix ! audioconvert ! audioresample ! avenc_ac3 ! queue2 ! mux. \
    uridecodebin uri=file://$SRC name=demux ! \
    queue2 ! audioconvert ! audioresample ! audiomix. \
    demux. ! queue2 ! decodebin ! nvvidconv ! videoconvert ! videoscale ! video/x-raw,width=640,height=360 ! videomix. \
    videotestsrc pattern=ball ! videoscale ! video/x-raw,width=100,height=100 ! videomix. \
    audiotestsrc freq=400 volume=0.2 ! audiomix.
```

> **tips** for jetson nano, use `nvvidconv` instead of `videoconvert` because `decodebin` use `nvv4l2decoder` to decode H.264 in default. Otherwise, explictly decode H.264 with `omxh264dec`.

receiver,

```
# receiver
[mpegts receiver](#mpegts-receiver)
gst-launch-1.0 -v udpsrc port=5000 caps="application/x-rtp" \
  ! rtpmp2tdepay ! tsparse ! tsdemux name=demux \
  demux. ! queue ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false \
  demux. ! queue leaky=1 ! decodebin ! audioconvert ! autoaudiosink sync=false
```

## SHM(Shared Memory)

### raw video

```
# send
gst-launch-1.0 -v videotestsrc is-live=true ! 'video/x-raw,width=1280,height=720,format=(string)RGB,framerate=(fraction)60/1' ! videoconvert ! shmsink socket-path=/tmp/foo name=/tmp/shm sync=false wait-for-connection=false shm-size=20000000

# receive
gst-launch-1.0 -v shmsrc do-timestamp=true socket-path=/tmp/foo name=/tmp/shm ! 'video/x-raw,width=1280,height=720,format=(string)RGB,framerate=(fraction)60/1' ! videoconvert ! fpsdisplaysink text-overlay=false sync=false -e
```

**tips:**

1. `do-timestamp=true`: solve `gst_segment_to_stream_time: assertion 'segment->format == format' failed` error. Seeing at[Getting shmsink/shmsrc to work with videomixer](https://mazdermind.wordpress.com/2014/08/29/getting-shmsinkshmsrc-to-work-with-videomixer/)
2. In local host, use `raw video` faster than `h264` due to memory cheaper/faster that cpu
3. increase `shm-size` for large image

### h264 encode/decode

```
# send
gst-launch-1.0 -v videotestsrc ! x264enc ! shmsink socket-path=/tmp/foo sync=false wait-for-connection=false shm-size=10000000

gst-launch-1.0 -v videotestsrc ! 'video/x-raw,width=1280,height=720,format=NV12,framerate=60/1' ! x264enc ! shmsink socket-path=/tmp/foo sync=false wait-for-connection=false shm-size=10000000

# receive
gst-launch-1.0 -v shmsrc socket-path=/tmp/foo ! h264parse ! decodebin ! videoconvert ! fpsdisplaysink text-overlay=false sync=false
```

## python-opencv

```
import cv2
import time
# capture

# videotestsrc
gst_str = 'videotestsrc ! video/x-raw,framerate=20/1 ! videoscale ! videoconvert ! appsink'

# udpsrc
gst_str = 'udpsrc port=5000 ! "application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" ! rtph264depay ! h264parse ! decodebin ! videoconvert ! autovideosink'

gst_str = 'udpsrc port=5000 caps = "application/x-rtp, media=(string)video, clock-rate=(int)90000, encoding-name=(string)H264, payload=(int)96" ! rtph264depay ! decodebin ! videoconvert ! appsink'

cap = cv2.VideoCapture(gst_str, cv2.CAP_GSTREAMER)

cap.isOpened()
ret, frame = cap.read()
cv2.imwrite("result.jpg", frame)
cap.release()

# write

# udpsink
gst_str = 'appsrc ! videoconvert ! x264enc tune=zerolatency bitrate=500 speed-preset=superfast ! rtph264pay ! udpsink host=127.0.0.1 port=5000'

fps = 30
frame_width = 640
frame_height = 480
out = cv2.VideoWriter(gst_str, cv2.CAP_GSTREAMER, 0, fps, (frame_width, frame_height), True)
out.isOpened()

img = cv2.imread('WechatIMG1.jpeg')
img.shape
img = cv2.resize(img, (frame_width, frame_height))
img.shape
cv2.imwrite("result.jpg", img)
while True:
    out.write(img)
    time.sleep(1/60)

out.release()
```