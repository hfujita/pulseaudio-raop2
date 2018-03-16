# What is this?

This is an ongoing effort to make it possible to stream musics from [PulseAudio](http://www.freedesktop.org/wiki/Software/PulseAudio/) to recent AirPlay-capable devices (e.g. Apple TV, AV receivers).

Apparently there are two versions in AirPlay protocol. The older version (version 1) only uses TCP for audio streaming. Version 1 is supported by PulseAudio, in a form of raop module. (RAOP stands for Remote Audio Output Protocol, a protocol underneath AirPlay.) However the newer version 2, which uses UDP as well, is not supported by PulseAudio (as of v4.0). Probably most AirPlay devices today support version 2, which makes it impossible for PulseAudio users to play their music over AirPlay.

## Brief history

There have been a few efforts to address this issue. First Christophe Fergeau started a [project](https://github.com/zx2c4/pulseaudio-raop2) to support raop2. Later Martin Blanchard [continued to enhance](http://repo.or.cz/w/pulseaudio-raopUDP.git/shortlog/refs/heads/raop) Fergeau's work. My work is actually based on Blanchard's work, meaning that it is also based on Fergeau's efforts.

Here are the several differences from Blanchard's work.

* Fixed several bugs which prevented PulseAudio from playing music on Pioneer VSX-43.
    * I did not intentionally include any device-specific codes but it is the only AirPlay device I currently have.
* Upgraded the codebase to PulseAudio 4.0 (previously around 2.99).

Then recently Colin Leroy [rebased my & Martin's latest patches on PulseAudio 5.0](https://github.com/colinleroy/pulseaudio).

See the commit history for details.

# Current status (as of 5/27/2016)

* Plays back music on Pioneer VSX-43 (AV receiver), Apple TV 3rd gen. and some other devices (see below for tested device list).
* Supposed to support both TCP an UDP protocols. However, TCP procotol is not tested at all because I don't own a device.
* We are trying to get it merged into the mainstream but not done yet.

## Known issues

As of 10/14/2013

* Sound test (launched from the GNOME sound setting dialog) does not work (no sound heard).
* Works reasonably with VLC/Totem/Rhythmbox but when playing music from Flash player, quite rough sound is heard.
* There's a huge latency. Okay for music player but absolutely terrible for movies/voice chat.
* [Volume calculation is wrong](https://bugs.freedesktop.org/show_bug.cgi?id=42804#c43)

* For more details, see the [github issues page](https://github.com/hfujita/pulseaudio-raop2/issues).

# Warnings

* This is still not more than an experimental branch. It is totally unstable and may not work at all on some devices. **Do not expect too much!**
* This is totally my personal hobby. It is not supported by any company or organization. My contribution totally depends on my free time and motivation. Especially, my primary goal is to make it work on my Pioneer VSX-43, so I may not be interested in other devices.
* You don't have to be a C/systems programming expert to try it, but you probably have to be an expert of building software/configuring your system.
* As written in the license, there is **absolutely no warranty**. You should try this at your own risk.

# How to try

Since I'm using Ubuntu 16.04, I'll give you an instruction in Ubuntu. Should be similar in Debian or other Debian/Ubuntu-based distros. For other systems, please send me a patch for this page if you figure out how.

## 0. Prerequisites

Install some packages:

    sudo apt-get install build-essential paprefs git pulseaudio-module-raop intltool libavahi-client-dev
    sudo apt-get build-dep pulseaudio

(Thanks Stephan Henningsen for [pointing out intltool](https://github.com/hfujita/pulseaudio-raop2/issues/16) and Denilson Figueiredo de Sá for [pointing out libavahi-client-dev](https://github.com/hfujita/pulseaudio-raop2/issues/54)!)

## 1. Retrieving the code

    $ git clone https://github.com/hfujita/pulseaudio-raop2.git

or

    $ git clone git@github.com:hfujita/pulseaudio-raop2.git # if you have github account and registered your key

**Note:** tarball or zip **won't work** because PulseAudio build system relies on git revision information.

## 2. Building the software

As written in pulseaudio-raop2/README

    $ cd pulseaudio-raop2
    $ ./autogen.sh
    $ CFLAGS="-ggdb3 -O0" LDFLAGS="-ggdb3" ./configure  # See below
    $ make

FYI, here is my command line for `configure`, taking some parameters from the Ubuntu PulseAudio package.

    $ CFLAGS="-ggdb3 -O0" LDFLAGS="-ggdb3" ./configure --prefix=$HOME --enable-x11 --disable-hal-compat

## 3. Before running the new PulseAudio

### paprefs

PulseAudio should be configured so that it will discover AirPlay devices on the network.

    $ paprefs &  # This is an X application

In the `Network Access` tab, turn on `Make discoverable Apple AirTunes sound devices available locally`, then close the application.

### Firewall

If you are using personal firewall (e.g. ufw or iptables), it should allow UDP packets to come in and go out between the AirPlay device. I can hardly tell which port should be opened, as ports are dynamically chosen.

### Disabling autospawn

In your environment, it is likely that PulseAudio is already running. First you need to stop it. However, by default PulseAudio re-spawns automatically even if you stop it manually.

    $ echo "autospawn=no" >> ~/.pulse/client.conf

This will disable autorespawning (thanks to [this page](http://askubuntu.com/questions/8425/how-to-temporarily-disable-pulseaudio)). You may later remove this directive if you lose interest in PulseAudio experiments and want to go back to the original configuration.

Then, finally we are ready to kill the existing PulseAudio daemon.

    $ pulseaudio -k

Ubuntu desktop users will see that the sound icon on the status bar changes.

### Configuration file for local devices (for PA 5.0 and later)

If you are currently using PA 4.0 or older version, you need to copy a configuration file in order to use local devices (internal speakers, headphones, HDMI, etc) other than AirPlay devices.

    $ cp /usr/share/pulseaudio/alsa-mixer/profile-sets/extra-hdmi.conf \
       $PA_SRC_DIR/src/modules/alsa/mixer/profile-sets/

Replace `$PA_SRC_DIR` with your pulseaudio working directory -- a directory to which you cloned the tree.

(Thanks Colin Leroy for this tip!)

## 4. Launching the new PulseAudio

    $ ./src/pulseaudio -n -F src/default.pa -p $(pwd)/src/ --log-time=1 -vvvv 2>&1 | tee pulse.log

Actually, options after `-p $(pwd)/src/` are optional (logging purpose), but definitely helpful for troubleshooting.

In the sound setting dialog, you'll see your AirPlay device as an output device. Select your device and start your favorite music.

Have fun!

When you are done, press Ctrl+C to kill the program. If the program hangs up and doesn't accept Ctrl+C, do

    $ pkill -KILL pulseaudio

### (5. Relaunching the original PulseAudio daemon)

After you finish the experiment, you can launch the original PulseAudio daemon by

    $ pulseaudio -D

### 6. Updating the source tree

To get the latest patches from my repository, run:

    $ cd pulseaudio-raop2
    $ git pull

# Tested music players

## Works well

* VLC
* Totem
* Rhythmbox
* Amarok (thanks to Ewan Cochevelou)

## Rough sound

* Adobe Flash player

# Tested device

Please [send me a report](https://docs.google.com/forms/d/1L6nNGISIFCLSLCyCqao3W7qOMkBkhLTPdDDJ6LsNHyI/viewform?usp=send_form) if you have any experience with your device. Both positive/negative results are welcome.

* Pioneer VSX-43
* Apple TV 3rd generation
* Minx Air 200 ([thanks to Matthias](https://bugs.freedesktop.org/show_bug.cgi?id=42804#c30))
* shairport ([thanks to Karl Stavestrand](https://bugs.freedesktop.org/show_bug.cgi?id=42804#c29))
* Pioneer VSX-922 ([thanks to Matthias](https://bugs.freedesktop.org/show_bug.cgi?id=42804#c43))
* NOCS NS2 (Thanks to Kim Tore Jensen)
* Freebox Revolution Server (Thanks to Pierre Gaudillere)
* Sony CMT-G2BNIP
* Pioneer VSX-822 (Thanks to Martin Owens)
* Yamaha RX-A820 (Thanks to Patryk Zawadzki)
* JBL On Air (Thanks to Aaron Brady)
* Pioneer XW-SMA4 (thanks to Giuseppe Iannello)
* Sony SA-NS510 (thanks to Giuseppe Iannello)
* [iHome IW2](http://ihomeaudiointl.com/iW2BC/) (thanks to Nicolas VEYSSIERE)
* Pioneer X-HM71 (thanks to Patrice Mandin)
* Philips AD7050W/10 (thanks to Ralf Ertzinger)
* Sony STR-DN840 (thanks to Cyrus Boadway)
* PURE Contour 200i Air (thanks to Johannes Schinko)

# Contact information

Please [post a new issue at github](https://github.com/hfujita/pulseaudio-raop2/issues) (probably a github account is required). You can also e-mail me at Hajime Fujita <crisp.fujita@nifty.com> but for general questions/bug reports/feature request etc. github is **strongly encouraged** because the discussion is open to public, and I tend to lose track of e-mails.
Also, this topic (raop2 support) is discussed at [PulseAudio Bug 42804](https://bugs.freedesktop.org/show_bug.cgi?id=42804).

# Related information

## Other implementations

Some other raop2 implementations which allegedly work. (I've not actually tested though.)

* [node-airtunes](https://npmjs.org/package/airtunes) node.js implementation of raop2
* [forked-daapd](https://github.com/jasonmc/forked-daapd)

## Protocol information

* http://git.zx2c4.com/Airtunes2/about/
* http://nto.github.io/AirPlay.html