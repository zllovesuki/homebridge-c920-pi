# homebridge-c920-pi

Refer to the [upstream](https://github.com/KhaosT/homebridge-camera-ffmpeg) for original instructions. This plugin is specifically optimized for C920 running on Raspberry Pi 3 Model B, preferably Raspian Stretch Lite.

## multi stream

Since my house has multiple people, I want the plugin to be able to handle multiple stream requests. Thus, we will have `v4l2rtspserver` grab the video feed, and stream it locally. Then, whenever users request for a live feed, it will grab from the local stream instead of taking control of the device.

Follow [this guide](http://c.wensheng.org/2017/05/18/stream-from-raspberrypi/) and compile `v4l2rtspserver`.

## ffmpeg

C920 has H264 compressed stream support, so we could technically decode and re-encode (all HW accelerated).

However, the default `ffmpeg` was not compiled with any of those support...

1. Refers to the first section of [this gist](https://gist.github.com/rafaelbiriba/7f2d7c6f6c3d6ae2a5cb) for `libfdk_aac` support
2. Refers to [this link on Reddit](https://www.reddit.com/r/raspberry_pi/comments/5677qw/hardware_accelerated_x264_encoding_with_ffmpeg/) for `omx` and `mmal` support, or else your RPi will be on fire and disintegrate.
3. You also need to include `--enable-network --enable-protocol=tcp --enable-demuxer=rtsp --enable-decoder=h264` when you are compiling or else you won't be able to use `rtsp` stream from `v4l2rtspserver` as an input.

## setup

For all intent and purposes, 720p stream is good enough. Load module `bcm2385-v4l2` with `sudo modprobe bcm2385-v4l2`, and make it load on boot with `sudo echo "bcm2385-v4l2" >> /etc/modules`

Then set the camera native resolution and stream format with `/usr/bin/v4l2-ctl --set-fmt-video=width=1280,height=720,pixelformat=1 -d /dev/video0`

v4l2server: `v4l2rtspserver -c -Q 512 -s -F 0 -H 720 -W 1280 -P 8555 -A 32000 -C 2 /dev/video0,hw:1,0`

## configuration

In `videoConfig`, point `source` to the `v4l2rtspserver`. ffmpeg should be able to demux. It has to look something like this: `-f rtsp -vcodec h264_mmal -i rtsp://127.0.0.1:8555/unicast`
