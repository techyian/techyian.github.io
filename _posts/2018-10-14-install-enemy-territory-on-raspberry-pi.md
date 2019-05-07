---
layout: post
image: /img/Enemy_territory_logo.jpg
title: Install Enemy Territory on a Raspberry Pi
category: gaming
description: Learn how install Enemy Territory Legacy on a Raspberry Pi.
tags: [gaming, raspberry-pi]
---

Ever since it was first released on PC, I have been a huge fan of Wolfenstein: Enemy Territory; it's broad community and excellent graphics for the era drew many players online to gain as much XP as possible, and get rewarded with the duel Akimbo pistols. Being a 15 year old game, the online community has taken a hit and unfortunately many servers are now kept alive with bots, however things could soon change.

I am also an avid user of the Raspberry Pi computer and recently decided to try to get this old game running on my Pi 3b+, a 1.4Ghz ARM powered computer with 1gb RAM and on-board VideoCore IV graphics. For your information, we are going to be installing Enemy Territory: Legacy, which is a community supported version of the original game which aims to fix bugs and add new features to the game.

 Let's get started.

## Install dependencies

The first thing you'll need to do is install some dependencies which will allow you to compile the game from source code. Open up a terminal on your Pi and type in the following:

```
sudo apt-get install build-essential libfreeimage-dev libopenal-dev libpango1.0-dev libsndfile-dev libudev-dev libasound2-dev libjpeg8-dev libwebp-dev automake libgl1-mesa-glx libjpeg62-turbo libogg0 libopenal1 libvorbis0a libvorbisfile3 zlib1g libraspberrypi0 libraspberrypi-bin libraspberrypi-dev libx11-dev libglew-dev libegl1-mesa-dev nasm autoconf git cmake zip gcc g++ libtool
```

**Note**: These are a list of dependencies I've tried to put together during the course of getting this to work - they may not all be required, but most certainly are.

Once these are installed, you now need to enable the experimental OpenGL driver. In the terminal type `sudo raspi-config`, go to the Advanced page and enable the OpenGL driver, this will require a restart.

Once restarted, you are now ready to download and compile SDL from source. Go to a terminal and type the following:

```
wget https://www.libsdl.org/release/SDL2-2.0.8.tar.gz
tar zxvf SDL2-2.0.8.tar.gz
cd SDL2-2.0.8 && mkdir build && cd build
../configure --host=armv7l-raspberry-linux-gnueabihf
make -j4
sudo make install
```

After completing all these commands, SDL should be installed and ready to use.

## Download and install ET:Legacy

We are now going to clone the Git repository for ET: Legacy. In your terminal, type `git clone https://github.com/etlegacy/etlegacy.git`. After this has finished, enter the new directory it has created called "etlegacy" and open the "easybuild.sh" script - I tend to use nano for editing files `sudo nano easybuild.sh`. Next, find the part of the file which sets the CMake variables for the RPI flag, you are looking for the following:

```
elif [ "$var" = "-RPI" ]; then
			einfo "Will enable Raspberry PI build ..."
			ARM=1
			CROSS_COMPILE32=0
			RENDERER_DYNAMIC=0
			FEATURE_RENDERER_GLES=0
			FEATURE_RENDERER2=0
			FEATURE_JANSSON=0
			BUNDLED_JANSSON=0
			# FIXME: ogg doesn't compile
			BUNDLED_OGG_VORBIS=0
			FEATURE_OGG_VORBIS=0
			# FIXME
			FEATURE_THEORA=0
			BUNDLED_THEORA=0
			# not required
			BUNDLED_GLEW=0
			# FIXME: needs -PIC
			FEATURE_FREETYPE=0
			BUNDLED_FREETYPE=0
			#FEATURE_DBMS=0
			#BUNDLED_SQLITE3=0
			FEATURE_LUASQL=1
```

You need to add the following to this list and also edit the Omnibot sections:

```
			BUNDLED_SDL=0
			BUNDLED_OPENAL=0
			FEATURE_OMNIBOT=1
			INSTALL_OMNIBOT=1
```

This tells the installer not to use the bundled SDL or OpenAL libraries as we have installed them previously. Save and close the file `CTRL+X` followed by "Y". Fire off the installer by entering `./easybuild.sh -RPI -j4`. If all went well during the compilation, the script should have now created a new directory called "etlegacy" in the `/home/pi/` directory, go to this directory and type `./etl` to run the executable.

Congratulations, you have now installed Enemy Territory on your Raspberry Pi. Currently with the experimental driver, I am able to get an average of 30fps on low/mid settings - I expect things will improve as the driver becomes more stable.

Now that you have got the game installed, you'll be eagar to get playing, however all of the servers currently available for the game will only support x86 based computers, with a few perhaps supporting the x64 based legacy mod. This introduces a problem as it's no fun playing on your own.

Introducing Omnibot for ARM!

## Installing Omnibot

This is a slightly more involved process, and is certaintly experimental. In order to install Omnibot, you first need to download and compile the boost library, you can do this as follows:

### Install boost and bjam

```
cd ~/Downloads
wget https://netcologne.dl.sourceforge.net/project/boost/boost/1.66.0/boost_1_66_0.tar.bz2
tar xvjf boost_1_66_0.tar.bz2
cd boost_1_66_0
sudo ./bootstrap.sh --prefix=/usr/local
sudo ./b2 install
sudo ./bjam install
```

### Download and configure omni-bot

If all was successful you should now have boost installed on your Raspberry Pi. Next we need to download Omnibot:

```
cd ~/Downloads
git clone https://github.com/jswigart/omni-bot.git -b stable
cd omni-bot
git submodule init
git submodule update
```

At this stage, please ensure that the MonkeyScript dependency (gmscriptex) has been downloaded into the `omni-bot/0.83/Omnibot/dependencies/` folder.

You now need to edit some source files in order for Omnibot to compile successfully - this is the experimental part and an issue has been raised with the Omnibot devs [here](https://github.com/jswigart/omni-bot/issues/6#issuecomment-427597474). If you read this ticket you can see the lines of code which require commenting out. Next we need to edit the omni-bot install script:

```
cd ~/Downloads/omni-bot/0.83/Omnibot/linux/
sudo nano buildbot.sh
```

Edit the file to match the below:

```
#!/bin/sh

export BOOST=/home/pi/Downloads/boost_1_66_0
export BOOST_BUILD_PATH=$BOOST/tools/build
export BOOST_LIB=/usr/local/lib
export BOOST_SUFFIX=

cd /home/pi/Downloads/omni-bot/0.83/Omnibot
$BOOST/bjam -a -q debug -d2
```

**Note**: I've made this file clean the build files each time it's ran.

### Jamfile update

I have also noticed that the game monkey directories referenced in the Jamfile located at `omni-bot/0.83/Omnibot` are incorrect. Open up the Jamfile, and change any reference to `dependencies/gamesrc_ex/...` to `dependencies/gmscriptex/gamesrc_ex`.

### Compile omni-bot

Now save the file and run the installer `./buildbot.sh` - if you need to make the file executable do the following prior to running `sudo chmod +x ./buildbot.sh`.

Hopefully this should now compile and install omni-bot for you. This will also result in a file being created that we need to copy.

### ETLegacy configuration

`sudo cp ~/Downloads/omni-bot/0.83/Omnibot/build/ET/gcc-gnu-6.3.0/debug/omnibot_et.so ~/.etlegacy/legacy/omni-bot/`. This will copy the arm library so it can be detected by Enemy Territory.

Finally, we need to instruct Enemy Territory Legacy to use omni-bot when it loads the executable:

`sudo nano ~/etlegacy/etmain/legacy.cfg`

Find the section of the file which has omni-bot related cvar settings, and uncomment out:

```
set omnibot_enable "1"
set omnibot_path "./legacy/omni-bot"
```

**If you want to bypass the compilation yourself, you can download the compiled binary from [here](https://techyian.github.io/downloads/omnibot_et.so).**

Congratulations. If all was successful at this point, you should be able to open Enemy Territory and start a local server. You can access the referee area to set how many bots you want to enable.




