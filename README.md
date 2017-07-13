# How-To: Select audio source from Linux command line (Debian 8.8)

I spent way to much time digging into ALSA configuration on this issue not
to write something up to remember later on. I recently picked up an audio
interface to both act as a microphone preamp and also a headphone amp for
my ~2008 Lenovo laptop. The goal was mostly to eliminate the noise in
the headphone output that is present on the analog audio out of the laptop.
This is especially worse when plugged into the wall because of what appears
to be a bad case of ground loop.

The audio interface was a `Behringer U-PHORIA UMC202HD`. I am running Debian 8
with the [DWM](http://dwm.suckless.org/) window manager and some [Mate](http://mate-desktop.org/) 
and [XFCE4](http://www.xfce.org/) utilities thrown in the mix... Need to clean that up.

After plugging in the interface I could see it get loaded in `dmesg` and listed in both
`lsusb` and `aplay -l`. But my audio was still coming out of the primary built-in 
audio card:

```
card 0: Intel [HDA Intel], device 0: CX20561 Analog [CX20561 Analog]
```

What I wanted is to switch to the interface:

```
card 1: U192k [UMC202HD 192k], device 0: USB Audio [USB Audio]
```

Since I figured that ALSA was managing these devices I dug into the [ALSA Docs](https://wiki.archlinux.org/index.php/Advanced_Linux_Sound_Architecture)
at Arch's Wiki. Because I never read a wall of text, I just started picking and choosing 
commands that made sense to me. To start with, I did a `speaker-test` to see if I could
even play anything from the interface:

To do this, first you need to find the device name. This is different than what you
see in `aplay -l`. That only shows the card info and device info that is not specific 
enough. So use `aplay -L` (captial L) to get what is really needed.

```
$ aplay -L
default
    Playback/recording through the PulseAudio sound server
null
    Discard all samples (playback) or generate zero samples (capture)
pulse
    PulseAudio Sound Server
equal
sysdefault:CARD=Intel
    HDA Intel, CX20561 Analog
    Default Audio Device

... #Lots of other things like `dmix` sw mixer and `dsnoop`, or my HDMI monitor sound device ...

# Here it is:
sysdefault:CARD=U192k
    UMC202HD 192k, USB Audio
    Default Audio Device

...
```
Now that I had the device name I can send a test signal to the two channels and verify
that I can even use the interface without anymore configuration.

```
speaker-test -D sysdefault:CARD=U192k -c 2 -l 2
```

This plays two passes of pink noise on the headphone and output ports\* of the device. (Note: I
only actually tested the headphone port and am just assuming the outputs in back also work since
that is how I interpret the specification for this interface.)

So now I know it works, now I just need to make this my default output. I discovered that I could
force `mplayer` to output to this by specifying which card I wanted to use:

```
$ mplayer --ao=alsa:device=hw=1 <mp3 file>
```

This actually took me a while to do correctly because I was using different
permutations of the device name given by `aplay -L` and nothing seemed to work
so basically I just grab the example from the `manpage` and tweaked it to fit my
device number. 

So now I know music can play, I just need to make this my default output. At this
point I wasted more time than I want to admit with different settings in my `~/.asoundrc` 
file but none of that lead me anywhere. So I will skip right to what actually worked.

Since ALSA is beneth PulseAudio I figured I would want to change ALSA's default to be a more
global effect. But since I was getting nowhere I settled on changing PulseAudio's default sink.
This actually worked out just fine for me and only took these two steps:

1. Get the name of the sink from Pulse Audio:

```
$ pactl list | grep -iP '^Sink #\d' -A2 | tail -n 1 | cut -d':' -f 2
```

2. Then for the current session, pass that as the default to PA's runtime config daemon.

```
$ echo "set-default-sink alsa_output.usb-BEHRINGER_UMC202HD_192k-00-U192k.analog-stereo" | pacmd
```

I can see that the interface is now set as my default sink:

```
$ pactl stat
...
Default Sink: alsa_output.usb-BEHRINGER_UMC202HD_192k-00-U192k.analog-stereo
Default Source: alsa_input.pci-0000_00_1b.0.analog-stereo
...
```

(Looking at that output tells me I will need to do the same for the source if
 I ever hook a mic up to this interface...)

And that is it. I can now listen to my headphones without ground loop noise and
use the nice analog dial on the interface to control my volume. 

