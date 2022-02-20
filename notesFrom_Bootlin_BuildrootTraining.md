R: 347pages slides: https://bootlin.com/doc/training/buildroot/buildroot-slides.pdf
R: other training materials: https://bootlin.com/docs/

## p14: Simplified Linux System Architecture
- Userspace:
  - Application, Library, C Library
- Linux kernel:
  - Task/memory management, Device drivers, networking, filesystems
- Bootloader
- Hardware

## p15: Overall Linux boot sequence
- Bootloader: loads DTB. kernel to RAM, starts kernel
- Kernel: Inits hardware devices & kernel subsys, mount rootfs, starts /sbin/init
- Rootfs:
  - /sbin/init: start other userspace services & apps
    - shell
    - other apps

## p16: Embedded Linux work:
- BSP: porting bootloader & linux kernel, dev Linux device drivers
- sys integration: assembling userspace components, configure, dev upgrade & recov, ..
- app dev: writ company-specific apps & libs

## p17: Embedded Linux build system: principle
- configuration & open-source/in-house components
  - embedded linux build system
    - rootfs image
    - kernel image
    - bootloader image(s)
    - toolchain
# INTRO
## p27: Configuring Buildroot
- like Linux kernel, uses Kconfig:
`make menuconfig`

## p29: Running the build
- simplest way:
`make`
- keeping logs of the build output for later analysis:
`make 2>&1 | tee build.log`

## p30: Buildr results
- build results located in `output/images`:
  - one/several rootfs images
  - one kernel image & possibly one/several DT blobs
  - one/several bootloader images
- no standard way to install those on any given device:
  - very deivce specific
  - tools to generate SD card / USB key images ( `genimages`) or to flash/boot

# MANAGING BUILD AND CONFIG

## p33: Default build organization
- all build output go into `output/`, within top-level BR source dir
    - `O = output`
- config file stored as `.config`in top-level BR source dir:
  - `CONFIG_DIR = $(TOPDIR)`
  - `TOPDIR = $(shell pwd)`
- `buildroot/`:
    - `.config`
    - `arch/`
    - `package/`
    - `output/`
    - `fs/`
    - `...`

## p34: Out of tree build: intro
- allows using an output directory different than `output/`
- useful to build different BR configs from same source tree
- setting output dir done by passing `O=/path/to/dir`on cli
- config file stored inside the `$(O)`dir, as opposed to inside BR source for in-tree build
- `project/`:
  - `buildroot/`: BR sources
  - `foo-output/`: output of 1st project
    - `.config`
  - `bar-output/`: output of 2nd project
    - `.config`

## p35: Out of tree build: using
- to start out-of-tree build, 2 ways:
  - from BR source tree, specifying `O=`var:
  `make O=../foo-output/ menuconfig`
  - from an empty output dir, specifying `O=` & path to BR source tree:
  `make -C ../buildroot/ O=$(pwd) menuconfig`
- once one out-of-tree op has been done ( menuconfig, loading a defconfig, .. ),
  BR creates a small wrapper `Makefile`in the output dir
- this wrapper `Makefile` then avoids the need to pass `O=` & path to the BR source tree

## p36: Out of tree build: example
- 1 in BR source tree: `ls # arch board boot .. Makefile .. package ..`
- 2 create new output dir & move to it:
  ```
  mkdir ../foobar-output
  cd ../foobar-output
  ```
- 3 start new BR config: `make -C ../buildroot/ O=$(pwd) menuconfig`
- 4 start the build ( non need to pass `0=`and `-C` thx to wrapper ): `make`
- 5 adjust config again, restart build, clean build:
  ```
  make menuconfig
  make
  make clean
  ```
  
## p37: Full config file vs `defconfig`:
- `.config` file is a FULL config file: it holds value for all options ( xcept unmet deps )
- default `.config` has >4467 lines ( BR 2021.02 ): not practical for human read/mod
- a DEFCONFIG stores only values for opts for which non-default value is chosen:
  - easier to read
  - human moddable
  - usable for auto constructions of configs

## p38: defconfig: example
- for default BR config, defconfig is empty: everything is the default
- changing th arch to be ARM, the defconfig is just one line: `BR2_arm=y`
- id also enabling the `stress` package, the defconfig 'll be just two lines:
  ```
  BR2_arm=y
  BR2_PACKAGE_STRESS=y
  ```
  
## p39: Using and creating a `defconfig`:
- to use one, copying it to `.config`is not enough as all missing/def opts must expand
- BR allows to load defconfig stored in `configs/`dir by doing: `make <foo>_defconfig``
  - it overwrites the current `.config` if any
- to create a defconfig: `make savedefconfig`
  - saved in file pointed by `BR2_DEFCONFIG`config opt
  - by def, points to `defconfig`in current dir if config was from scratch or original
    defconfig if config was loaded from a defconfig
  - move it to `configs/`to make it easily loadable with `make <foo>_defconfig`

## p40: Existing defconfigs:
- BR comes with existing defconfigs for hardware platforms: RPi, BB, QEMU, ..
- list them using `make list-defconfigs`
- most buil-tin defconfigs are minimal: only build toolchain, bootloader, kernel & rootfs
  ```
  make qemu_arm_vexpress_defconfig
  make
  ```
- additional instructions often available in `board/<boardname>/readme.txt`
- custom defconfigs can obviously be more featureful

## p41: Assembling a defconfig:
- defconfigs are trivial text files & we can use concatenation to assemble them from
  fragments:
  - `platform1.frag`:
    ```
    BR2_arm=y
    ..
    ```
  - `platform2.frag`:
    ```
    BR2_mipsel=y
    ..
    ```
  - `packages.frag`:
    ```
    BR2_PACKAGE_STRESS=y
    ..
    ```
  - `debug.frag`:
    ```
    BR2_ENABLE_DEBUG=y
    ..
    ```
- Building a release system for platform1:
  ```
  ./support/kconfig/merge_config.sh platform1.frag packages.frag
  make
  ```
- Building a debug system for platform2:
  ```
  ./support/kconfig/merge_config.sh platform2.frag packages.frag debug.frag
  make
  ```
- `olddefconfig`expands a minimal defconfig to a full `.config`
- saving fragments is not possible: it must be done manually from an existing defconfig

## p43: Other building tips:
- cleaning targets:
  - cleaning all build output, but keeping the config file: `make clean`
  - cleaning everything, including config file & DL file if at def location: `make distclean`
- verbose build:
  - by def, BR hides cmds ran during build & show most important, for more: `make V=1`
    ( applies to packages, like Linux kernel, busybox, .. )
    
# BUILDROOT SOURCE AND BUILD TREES

## p46: Source tree:
- `Makefile`: top-level one handles config & orchestrates build
- `Config.in`: top-level one is main opts, includes other `Config.in` files
- `arch/`:
  - `Config.in.*` files defining the arch variants
  - `Config.in, Config.in.arm, Config.x86, ..`
- `toolchain/`:
  - packagesfor gene/using toolchains
  - `toolchain/` virtual package depends on either `toolchain-buildroot` or `toolchain-external`
  - former build internal one, latter DL/import xternal one
- `system/`
  - `skeleton/`: rootfs skeleton
  - `Config.in`: opts for sys-wide feats like init sys, `/dev` handling, ..
- `linux/`:
  - `linux.mk`: Linux kernel package
- `package/`:
  - all user space package ( 2800+ )
  - `busybox/, gcc/, qt5/, ..`
  - `pkg-generic.mk`: core package infrastructure
  - `pkg-cmake.mk, pkg-autotools.mk, pkg-perl.mk, ..`: specialized package infrastructures
- `fs/`:
  - logic to generate fs images in various formats
  - `common.mk`: common logic
  - `cpio/, ext2/, squashfs/, tar/, ubifs/, ..`
- `boot/`:
  - bootloader packages
  - `barebox/, grub2/, syslinux/, uboot/, ..`
- `configs/`:
  - def config files for various platforms
  - similar to kernel defconfigs
  - `raspberrypi_defconfig, ..`
- `board/`:
  - board-specific files ( kernel config files, kernel patches, images flashing scripts, .. )
  - typically go togther with a defconfig in `configs/`
- `support/`:
  - misc utilities ( kconfig code, libtool patches, DL helpers, .. )
- `utils/`:
  - various utilities useful to BR devs
  - `brmake`: make wrapper with logging
  - `get-developers`: to know to whom patches should be sent
  - `test-pkg`: to validate a package builds properly
  - `scanpipy, scancpan`: to generate Python/Perl package `.mk` files
  - ..
- `docs/`:
  - BR doc
  - ~ 135 pages PDF
  - https://buildroot.org/downloads/manual/manual.html

# BUILD TREE

## p52: Build tree $(O):
- `output/`
- global output dir
- can be customized for out-of-tree build by passing `O=<dir>`
- variable `O`( as passed on the cli )
- variable `BASE_DIR` ( as abs path )

## p53: Build tree $(O)/build:
- `output/`
  - `build/`
    - `buildroot-config/`
    - `busybox-1.22.1/`
    - `host-pkgconf-0.8.9/`
    - `kmod-1.18/`
    - `buil-time.log`
  - where all source tarballs are extracted
  - where the build of each package takes place
  - in addition to package sources & obj files, `stamp` files created by BR
  - variable `BUILD_DIR`

## p54: Build tree $(O)/host:
- `output/`
  - `host/`
    - `lib`
    - `bin`
    - `sbin`
    - `<tuple>/sysroot/bin`
    - `<tuple>/sysroot/lib`
    - `<tuple>/sysroot/usr/bin`
    - `<tuple>/sysroot/usr/lib`
  - contains both tools built for host ( Xcompile, .. ) & toolchain sysroot
  - variable `HOST_DIR`
  - Host tools are directly in `host/`
  - the sysroot is in `host/<tuple>/sysroot/usr
  - `<tuple>`is an identifier of the arch, vendor, OS, Clib & ABI
    - ex: `arm-unknown-linux-gnueabihf`
  - variable for the sysroot: `STAGING_DIR`
  
## p55: Build tree $(O)/staging:
- `output/`
  - `staging/`
  - just a symbolic link to the sysroot, ie to `host/<tuple>/sysroot/`
  - available for convenience

## p56: Build tree $(O)/target:
- `output/`
  - `target/`
    - `lib`
    - `bin`
    - `etc`
    - `usr/lib`
    - `usr/bin`
    - `usr/sbin`
    - `usr/share`
    - `THIS_IS_NOT_YOUR_ROOT_FILESYSTEM`
    - `..`
  - target rootfs
  - usually Linux hierarchy
  - not completely ready for target: permissions, device files, ..
  - BR doesn't run as root: all files are owned by the user running BR, not `setuid`, ..
  - used to generate the final root fs images in `images/`
  - variable `TARGET_DIR`

## p57: Build tree $(O)/images:
- `output/`
  - `images/`
    - `zImage`
    - `armada-370-mirabox.dtb`
    - `rootfs.tar`
    - `rootfs.ubi`
  - contains final images: kernel img, bootloader img, rootfs img(s)
  - variable: `BINARIES_DIR`

## p58: Build tree $(O)/graphs:
- `output/`
  - `graphs/`
  - visualization of BR op: deps between packages, time to build them
  - `make graph-depends`
  - `make graph-build`
  - `make graph-size`
  - variable `GRAPHS_DIR`
  - see section 'Analyzing the built later' in original training slides

## p59: Build tree $(O)/legal-infos:
- `output/`
  - `legal-infos/`
    - `manifest.csv`
    - `host-manifest.csv`
    - `licenses.txt`
    - `licenses/`
    - `sources/`
    - `..`
  - legal infos: license of all packages & src code + licensing manifest
  - useful for license compliance
  - `make legal-infos`
  - variable `LEGAL_INFO_DIR`

# TOOLCHAINS IN BUILDROOT

