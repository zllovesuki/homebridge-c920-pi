# homebridge-c920-pi

Refer to the [upstream](https://github.com/KhaosT/homebridge-camera-ffmpeg) for original instructions. This plugin is specifically optimized for C920 running on Raspberry Pi 3 Model B, preferably Raspian Stretch Lite.

## ffmpeg

C920 has H264 compressed stream support, so we could technically decode and re-encode (all HW accelerated).

However, the default `ffmpeg` was not compiled with any of those support...

1. Refers to [this link on SE](https://raspberrypi.stackexchange.com/a/70544) for `alsa` support, or else your stream will have no sound.
2. Refers to [this link on Reddit](https://www.reddit.com/r/raspberry_pi/comments/5677qw/hardware_accelerated_x264_encoding_with_ffmpeg/) for `omx` and `mmal` support, or else your RPi will be on fire and disintegrate.

## setup

For all intent and purposes, 720p stream is good enough. Load module `bcm2385-v4l2` with `sudo modprobe bcm2385-v4l2`, and make it load on boot with `sudo echo "bcm2385-v4l2" >> /etc/modules`

Then set the camera native resolution and stream format with `/usr/bin/v4l2-ctl --set-fmt-video=width=1280,height=720,pixelformat=1 -d /dev/video0`

## configuration

In `videoConfig`, we have a new format:

```JSON
"videoConfig": {
    "videoSource": "/dev/video0",
    "audioSource": "hw:1,0",
    "stillImageSource": "-i http://faster_still_image_grab_url/this_is_optional.jpg",
    "maxStreams": 2,
    "maxWidth": 1280,
    "maxHeight": 720,
    "maxFPS": 30
}
```
