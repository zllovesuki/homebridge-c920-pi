# homebridge-camera-multistream

Refer to the [upstream](https://github.com/KhaosT/homebridge-camera-ffmpeg) for original instructions. This plugin is specifically optimized for C920 running on Raspberry Pi 3 Model B, preferably Raspian Stretch Lite.

## why fork?

Since multiple people are living in my house, I want the plugin to be able to handle multiple stream requests. Thus, we will have `v4l2rtspserver` grab the video feed, and stream it locally on RPi. Then, whenever users request for a live feed, it will request from the local live stream instead of taking control of the device.

Or, in a simpler term, turning your USB Webcam into an IP Camera, and have fun with it. (Thus strictly speaking this will work with any other webcams that outputs H264 stream. MJPEG will work too but you will have to purchase the license for your RPi for the MPEG hardware decode.)

## configure your RPi

Since we are going to use GPU decoding/encoding extensively, make sure to configure your RPi to have 256MB available to GPU.

## compile: v4l2rtspserver

Follow [this guide](http://c.wensheng.org/2017/05/18/stream-from-raspberrypi/) and compile `v4l2rtspserver`.

## compile: ffmpeg

We *NEED* to compile ffmpeg with hardware acceleration support. Technically, one could use `avconv` and everyone will be happy. Unfortunately, `avconv` was not compiled with `libfdk_aac` support, meaning that our stream will not have audio.

Moving on to compile ffmpeg...

1. Refers to the first section of [this gist](https://gist.github.com/rafaelbiriba/7f2d7c6f6c3d6ae2a5cb) for `libfdk_aac` support
2. Refers to [this link on Reddit](https://www.reddit.com/r/raspberry_pi/comments/5677qw/hardware_accelerated_x264_encoding_with_ffmpeg/) for `omx` and `mmal` support, or else your RPi will be on fire and disintegrate.
3. You also need to include `--enable-network --enable-protocol=tcp --enable-demuxer=rtsp --enable-decoder=h264` when you are compiling or else you won't be able to use `rtsp` stream from `v4l2rtspserver` as an input.

To make you life easier, this is my `configure` flags:
```
--enable-mmal --enable-omx-rpi --enable-nonfree --enable-gpl --enable-libfdk-aac --enable-network --enable-protocol=tcp --enable-demuxer=rtsp --enable-decoder=h264
```

## setup: streaming server

Unfortunately, RPi's hardware decoder/encoder cannot handle more than one 1080p stream. Thus, we are forced to use 720p stream.

Load module `bcm2385-v4l2` with `sudo modprobe bcm2385-v4l2`, and make it load on boot with `sudo echo "bcm2385-v4l2" >> /etc/modules`

Then set the camera native resolution and stream format with `/usr/bin/v4l2-ctl --set-fmt-video=width=1280,height=720,pixelformat=1 -d /dev/video0`

v4l2server: `v4l2rtspserver -c -Q 512 -s -F 0 -H 720 -W 1280 -I 127.0.0.1 -P 8555 -A 32000 -C 2 /dev/video0,hw:1,0`

Or, refer to `v4l2rtspserver.service` for systemd configuration

*Explanation for the curious*:
- `-c` don't repeat config (default repeat config before IDR frame)
- `-Q 512` we want to increase the queue length
- `-s` use live555 main loop so we don't have to get a new thread to do all the work
- `-F 0` use native framerate
- `-H 720 -W 1280` this should be pretty self explanatory
- `-I 127.0.0.1 -P 8555` we only listen on localhost, so pervs cannot watch us
- `-A 32000 -C 2` this is the audio setting. C920 samples at 32000 MHz so we adjust accordingly
- `/dev/video0,hw:1,0` video device and the audio device

---

However, if you only expect one stream at any given time, you can change it to 1080p.

## setup: snapshot

In the upstream, ffmpeg will take control of the device to snapshot and return a still image. However, that will take control of the device and it doesn't work like that. Since we are using streaming server, (see caveats below), it takes too long to get a still image. Thus, we will have a dedicated ffmpeg process to handle snapshotting for us.

One advantage of such is that you are easily implement your favorite tools to save periodic images to your favorite storage (Minio, S3, FreeNAS, etc).

ffmpeg: `ffmpeg -f rtsp -vcodec h264_mmal -i rtsp://127.0.0.1:8555/unicast -vf fps=fps=1/5 -f image2 -update 1 /dev/shm/latest.jpg`

Or, refer to `snapshot.service` for systemd configuration

*Explanation for the curious*:
- `-f rtsp -vcodec h264_mmal -i rtsp://127.0.0.1:8555/unicast` that's our live stream
- `-vf fps=fps=1/5` instruct ffmpeg to take a snapshot every 5 seconds
- `-f image2` output as jpeg
- `-update 1` instruct ffmpeg to overwrite the same image
- `/dev/shm/latest.jpg` write to ram disk so your SD card doesn't explode

## configuration

```json
{
    "platform": "camera-multistream",
    "cameras": [
        {
            "name": "Garage",
            "videoConfig": {
                "source": "-f rtsp -vcodec h264_mmal -i rtsp://127.0.0.1:8555/unicast",
                "stillImageSource": "-i /dev/shm/latest.jpg",
                "maxStreams": 2,
                "maxWidth": 1280,
                "maxHeight": 720,
                "maxFPS": 30
            }
        }
    ]
}
```

## caveats/notes

1. Using RTSP + h264_mmal will increase the time to initialize the stream. In my case it takes 15 seconds for the video to start playing (as opposed to almost instantly with direct access to `/dev/video0`). (However still image are fine since it is handled by a separate process.) I'm still trying to figure out the way.
2. RTSP does complain with `max delay reached. need to consume packet`. However this only happens upon initialization. There's no problem after the video/audio streams started.
3. Depending on the movement of the stars, the direction of the wind, and the amount of water your drank, RTP might complained with `Non-monotonous DTS in output stream 1:0; previous: 605510, current: 559023; changing to 605510. This may result in incorrect timestamps in the output file`. *However this doesn't seem to affect the stream.* I'm still trying to investigate that.
