---
layout: post
image: /img/raspberrypi.png
title: Raspberry Pi 4 OpenGLES 1.1 support
category: gaming
description: How to setup OpenGLES 1.1 on the Raspberry Pi 4
tags: [gaming, raspberry-pi]
---

On Monday 24th June 2019, the Raspberry Pi 4 was announced. This new release of the popular single board computer features a significant upgrade from the previous model 3B+ and is available in 1GB, 2GB and 4GB variants. I bought a 2GB model with the goal of getting Enemy Territory: Legacy working with both OpenGL and GLES, with high hopes of even better performance.

The Pi 4 features an upgraded VideoCore VI GPU clocked at 500Mhz by default and comes with GLES 1.1, 2.0 and 3.0 support and in addition, X11 is now the window manager used by default using the Fake KMS kernel driver.

Trying my already tested Pi 3B+ setup for ET: Legacy using full OpenGL works exactly the same on the Pi 4; the game works well in X11, and there is still an issue running the game in fullscreen mode. GLES support however does not work and this is mainly due to the fact that the `libbrcmGLESv2` library should no longer be linked to when compiling your GLES programs as that is for the legacy dispmanx driver. Instead, the `GLESv1_CM` Mesa library found within `/usr/lib/` should be linked to. I also came across another problem too - `libgles1-mesa-dev` is **not** available in Raspbian Buster, Debian appear to have dropped support for the GLES 1.1 headers, this means that you need to use the firmware ones found within `/opt/vc/include`. I've been told that this isn't technically the correct way of doing things, but as it currently stands, I'm not aware of any other way of getting the correct headers without a custom build of Mesa.

Regarding SDL2, it has long been discussed that in order to enable GLES support, you need to pass a number of flags to the configure command. As the Pi 4 now uses X11 as its window manager, this is no longer the case, simply call `./configure` and build/install as normal and you should be good to go!

Hope this helps someone.