# resilient_srt_streamer
A command-line tool built using GStreamer (C++) that streams a local video file over SRT (Secure Reliable Transport) with:

- Low-latency H.264 encoding
- Real-time SRT statistics (RTT, Packet Loss)
- Automatic fallback stream on failure
- Configurable encoder parameters

## Prerequisites

Ensure the following are installed:

sudo apt-get update
sudo apt install -y g++ pkg-config \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev \
    gstreamer1.0-tools \
    gstreamer1.0-libav \
    gstreamer1.0-plugins-base \
    gstreamer1.0-plugins-good \ 
    gstreamer1.0-plugins-bad  \
    gstreamer1.0-plugins-ugly \ 

## Build Instructions
g++ main.cpp -o srt_streamer `pkg-config --cflags --libs gstreamer-1.0 gstreamer-plugins-base-1.0`

## Usage
### Sender Application:
./srt_streamer <input_video>

Example:
./srt_streamer sample.mp4

### Encoder Configuration (Optional)

You can override default x264 parameters:

#### Supported Parameters
Flag	    Description	            Default
--bitrate	Video bitrate (kbps)	4000
--preset	x264 speed preset (ultrafast, veryfast, faster)	ultrafast
--gop	Keyframe interval (frames)	30


Example:
./srt_streamer input.mp4 --bitrate 5000 --preset veryfast --gop 60

### Receiver (Test Command)

Run this in a separate terminal:

gst-launch-1.0 srtsrc uri=srt://:9000?mode=listener \
! queue \
! tsdemux \
! queue \
! decodebin \
! queue \
! videoconvert \
! queue \
! ximagesink