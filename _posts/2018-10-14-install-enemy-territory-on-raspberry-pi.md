---
layout: post
image: /img/Enemy_territory_logo.jpg
title: Install Enemy Territory on a Raspberry Pi
category: gaming
description: Learn how install Enemy Territory Legacy on a Raspberry Pi.
tags: [gaming, raspberry-pi]
---

**Updated March 2020:** Updated dependency list, use latest boost library, remove bjam references, prefer bundled SDL, add Pi 4 instructions.

Ever since it was first released on PC, I have been a huge fan of Wolfenstein: Enemy Territory; it's broad community and excellent graphics for the era drew many players online to gain as much XP as possible, and get rewarded with the duel Akimbo pistols. Being a 15 year old game, the online community has taken a hit and unfortunately many servers are now kept alive with bots, however things could soon change.

I am also an avid user of the Raspberry Pi computer and recently decided to try to get this old game running on my Pi 3B+ (and Pi 4), a 1.4Ghz ARM powered computer with 1gb RAM and on-board VideoCore IV graphics. For your information, we are going to be installing Enemy Territory: Legacy, which is a community supported version of the original game which aims to fix bugs and add new features to the game.

Let's get started.

## Install dependencies

The first thing you'll need to do is install some dependencies which will allow you to compile the game from source code. Open up a terminal on your Pi and type in the following:

```
sudo apt-get install build-essential libfreeimage-dev libopenal-dev libpango1.0-dev libsndfile-dev libudev-dev \
libasound2-dev libjpeg8-dev libwebp-dev automake libgl1-mesa-glx libjpeg62-turbo libogg0 libopenal1 libvorbis0a \
libvorbisfile3 zlib1g libraspberrypi0 libraspberrypi-bin libraspberrypi-dev libx11-dev libglew-dev libegl1-mesa-dev \
nasm autoconf git cmake zip gcc g++ libtool libxrandr-dev x11proto-randr-dev
```

**Note**: These are a list of dependencies I've tried to put together during the course of getting this to work - they may not all be required, but most certainly are.

Enemy Territory: Legacy supports both OpenGL and OpenGL ES and I have personally tested it on models 1B, 3B+ and 4B. 

It is recommended to use the bundled SDL binaries (this is the default) as they make the install process much easier.


## Download and install ET:Legacy

We are now going to clone the Git repository for ET: Legacy. In your terminal, type `git clone https://github.com/etlegacy/etlegacy.git`. After this has finished, enter the new directory it has created called "etlegacy" and open the "easybuild.sh" script - I tend to use nano for editing files `sudo nano easybuild.sh`. Next, find the part of the file which sets the CMake variables for the RPI flag, you are looking for the following:

```
elif [ "$var" = "-RPI" ]; then
```

To enable Omnibot ensure the following two lines are added to the RPI section:

```
FEATURE_OMNIBOT=1
INSTALL_OMNIBOT=1
```

**Raspberry Pi 3B+ (and earlier) notes**: The RPi 3B+ has 3 different graphics drivers: Legacy (GLES only), GL - Fake KMS and GL - Full KMS. These can be chosen using the `raspi-config` application. Depending on how you want to build ET will depend on which graphics driver you choose. If building for OpenGL ES you will need the Legacy graphics driver and ET will only run from a non-X11 environment so you'll need to exit to the terminal in order to run the game; full OpenGL will run from an X11 environment and you'll need either the Fake or Full KMS driver.

**Raspberry Pi 4B notes**: The OpenGL driver used is the Fake KMS driver and currently both OpenGL and GLES are ran within an X11 session.

Within the easybuild script you will see a line called `FEATURE_RENDERER_GLES`; if you want to use OpenGL ES this needs to be set to 1 (default), otherwise set it to 0. Close and Save the file `CTRL+X` followed by "Y". Fire off the installer by entering `./easybuild.sh -RPI -j4`. If all went well during the compilation, the script should have now created a new directory called "etlegacy" in the `/home/pi/` directory. Next you will need to copy over the official pak files (`pak0.pk3, pak1.pk3 and pak2.pk3`) into the `/home/pi/etlegacy/etmain` directory. Once you've done that go to the root of the `etlegacy` directory and type `./etl` to run the executable.

Congratulations, you have now installed Enemy Territory on your Raspberry Pi. You typically want to be running the game at around 1024x768 and turn the graphics settings down to achieve playable framerates. I have found the best performance (not surprisingly) to be on the Pi 4 using the GLES renderer.

Now that you have got the game installed, you'll be eagar to get playing, however all of the servers currently available for the game will only support x86 based computers, with a few perhaps supporting the x64 based legacy mod. This introduces a problem as it's no fun playing on your own.

Introducing Omnibot for ARM!

## Installing Omnibot

This is a slightly more involved process and is certaintly experimental. The official installation instructions for Omnibot can be found [here](http://omni-bot.invisionzone.com/wiki/index.php?title=Compile), however I've found there to be a few additional steps you need to take to successfully install for ARM.

### Edit `buildbot.sh`

Within the `omni-bot/0.83/Omnibot/linux` directory you will see a file called `buildbot.sh`, this is the file we need to run to compile the project. Open this file with nano and edit to match the below:

```
#!/bin/sh

export BOOST=/home/pi/Downloads/boost_1_72_0
export BOOST_BUILD_PATH=$BOOST/tools/build
export BOOST_LIB=/usr/local/lib
export BOOST_SUFFIX=

cd /home/pi/Downloads/omni-bot/0.83/Omnibot
$BOOST/b2 architecture=arm -a -q debug
```

**Note**: I've made this file clean the build files each time it's ran, there are other build scripts which clean the project specifically but I wanted one script to do everything.

### Jamfile update

I have also noticed that the game monkey directories referenced in the following Jamfiles are incorrect: `omni-bot/0.83/Omnibot/Jamfile`, `omni-bot/0.83/Omnibot/Common/Jamfile`. Change any reference to `dependencies/gmsrc_ex/...` to `dependencies/gmscriptex/gmsrc_ex`.

### Compile omni-bot

Now save the file and run the installer `./buildbot.sh` - if you need to make the file executable do the following prior to running `sudo chmod +x ./buildbot.sh`.

Hopefully this should now compile and install omni-bot for you. This will also result in a file being created that we need to copy.

### ETLegacy configuration

`sudo cp ~/Downloads/omni-bot/0.83/Omnibot/build/ET/gcc-gnu-8/debug/omnibot_et.so ~/.etlegacy/legacy/omni-bot/`. This will copy the arm library so it can be detected by Enemy Territory.

Finally, we need to instruct Enemy Territory Legacy to use omni-bot when it loads the executable:

`sudo nano ~/etlegacy/etmain/legacy.cfg`

Find the section of the file which has omni-bot related cvar settings, and uncomment out:

```
set omnibot_enable "1"
set omnibot_path "./legacy/omni-bot"
```

**If you want to bypass the compilation yourself, you can download the compiled binary from [here](https://techyian.github.io/downloads/omnibot_et.so).**

Congratulations. If all was successful at this point, you should be able to open Enemy Territory and start a local server. You can access the referee area to set how many bots you want to enable.




