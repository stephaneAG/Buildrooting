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

## p61: What's a Xcompile toolchain ?
- set of tools to build & debug code for target arch from a machine running a diff arch
- ex: building code for ARM from x86 PC
- comprises: binutils, kernel headers, C/C++ libs, GCC compiler, GDB debugger (opt)

## p62: Two possibilities:
- BR offers 2 `toolchain backends`:
  - `internal`: BR builds toolchain from src
  - `external`: BR uses existing pre-built toolchain
- selected from `Toolchain -> Toolchain type`

## p63: Internal toolchain backend:
- Makes BR build entire Xcompile toolchain from source:
- lot of flexibility in config:
  - Kernel header version
  - C lib: BR supports uClibc, (e)glibc & musl:
    - glibc: std C lib: good choice if no tight space constraints ( >= 10Mb )
    - uClibc-ng & musl: smaller C libs, uClibc-ng supports non-MMU archs ( <10Mb )
  - different versions of binutils & gcc ( keep defaults unless specific needs )
  - numerous toolchain opts: C++, LTO, OpenMP, libmudflap, graphite, .. ( deps on Clib )
- Building toolchain takes 15..20min minimum

## p64: Internal toolchain backend: results:
- `host/bin/<tuple>-<tool>`: Xcompile tools: compiler ( wrapped ), linker ,assembler, ..
- `host/<tuple>`:
  - `sysroot/usrc/include/`: kernel headers & C lib headers
  - `sysroot/lib` & `sysroot/usr/lib`: C lib & gcc runtime
  - `include/c++/`: C++ lib headers
  - `lib/`: host lib needed by gcc/binutils
- `target/`:
  - `lib/`& `usr/lib/`: C & C++ libs
- compiler is configured to:
  - gen code for arch, variant, FPU & ABI selected in `Target Options`
  - look for libs & headers in sysroot
  - no need to pass weird gcc flags !

# R: External toolchain stuff skipped for now ..

## p71: Kernel headers version:
- important toolchain menu option: kernel headers version
- when building userspace programs, libs or the C lib, kernel headers are used to know
  how to interface with kernel
- this kernel/userspace interface is `backward compatible`, but can introduce features
- hence it's important to use kernel headers with version >= target kernel version
- using internal toolchain backend, choose appropriate kernel headers version

## p72: Other toolchain menu opts:
- toolchain menu offers fe other opts:
  - target optims:
    - allows to pass additional compiler flags when building target packages
    - don't pass flags to select CPU/FPU, BR already passed those
    - careful with flags passed, these affect entire build
  - target linker opts:
    - allows to pass additonal linker flags when building target packages
  - gdb/debugging related opts:
    - covered in 'App dev' section later

# MANAGING THE LINUX KERNEL CONFIGURATION

## p74: Intro:
- Linux kernel itself uses `kconfig` to define its config:
- BR can't replicate all Linux kernel config opts in its `menuconfig`
- defining Linux kernel config needs to be done in a special way
- nb: while described with Linux kernel, discussion also valid for other packages
  that use kconfig: `barebox, uclib, busybox, uboot`

## p75: Defining the config:
- In `Kernel` menu in `menuconfig`, 3 possibilities to configure kernel:
  - 1: `Using a defconfig`:
    - 'll use a defconfig provided with kernel srcs
    - available in `arch/<ARCH>/configs` in kernel srcs
    - used unmodified by BR
    - good starting point
  - 2: `Using a custom config file`:
    - allows to gie path to either full `.config` or minimal defconfig
    - usually what'll be used, so that we can have a custom config
  - 3: `Using arch def config`:
    - uses the defconfig provided by arch in kernel src tree, some have a single file
- Config can be further tweaked with `Additional Fragments`:
  - allows to pass a list of config file fragments
  - these can complement or override config opts specified in a defconfig or full config

## p76: Example of kernel config:
- custom config file:
  ```
  BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG=y
  BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="/board/<vendor>/<platform>/linux.config
  ```
- std kernel defconfig:
  ```
  BR2_LINUX_KERNEL_DEFCONFIG="<defconfigname>"
  ```
- std kernel defconfig + fragment:
  ```
  BR2_LINUX_KERNEL_DEFCONFIG="<defconfigname>"
  BR2_LINUX_KERNEL_CONFIG_FRAGMENT_FILE="/board/<vendor>/<platform>/linux.fragment
  ```
- linux fragment: contains extra kernel options:
  ```
  CONFIG_CFG80211_WEXT=y
  ```

## p77: Changing the config:
- Running [one of] the Linux kernel conf interfaces: `make linux-menuconfig`
- 'll load either the defined kernel defconfig or custom config file & start corresponding Linux kenrel config interface
- changes only made in `$(O)/build/linux-<version>`, NOT preserved across clean rebuild
- to save them:
  - `make linux-update-config`: to save a full config file
  - `make linux-update-defconfig`: to save a minimal defconfig
  - only works if a custom configuration file is used

## p78: Typical flow:
- 1: `make menuconfig`
  - start with a defconfig from kernel, say `mvebu_v7_defconfig`
- 2: `make linux-menuconfig` to customized the config
- 3: build, test, tweak config as needed
- 4: we CAN'T do `make linux-update-{config,defconfig}`, since BR config points to a kernel defconfig
- 5: `make menuconfig`
  - change to a custom config file ( no need to exist: 'll be created by BR )
- 6: `make linux-update-defconfig`
  - 'll create the custom config file as a minimal defconfig

# ROOT FILESYSTEM IN BUILDROOT

## p80: Overall rootfs construction steps:
- 1: copy the skeleton to `$(TARGET_DIR)`
- 2: build/install all packages
- 3: run a number of cleanup steps
- 4: copy rootfs overlays
- 5: execute post-build scripts
- 6: create rootfs images
- 7: execute post-image scripts

## p81: Rootfs skeleton:
- base of a Linux fs: UNIX dirs hierarchy, config files & scripts in /etc, no apps/libs
- all target packages depends on the `skeleton`package, essentially the 1st thing
  copied to `$(TARGET_DIR)` at the beginning of the build
- `skeleton` is a virtual package that 'll depend on:
  - `skeleton-init-{sysv,systemd,openrc,none}` depending on selected init sys
  - `skeleton-custom` when a custom skeleton is selected
- all of `skeleton-init-{sysv,systemd,openrc,none}` depend on `skeleton-init-common`
  - copies `system/skeleton/*` to `$(TARGET_DIR)`
- `skeleton-init-{sysv,systemd,openrc,none}` install more files specific to init syss

## p82: Skeleton packages deps:
- BR2_ROOTFS_SKELETON_CUSTOM=y
  - skeleton-custom
- BR2_ROOTFS_SKELETON_DEFAULT=y
  - skeleton-init-common
  - BR2_INIT_NONE=y
    - skeleton-init-none
  - BR2_INIT_OPENRC=y
    - skeleton-init-openrc
  - BR2_INIT_SYSTEMD=y
    - skeleton-init-systemd
  - BR2_INIT_SYSV=y & BR2_INIT_BUSYBOX=y
    - skeleton-init-sysv

## p83: Skeleton packages deps:
- can be used through `BR2_ROOTFS_SKELETON_CUSTOM` & `BR2_ROOTFS_SKELETON_CUSTOM_PATH` opts
- in such case, `skeleton` depends on `skeleton-custom`
- completely replaces `skeleton-init-*` so custom skeleton must provide everything
- no recommended though:
  - base is usually good for most projects
  - skeleton only copied at beginning of build, so a change needs a full rebuild
- use `rootfs overlays` or `post-build scripts` for rootfs customizations ( covered later )

## p84: Installation of packages:
- all selected target packages 'll be built ( busybox, Qt, OpenSSH, lighttpd, .. )
- most 'll install files in `$(TARGET_DIR)`: programs, libs, fonts, data files, conf, ..
- really the steps that 'll bring vast majority of files in rootfs
- covered in more details in section about creating own BR packages

## p85: Cleanup step:
- once all packages are installed, a cleanup is done to reduce size of rootfs
- mainly involves:
  - removing header files, pkg-config files, CMake files ,static libs, man pages, doc
  - stripping all programs & libs using `strip` to remove unnedded info
    - needs `BR2_ENABLE_DEBUG` & `BR2_STRIP_*` opts
    - additional specific cleanup steps: unneeded Python files when Python is used, ..
      ( see `TARGET_FINALIZER_HOOKS` in BR code )
      
## p86: Rootfs overlay:
- to customize content of rootfs, add config files, scripts, symlinks, dirs or any file
- `root filesystem overlay` is simply a dir whose contents 'll be copied over the rootfs
  after all packages have been installed ( ovewriting files is allowed )
- option `BR2_ROOTFS_OVERLAY` contains space-separated list of overlay paths
  ```
  $ grep ^BR2_ROOTFS_OVERLAY .config
  BR2_ROOTFS_OVERLAY="board/myproject/rootfs-overlay"
  $ find -type f board/myproject/rootfs-overlay
  board/myproject/rootfs-overlay/etc/ssh/sshd_config
  board/myproject/rootfs-overlay/etc/init.d/S99myapp
  ```
  
## p87: Post-build scripts:
- when a rootfs overlay is not enough
- can be used to customize existing files, remove unneeded ones, add new auto-generated ( build date, .. )
- executed before rootfs image creation
- can be written in any language, shell scripts often used
- `BR2_ROOTFS_CUSTOM_BUILD_SCRIPT` contains space-separated list of post-build script paths
- `$(TARGET_DIR)` path passed as 1st arg,  `BR2_ROOTFS_POST_SCRIPT_ARGS` for add. args
- various env vars available:
  - `BR2_CONFIG`: path to BR .config file
  - `HOST_DIR`, `STAGING_DIR`, `TARGET_DIR`, `BUILD_DIR`, `BINARIES_DIR`, `BASE_DIR`

## p89: Generating the fs images:
- in `Filesystem images` menu, we can select which fs image formats to gen
- to gen those images, BR'll generate a shell script that:
  - `changes the owner` of all files to `0:0` ( root user)
  - take in account `global permission & per-package device tables`
  - take into account the `global & per-package user tables`
  - runs the `fs image generation util` that deps on each fs type (`genext2fs, mkfs.ubifs, tar,..`)
- this script is executed using a tool called `fakeroot`:
  - allows to fake being root so that permissions & ownership can be modified, device files created, ..
  - advanced: possibility of running a custom script inside `fakeroot`, see `BR2_ROOTFS_POST_FAKEROOT_SCRIPT`

## p90: Permission table:
- by def, all files are owned by `root` user & permissions with which they're installed in `$(TARGET_DIR)` are preserved
- to customize ownership or permission of installed files, we can create 1/+ `permission tables`
- `BR2_ROOTFS_DEVICE-TABLE` contains a space-separated list of permission table files, option name contains `device` for backward-compatibility reasons only
- `system/device_table.txt` file used by default
- packages can also specify their own permissions ( see `Advanced package aspects` section )
- permission table example
```
#<name>     <type> <mode> <uid> <gid> <major> <minor> <start> <inc> <count>
/dev        d      755    0     0     -       -       -       -     -
/tmp        d     1777    0     0     -       -       -       -     -
/var/www    d      755    33    33    -       -       -       -     -
```

## p91: Device table:
- when system is using a static `/dev`, one may need to create additional `device nodes`
- done using one or several `device tables`
- `BR2_ROOTFS_STATIC_DEVICE_TABLE` contains space-separated list of device table files
- the `system/device_table_dev.txt` file is used by def
- packages can also specify their own device files ( see `Advanced package aspects` section )
- device table example
```
#<name>     <type> <mode> <uid> <gid> <major> <minor> <start> <inc> <count>
/dev/mem    c      640    0     0     1       1       0       0     -
/dev/kmem   c      640    0     0     1       2       0       0     -
/dev/i2c-   c      666    0     0     89      0       0       1     4
```

## p92: Users table:
- one may need to add specific UNIX users & groups in addition to the ones available in def skeleton
- `BR2_ROOTFS_USERS_TABLES` is a space-separated lsit of user tables
- packages can also specify their own users ( see `Advanced package aspects` section )
- users table example
```
#<username> <uid> <group> <gid> <password> <home>    <shell> <groups>    <comment>
foo         -1    bar     -1    !=blabla   /home/foo /bin/sh alpha,bravo Foo user
test        8000  wheel   -1    =          -         /bin/sh -           Test user
```

## p93: Post-image scripts:
- once all fs imgs have been created, `post-image` scripts are caleld at end of build
- these allow to do any custom action at the end of build, ex:
  - extracting the rootfs to do NFS booting
  - generating a final firwmare img
  - start the flashing process
- `BR2_ROOTFS_POST_IMAGE_SCRIPT` is a space-separated list of post-image scripts to call
- post-image scripts are called:
  - from BR src dir
  - with `$(BINARIES_DIR)` path as 1st arg
  - with contents of the `BR2_ROOTFS_POST_SCRIPT_ARGS` as other args
  - with a number of available env vars:
    `BR2_CONFIG, HOST_DIR, STAGING_DIR, TARGET_DIR, BUILD_DIR, BINARIES_DIR & BASE_DIR`
    
## p94: Init mechanism:
- BR supports multiple `init` implms:
  - `Busybox init`: def, simplest solution
  - `sysvinit`: old-style featureful `init` implm
  - `systemd`: moder init system
  - `OpenRC`: init system used by Gentoo
- selecting the `init` implm in the `System Configuration` menu 'll:
  - ensure necesarry packages are selected
  - make sure appropriate init scripts & conf files are instaleld by packages

## p95: /dev management method:
- BR supports 4 mthds to handle the `/dev` dir:
  - using `devtmpsfs`: `/dev` is managed by kernel devtmpfs, which auto creates device files ( def opt )
  - using `static /dev`: old way of doing `/dev`, not very practical
  - using `mdev`: `mdev` is part of Busybox & can run custom actions on device add/remove ( requires devtmpfs kernel support )
  - using `eudev` ( Tef: `udev` typo ? ): forked from `systemd`, allows to run custom action ( requires devtmpfs support )
- when `systemd`is used, only option is `udev` from `systemd` itself

## p96: Other customization options:
- various other opts to customize the rootfs:
  - `getty` options, to run a login prompt on a serial port or screen
  - `hostname` & `banner` opts
  - `DHCP network` on one interface ( for more complex setups, use an `overlay`)
  - `root password`
  - `timezone` installation & selection
  - `NLS`, Native Language Support, to support message translation
  - `locale` files installation & filtering ( to install translations only for subset of languages, or none at all )

## p97: Deploying the images:
- by def, BR simply stores the different images in `$(O)/images`
- up to user to deploy those to target device
- possible solutions:
  - removable storage ( SD, USB key ):
    - manually create partitions & extract rootfs as tarball to appropriate one
    - use tool like `genimage` to create complete image of media including all partitions
  - NAND flash:
    - transfer img to target & flash it
  - NFS booting
  - initramfs

## p98: Deploying the images: genimage
- `genimage` allows creating complete image of block device ( SD, USB key , HDD ), including multiple partitions and fs
- ex: allows img with 2 partitions: FAT for bootloader & kernel, ext4 for rootfs
- also allows placing bootloader at fixed offset in img if required
- helper script `support/scripts/genimage.sh` can be used as a `post-image`script to call `genimage`
- more & more widely used in BR def configs
- genimage-raspberrypi.cfg example:
```
( .. ) <-- std example
# Tef nb: the default .cfg doesn't add 'labels' to partitions, but supported
vfat {
  label = "boot"
}
```
- defconfig example:
```
BR2_ROOTFS_POST_IMAGE_SCRIPT="support/scripts/genimage.sh"
BR2_ROOTFS_POST_SCRIPT_ARGS="-c board/raspberrypi/genimage-raspberrypi.cfg"
```
- flash example:
```
dd if=output/images/sdcard.img of=/dev/sdb
```

## p100: Deploying the images: NFS booting
- many people try using `$(O)/target` directly for NFS booting
  - can't work, due to incorrect permissions/ownership
  - clearly explained in `THISIS_NOT_YOUR_ROOT_FILESYSTEM` file
- generate a tarball of rootfs
- use `sudo tar -C /nfs -xf output/images/rootfs.tar` to prepare NFS share

## p100: Deploying the images: initramfs
- other common use case, `initramfs`, a rootf fully in RAM
  - useful for small fs, fast booting or kernel dev
- 2 solutions:
  - `BR2_TARGET_ROOTFS_CPIO=y` to generate a `cpio` archive, that can be loaded from bootloader next to kernel image
  - `BR2_TARGET_ROOTFS_INITRAMFS=y` to directly include the `initramfs` inside kernel image ( only available when kernel is built by BR )

# DOWNLOAD INFRASTRUCTURE OF BUILDROOT

## p104: Intro
- important BR aspect: fetching src code or binary foro m3rd party projects
- DL supported from http(s), ftp, Git, Subversion, CVS, Mercurial, ..
- being able to do reproducible builds over a long period of time reuires understanding the DL infrastructure

