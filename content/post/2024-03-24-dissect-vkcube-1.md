---
title: "vulkan: Dissecting vkcube: Build"
date: '2024-03-24'
categories:
  - Programming
tags:
  - Vulkan
---

This post lists the steps to build debug versions of mesa, libdrm and
vkcube.

---

``` sh
    # Build debug versions of mesa, libdrm, and vulkan-tools
    # build-mesa-vt-dbg.sh
    # Run as: sh build-mesa-vt-dbg.sh
    
    set -e
    set -u
    
    DOWNLOAD=false
    SRC=/tmp/src
    BLD=/tmp/bld
    INSTALL=$HOME/tools/vt-dbg
    
    mkdir -p $SRC
    if [ "$DOWNLOAD" == "true" ]; then
        cd $SRC
        git clone --depth 1 https://github.com/KhronosGroup/Vulkan-Headers.git vh
        git clone --depth 1 https://github.com/KhronosGroup/Vulkan-Tools.git vt
        git clone --depth 1 https://github.com/zeux/volk.git volk
        git clone --depth 1 https://github.com/CLRX/CLRX-mirror.git clrx
        git clone --depth 1 https://gitlab.freedesktop.org/wayland/wayland-protocols.git wp
        git clone --depth 1 https://gitlab.freedesktop.org/mesa/drm.git drm
        git clone --depth 1 https://gitlab.freedesktop.org/mesa/mesa.git mesa
    fi
    
    # Some other dependencies assumed to be already installed:
    # wayland-scanner, wayland-client, meson, cmake, pkgconf, python-mako,
    # llvm, llvm-libs
    
    mkdir -p $BLD/vh
    mkdir -p $BLD/vt
    mkdir -p $BLD/wp
    mkdir -p $BLD/clrx
    mkdir -p $BLD/volk
    mkdir -p $BLD/drm
    mkdir -p $BLD/mesa
    
    # clrx disassembler seems to be required only for older AMD GPUs,
    # GFX6-GFX7 generation. Disabled for now.
    # cd $BLD/clrx
    # cmake -DCMAKE_INSTALL_PREFIX=$INSTALL/clrx -DCMAKE_BUILD_TYPE=Release $SRC/clrx
    # make install -j4
    
    # Vulkan-Headers
    cd $BLD/vh
    cmake -DCMAKE_INSTALL_PREFIX=$INSTALL/vh $SRC/vh
    make install -j4
    
    # volk
    cd $BLD/volk
    cmake -DCMAKE_INSTALL_PREFIX=$INSTALL/volk -DVOLK_HEADERS_ONLY=ON \
    -DVOLK_INSTALL=ON -DVULKAN_HEADERS_INSTALL_DIR=$INSTALL/vh $SRC/volk
    make install -j4
    
    # wayland-protocols
    cd $SRC/wp
    meson setup --reconfigure $BLD/wp -Dprefix=$INSTALL/wp -Dtests=false
    ninja -C $BLD/wp install -j4
    
    # Vulkan-Tools (only cube on wayland)
    cd $BLD/vt
    PKG_CONFIG_PATH=$INSTALL/wp/share/pkgconfig \
    cmake -DCMAKE_INSTALL_PREFIX=$INSTALL/vt \
    -DCMAKE_PREFIX_PATH=$INSTALL/volk -DBUILD_ICD=OFF -DBUILD_CUBE=ON \
    -DBUILD_VULKANINFO=OFF -DVULKAN_HEADERS_INSTALL_DIR=$INSTALL/vh \
    -DBUILD_WSI_XCB_SUPPORT=OFF -DBUILD_WSI_XLIB_SUPPORT=OFF \
    -DBUILD_WSI_WAYLAND_SUPPORT=ON -DCUBE_WSI_SELECTION=WAYLAND \
    -DCMAKE_BUILD_TYPE=Debug $SRC/vt
    make install -j4
    
    # libdrm (only vulkan for AMD)
    cd $SRC/drm
    meson setup --reconfigure $BLD/drm -Dprefix=$INSTALL/drm -Dbuildtype=debug \
    -Dradeon=disabled -Dintel=disabled -Damdgpu=enabled -Dnouveau=disabled \
    -Dvmwgfx=disabled -Dfreedreno=disabled -Dvc4=disabled -Detnaviv=disabled \
    -Dcairo-tests=disabled -Dtests=false -Dman-pages=disabled -Dvalgrind=disabled
    ninja -C $BLD/drm install -j4
    
    # mesa (only vulkan for AMD + LLVM)
    cd $SRC/mesa
    PKG_CONFIG_PATH=$INSTALL/wp/share/pkgconfig \
    meson setup --reconfigure $BLD/mesa -Dprefix=$INSTALL/mesa \
    -Dbuildtype=debug -Dgallium-drivers= -Dvulkan-drivers=amd -Degl=disabled \
    -Dplatforms=wayland -Dtools= -Dllvm=enabled -Dvideo-codecs=
    ninja -C $BLD/mesa install -j4
    
    # rm -rf /tmp/src
    # rm -rf /tmp/bld
    
    # To run vkcube, with various debug-logs enabled.
    
    # INSTALL=$HOME/tools/vt-dbg;\
    # PATH=$PATH:$INSTALL/clrx/bin \
    # LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$INSTALL/clrx/lib64 \
    # LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$INSTALL/drm/lib \
    # LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$INSTALL/mesa/lib \
    # RADEON_ICD_FILENAME=radeon.icd.x86_64.json \
    # VK_ICD_FILENAMES=$INSTALL/mesa/share/vulkan/icd.d/$RADEON_ICD_FILE_NAME \
    # MESA_SHADER_CACHE_DISABLE=true \
    # NIR_DEBUG=print,print_vs,print_fs,print_cs,print_internal,print_pass_flags \
    # RADV_DEBUG=epilogs,img,info,llvm,metashaders,nomemorycache,preoptir,prologs,\
    # shaders,shaderstats,spirv,startup \
    # $INSTALL/vt/bin/vkcube-wayland --c 1 &> /tmp/out.txt
```
---
