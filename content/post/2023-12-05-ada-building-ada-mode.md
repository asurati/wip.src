---
title: "ada: Building ada-mode for Emacs"
date: '2023-12-05'
categories:
  - Tools
tags:
  - Ada
  - Emacs
---

This post lists steps that build
[`ada-mode`](https://elpa.gnu.org/packages/ada-mode.html) for Emacs. The build
depends on the GNAT FSF compiler being available on the build system. On
Arch Linux, the compiler is available as the
[`gcc-ada`](https://wiki.archlinux.org/title/Ada) package.

---
``` bash
    # build-ada-mode.sh
    # Run as: sh build-ada-mode.sh

    set -e
    set -u

    DOWNLOAD=true
    SRC=/tmp/src
    TOOLS=$HOME/tools/ada
    LR1_SRC=ada_annex_p_lr1_parse_table.txt-8.1.0
    LR1_TAB=ada_annex_p_lr1_re2c_parse_table.txt
    LR1_URL=https://download.savannah.nongnu.org/releases/ada-mode/$LR1_SRC
    SAVE=$PATH

    mkdir -p $SRC
    if [ "$DOWNLOAD" == "true" ]; then
        cd $SRC
        git clone --depth 1 https://github.com/AdaCore/gprbuild.git
        git clone --depth 1 https://github.com/AdaCore/xmlada.git
        git clone --depth 1 https://github.com/AdaCore/gprconfig_kb.git
        git clone --depth 1 https://github.com/AdaCore/gnatcoll-core.git
        git clone --depth 1 --single-branch --branch externals/wisi \
            https://git.savannah.gnu.org/git/emacs/elpa.git wisi
        git clone --depth 1 --single-branch --branch externals/ada-mode \
            https://git.savannah.gnu.org/git/emacs/elpa.git ada-mode
        sed -i '/wisitoken-bnf-generate/d' $SRC/ada-mode/build.sh
        wget --show-progress -O $LR1_TAB $LR1_URL
    fi

    # Bootstrap GPRbuild
    cd $SRC/gprbuild
    ./bootstrap.sh --with-xmlada=../xmlada --with-kb=../gprconfig_kb \
    --prefix=$TOOLS/bgprb
    export PATH=$SAVE:$TOOLS/bgprb/bin

    # XML/Ada
    cd $SRC/xmlada
    ./configure --prefix=$TOOLS/xmlada --enable-shared
    make
    make install
    export GPR_PROJECT_PATH=$TOOLS/xmlada/share/gpr

    # GPRbuild
    cd $SRC/gprbuild
    make prefix=$TOOLS/gprb setup
    make LIBRARY_TYPE=relocatable all
    make libgpr.build
    make install
    make libgpr.install
    export PATH=$SAVE:$TOOLS/gprb/bin
    export GPR_PROJECT_PATH=$GPR_PROJECT_PATH:$TOOLS/gprb/share/gpr

    # GNATcoll-Core
    cd $SRC/gnatcoll-core
    make prefix=$TOOLS/gc setup
    make
    make install
    export GPR_PROJECT_PATH=$GPR_PROJECT_PATH:$TOOLS/gc/share/gpr

    # ada-mode
    cd $SRC/ada-mode
    export WISI_DIR=../wisi
    ./build.sh
    ./install.sh $TOOLS/am

    # lr1 table
    mv $SRC/$LR1_TAB $TOOLS/am/bin/
```
---
### Emacs configuration:
Modify the Emacs init file, to append `exec-path` with the path to the
`ada-mode` executables.

``` lisp
    (setq exec-path (append exec-path '(expand-file-name "~/tools/ada/am/bin")))
```

When launching Emacs, ensure that `$TOOLS/gprb/bin` is added to `$PATH` so that
GPRbuild binaries are available. When trying to compile an `.adb` file
through the `C-c C-c` command, Emacs attempts to launch `gprbuild`.

---
