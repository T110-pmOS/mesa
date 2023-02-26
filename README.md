Build
==================
For goyawifi pmOS:

Initial Python 3 support. Now you should be able to build inside latest pmOS / Alpine.

build https://github.com/T110-pmOS/etna_viv e.g.:

    apk add make mesa-dev gcc alpine-sdk linux-headers libpng-dev git
    cd /root
    git clone https://github.com/T110-pmOS/etna_viv
    cd etna_viv
    export GCABI="goya"
    cd attic
    make -j2
    cd egl
    make -j2
    
Note: You need to build enta_viv first to get libetnaviv.a

Build mesa (this):

    cd /root
    git clone https://github.com/T110-pmOS/mesa -b py3_pre_rebase_2014_09 --depth 1
    apk add automake autoconf libtool bison libpthread-stubs eudev-dev glproto flex libdrm-dev sysfsutils sysfsutils-dev expat-dev libdrm-dev
    apk add python3 py3-libxml2 gettext
    cd mesa
    autoreconf -v --install
    apk add nano
    nano build.sh
    
Example of build.sh:

    #!/bin/bash
    ETNAVIV_BASE="/root/etna_viv"
    ETNAVIV_LIB="${ETNAVIV_BASE}/attic/etnaviv/" # important!
    ETNAVIV_INC="${ETNAVIV_BASE}/attic/" # important!
    
    export CFLAGS="-I${ETNAVIV_INC} -Wno-error=implicit-function-declaration"
    export CXXFLAGS="-I${ETNAVIV_INC}"
    export LDFLAGS="-L${ETNAVIV_LIB}"
    export LIBDRM_LIBS="-ldrm"
    
    #export ETNA_LIBS="-letnaviv" # important!
    export ETNA_LIBS="/root/etna_viv/attic/etnaviv/libetnaviv.a"
    export LIBTOOL_FOR_BUILD="/usr/bin/libtool" # important
    
    ./configure --enable-gles2 --enable-gles1 --disable-glx --enable-egl --enable-dri \
        --with-gallium-drivers=swrast,etna --with-egl-platforms=fbdev \
        --enable-gallium-egl --enable-debug --with-dri-drivers=

    make -j2

Then use build.sh to build:

    sh build.sh

Now you can use [mesatest](https://github.com/laanwj/mesatest_gles) to test etna mesa driver on Goyawifi.


Etnaviv Mesa fork
=================

Open source GLES1/2 driver for Vivante GPU hardware. It is capable of running most of the glmark tests
and some games.
This driver has been used to run glquake and d2x and other GLES games so it should be fairly stable.
There may still be quite a few rendering bugs, specific bug reports are very welcome.

This is being maintained as a set of patches on top of the Mesa main repository. Expect frequent rebases
while in this phase of development.

<details>
  <summary>Cubox build instructions</summary>
Build instructions
-------------------

To be written.

My configure script for cubox:
```bash
#!/bin/bash
DIR=... # path to target headers and libraries
ETNAVIV_BASE="${HOME}/projects/etna_viv"
ETNAVIV_LIB="${ETNAVIV_BASE}/src/etnaviv" # important!
ETNAVIV_INC="${ETNAVIV_BASE}/src" # important!

export TARGET="arm-linux-gnueabihf"
export CFLAGS="-I${DIR}/cubox/include -I${ETNAVIV_INC}"
export CXXFLAGS="-I${DIR}/cubox/include -I${ETNAVIV_INC}"
export LDFLAGS="-L${DIR}/cubox/lib -L${ETNAVIV_LIB}"
export LIBDRM_LIBS="-L${DIR}/cubox/lib -ldrm"

export ETNA_LIBS="-letnaviv" # important!
export LIBTOOL_FOR_BUILD="/usr/bin/libtool" # important!

./configure --target=${TARGET} --host=${TARGET} \
    --enable-gles2 --enable-gles1 --disable-glx --enable-egl --enable-dri \
    --with-gallium-drivers=swrast,etna --with-egl-platforms=fbdev \
    --enable-gallium-egl --enable-debug --with-dri-drivers=
```

- The etna gallium driver uses `libetnaviv.a` and its headers from the
  `etna_viv` project (https://github.com/laanwj/etna_viv) for access to the kernel driver and register descriptions.
  *You only need to build libetnaviv by running `make` in `native/etnaviv`*. The rest is part of the test
  and reverse engineering framework, and not needed for the driver.

```bash
export GCABI=v2/v4/dove/imx6/...
# rest of cross-compile target settings
cd native/etnaviv
make
```

Mesa cross compiling
---------------------
- libexpat and libdrm need to be available on the target (neither is used at the moment, but they are
dependencies for Mesa).
In many cases these can be copied from the device, after installing the appropriate development package.
</details>

Setup
===================

I use this script to set up the framebuffer console for (double or single buffered) rendering,
as well as prevent blanking and avoid screen corruption by hiding the cursor.

    #!/bin/bash
    # Set to usable resolution (double buffered)
    fbset 1280x1024-60 -vyres 2048 -depth 32
    # Set to usable resolution (single buffered)
    #fbset 1280x1024-60 -vyres 1024 -depth 32

    # Disable automatic blanking
    echo -e '\033[9;0]' > /dev/tty1
    echo 0 > /sys/class/graphics/fb0/blank

    # Disable blinking cursor
    echo -e '\033[?17;0;0c' > /dev/tty1

Switching between Etna en Swrast
--------------------------------
Frequently it is useful to compare the rendering from etna to the software rasterizer;
this can be done with the environment variable `EGL_FBDEV_DRIVER`, i.e.

    # Run with etna driver
    export EGL_FBDEV_DRIVER=etna
    (run EGL demo...)

    # Run with software rasterizer
    export EGL_FBDEV_DRIVER=swrast
    (run EGL demo...)

Force single buffering
-----------------------

To force single buffering (without wait for vsync) use the following:

    export EGL_FBDEV_BUFFERS=1

This can be useful for testing or benchmarking.

Testing
====================

This section lists some tests and demos that can be used to exercise the driver.

Mesatest
-------------
A few testcases that I made especially for this driver, based on the samples from the OpenGL ES 2.0 programming
guide (http://www.opengles-book.com/) can be found here:

https://github.com/laanwj/mesatest_gles

Glmark2
--------------
Some of the Glmark2 demos already run with this driver, but this is a work in progress.

Need a special glmark2 with fbdev support, which can be got here:

https://code.launchpad.net/~laanwj/glmark2/fbdev

Fetch:

    bzr branch lp:~laanwj/glmark2/fbdev

Build:

    ./waf configure --with-flavors=fbdev-glesv2 --data-path=${PWD}/data
    ./waf

Run:

    cd build/src
    ./glmark2-es2 -b shading -s 1280x1024 --visual-config alpha=0

Support matrix (cubox, v2, gc600):

    OK:
    [Scene] build    -> renders
    [Scene] shading  -> renders
    [Scene] texture  -> renders
    [Scene] effect2d -> renders
    [Scene] bump     -> renders
    [Scene] desktop  -> renders
    [Scene] clear    -> shows nothing (is not supposed to, either)
    [Scene] pulsar   -> renders
    [Scene] conditionals -> renders
    [Scene] function -> renders
    [Scene] jellyfish -> renders on GPUs with SQRT_TRIG

    Corrupted:
    [Scene] shadow   -> rendering corrupted
    [Scene] buffer   -> renders, but lines are not right
    [Scene] ideas    -> doesn't show much

    Shader assertion:
    [Scene] loop      -> missing loops support

    Crash:
    [Scene] refract   -> memory full
    [Scene] terrain   -> memory full / shader too long

Mesa demos
-------------

Mesa also comes with a few OpenGL ES 1 and 2 demos. These can be found in the following repository:

    git://anongit.freedesktop.org/mesa/demos

All the demos in `src/egl/opengles1` and `src/egl/opengles2` with fbdev and screen backend work.

OpenGL ES 1:

- eglfbdev
- drawtex_screen
- gears_screen
- torus_screen
- tri_screen

OpenGL ES 2:

- es2gears_screen
