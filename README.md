## What is this
This is a docker container that runs headless Steam. Much of the steam-headless code is forked from https://github.com/Steam-Headless/docker-steam-headless
- Required programs are started/kept alive with supervisord.
- Inputs in the docker container come from the input-handler code which creates a virtual keyboard and mouse (eventually a gamepad) and connects to a hardcoded 'steam-test' room datachannel to receive inputs and pass them on using uinput.
- Inputs for the frontend (currently from a web view only) come from a hacked up version of livekit meet in the frontend/ dir. Changes mainly add code to capture keyboard/mouse inputs and send them to the datachannel
- Gstreamer is used along with the [livekitwebrtcsink](https://gstreamer.freedesktop.org/documentation/rswebrtc/livekitwebrtcsink.html?gi-language=c)
- the dockerfile downloads and compiles the livekitwebrtcsink plugin from [here](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs)
- support for hardware encoding currently requires this renegotiation to be commented out and the plugins recompiled [here](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/-/blob/main/net/webrtc/src/webrtcsink/imp.rs?ref_type=heads#L3274-3286)
- Gstreamer is hardcoded to connect to sfo3 DC and use the same 'steam-test' room. Gstreamer is using h264 hardware encoding enabled by cuda/GPU drivers.

## How to run
 Requires a VM with an attached GPU and installed drivers.
- `docker run -e ENV_LK_API_KEY={api-key} -e ENV_LK_SECRET_KEY={api-secret} --gpus all --privileged --device /dev/uinput --shm-size=4G -d livekit/steam-test:latest`

## How to build
- `docker build --platform linux/amd64 -t livekit/steam-test:latest -f Dockerfile.debian .`

## Troubleshooting
- installing the nvidia driver breaks the x11 config and currently have to take these steps to fix:
- make sure the input-handler is started
- run `evtest` to get a list of inputs, make sure that the virtual mouse/keyboard are defined correctly on the lines you add to xorg.conf in the next step
- manually edit `/etc/X11/xorg.conf` add the following lines, and then kill xorg. `ps aux | grep xorg` to find the PID. Supervisord will start it back up.
```
Section "InputDevice"
    Identifier "Keyboard0"
    Driver "evdev"
    Option "Device" "/dev/input/event4" # You need to identify the correct event number for your keyboard
    Option "AutoServerLayout" "on"
EndSection

Section "InputDevice"
    Identifier "Mouse0"
    Driver "evdev"
    Option "Device" "/dev/input/event5" # You need to identify the correct event number for your mouse
    Option "AutoServerLayout" "on"
EndSection

Section "InputDevice"
    Identifier "MyGamepad"
    Driver "evdev"
    Option "Device" "/dev/input/event6"
    Option "Name" "Gamepad"
EndSection
```
-- remove any Sections that were added by the driver install that start with `Section "InputDevice`, should only have the ones above defined.


## I want to help what should I do
- Current goal is an internal setup of this to support up to 4 user inputs. Then create a blog/marketing material around it for demo purposes.

TODO:
## Frontend
- add auth to the front-end code so we can deploy this for internal use. Maybe gmail auth based on livekit.io address?
- currently every user in the room sends inputs across the datachannel. Need to support up to four players and allow for an audience who's inputs aren't captured
- support gamepads (because i'm a couch gamer ok!)
## input-handler
- better support: keyboard and mouse tracking isn't great, could be refined a lot
- gamepad support
## backend
- fix issue with xorg.conf so that inputs don't have to manually be defined, and xorg restarted when input-handler restarted
- gstreamer renegotiation issue with hardware encoding
- make build lighter by using intermediate containers and only copying what we need into final docker container
