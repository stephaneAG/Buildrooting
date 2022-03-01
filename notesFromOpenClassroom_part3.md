## R: from https://openclassrooms.com/fr/courses/5281406-creez-un-linux-embarque-pour-la-domotique/5464296-faites-vos-premiers-pas-avec-buildroot-et-l-environnement-microlinux

# Partie 3 - Utilisez un environnement dédié à la génération de Microlinux embarqué
## Faites vos premiers pas avec Buildroot et l’environnement Microlinux
### Pourquoi générer son propre GNU/Linux ?
- R: Raspbian is a distro compatible with ARM & RPis, installed rapidly, with a classic GNU/Linux & all tools to admin a sys
- major drawback: ilpact on perfs of the obtained sys, plus disk space taken of many Gos
- other drawback: boot time, since Raspbian offers a desktop-style setup, all classic services are executed on boot
- we obtain a multi-users sys with or without gui
- finally, memory impact: the provided apps & executed services are the classic desktop versions, not optimised for embedded
- R: when one's goal is to dev an embedded sys delivering a services ( ex: domotic ), it's advised to generate a sys with only that service
- hence, instead of using Raspbian & add services/apps on top, better generate a minimal GNU/Linux using embedded-dedicated services/apps
- benefits of that approach:
  - reduced disk space taken: from few Gos to few Mos
  - reduced RAM footprint: less apps/services running
  - faster boot time: only launches the target app & its deps
### Les principaux composants d'un GNU/Linux embarqué
- a classic GNU/Linux sys doesn't have same constraints as an embedded sy
- to reduce disk space used, memory footprint & boot time, some apps can be integrated
- the `uClibc` is a reduced version of the std C lib `glibc`
- originally, this lib was created for the `uLinux` project, to conceive a minimal GNU/Linux for embedded
- while `glibc` weights 200-400Mo, this version only is few Mos
- no miracles here: a bin 100 times smaller means features/perf impact
- to reduce the final size, some fcns or optimis were suppressed in `uCLibc`, hence some apps are not compatible & need `glibc` to be installed
- GNU/Linux is a set of std UNIX cmds, like ls, touch, .. more than 200 installed on a classic distro
- put one after another, these has a huge impact on disk space
- `Busybox` is an app that implements all these cmds in one executable
- also, Busybox offers a lightweight version of the cmds, handling only essential portions, helping in reducing disk space taken
- `TinyLogin` is a lightweight version of the suite handling auth & users on GNU/Linux
- just as Busybox, it combines in one executable the different cmds of managments like `adduser`, `passwd`, `su`, `login`, ..
- once again, the goal is to reduce the final image size
- Nb: this poject is now integrated to Busybox
- the Busyox project also provides alightweight version of the services boot system `SYSVinit`
- on sys boot, the kernel loads, then executes the launch procedures of services `Busybox init`
- this app has for task to exec the startup scripts stored in `/etc/init.d/`
- this vesion of `init` is also lightweight & dedicated to embedded sys, allowing a sys boot in few seconds
### Quelques exemples de plateformes de génération de systèmes pour l'embarqué
- to create an embedded GNU/Linux, 2 possibilities:
  - either build the project `by hand`by DLing apps sources, libs, kernel, .. by ourselves & handling deps, aka `Linux from scratch`
  - otherwise, we can use a `generation kit`written specially for that
  - 4 projects ( mainly ) offers such so-called genration kit:
    - OpenEmbedded, since 2K3, framework for building & Xcompiling
      - actually more a base to build a generation kit than a full kit
      - recommended as building sys for the Yocto project
      - handling building the base of a sys by DLing the sources, creating a Xcompile toolchain & compiling a base sys ( kernel, libs, init, .. )
    - Yocto is a Linux fundation whose goal is to provide tools allowing to build Linux sys for embedded, whatever the hw used
      - it wishes to offer a unified solution by regrouping different existing tools/projects
      - 1st version dates ok 2K10
      - contrary to Buildroot, Yocto doesn't have any 'graphical' interface to configure the target sys: all is done via config files
      - hence, Yocto is harder to grasp, but offers more advanced possibilities for an industrial approach
    - Buildroot is a tool to automate the generation of a Linux sys for an embedded board
      - since 2K5
      - offers a 'graphical' interface allowing to configure the main elements of a sys ( arch, kernel app to install ), then handles DLing sources, generating a Xcompile toolchain, compile & generate the sys, finally building an disk img of the sys
      - offers the advantage of providing default configurations for the most used boards ( ex: RPi's ), which makes it an excellent choice to start with
    - OpenWRT is a distro for embedded Linux sys
      - targets mainly wifi routers ( allows replacing the sys on routers for a more advanced version than can handle additonal apps )
      - is also a framework for building apps for an embedded sys using the OpenWRT firmware
### Et maintenant ?
- our goal is to create a domotic box with a uLinux working on RPi
- to build such sys, we're gonna use one of the sys generation kits
- the choice is between Yocto & Buildroot, and we'll choose Buildroot
- even if old school austere interface, it offers a config interface, simpler than having to start by editing a dozen of files
- moreover, Buildroot fully supports the RPi boards & many apps we may need
- the next chapter is on installing & getting a grip on Buildroot

## Prenez en main votre environnement Buildroot
### Installez l'environnement Buildroot
- we start by installing its deps
- `sudo apt-get install libncurses5-dev bc`
- then we clone Buildroot sources in our dev dir
- `cd ~/Development-tools`
- `git clone https://github.com/buildroot/buildroot.git`
- Buildroot's dir has the following dirs:
  - docs: text doc
  - package: config files & compile ( Makefile ) for apps provided by Buidroot
  - project: config files & compilation for creating the project ( a project allows to generate automatically a target sys )
  - board: config files for different boards supported by Buildroot
  - configs: config files for the kernel specific to the board ( RPi, .. )
  - arch: config files for the kernel specific to the arch ( ARM, .. )
  - target/linux: 'll contain the recompiled linux kernel
  - target/<fstype>: 'll contain the image of the root fs ( the SD card's content in our case )
  - toolchain: the Xcompile toolchain generated by Buildroot
### Support pour Raspberry fourni par Buildroot
- we 1st have to make sure BR is compatible with our RPi
- for this, we list the available dirs in the `/board`dir & make sure `raspberrypi` is listed
- `cd ~/Development-tools/buildroot/`
- `ls board/ | grep rasp`
- the `board/raspberrypi/genimage-raspberrypi3.cfg` describes how to generate the SD card:
  - `boot.vfat`: 1st partition of 32Mo is FAT32 format é 'll be marked as bootable
  - the necessary files 'll be copied to it:
    - the `.dtb`files contains hw description of the board
    - `bootloader.bin`, `start.left` are the files necessary to boot
    - `zImage` is the kernel
    - `cmdline.txt` is the kernel's config file
    - `config.txt` is the config file of the RPi's peripherals ( graphic output, wifi, bluetooth, .. )
  - `sdcard.img`: 2nd partition 'll use an ext4 fs & 'll correspond to the root of the Linux fs ( `rootfs` )
- the `configs`dir conains the default configs of Buildroot & the kernel for different boards it supports
- we can verify our RPi is present
- `cd ~/Development-tools/buildroot`
- `ls configs/raspberrypi*`
- examining the `configs/raspberrypi3_defconfig` gives a handful of infos:
  - it allows to generate a kernel for an ARM arch & a Cortex A53 cortex, bsed on generic config for the SoC BCM2709
  - by default, the config activates the ext4 fs by def for the kernel
  - during startup, the bootloader loads the kernel that executes
  - then, the kernel mounts the rootfs to be able to execute the startup service ( `systemd` or `init` )
  - if the fs corresponding to rootfs is not enabled by def in the kernel, the kernel won't be capable of mounting the rootfs & thus launching the startup service
  - this error translates to a `kernel panic`: `VFS not synced`
### Générez votre première image
- we can now generate our 1st minimal img of GNU/Linux from Buildroot
- to NOT loose our config nor the generated sys on each test, we're gonna create some dirs that differs between the board & the target sys ( Qemu or RPI )
- thus, we'll name our dirs as the following: `buildroot-{qemu,carte}-{architecture}`
- we start by creating a dir to gen the img we're gonna test in Qemu for an ARM 64bits sys ( ex RPi3 )
- to do so, we duplicate the dir `buildroot` as `buildroot-qemu-aarch64 `
- `cd ~/Development-tools`
- `cp -R buildroot buildroot-qemu-aarch64`
- for this 1st test, we'll use the config provided by Buildroot
- this way, fenerating the sys 'll only require 3 cmds
- 1st, we go to the BR dir related to our sys
- then, we configure BR for the target sys ( using the provided defconfig file )
- then, we finally run the sys generation
- `cd buildroot-qemu-aarch64`
- `make qemu_aarch64_virt_defconfig`
- `make`
- the last cmd, `make`, 'll:
  - generate the Xcompile toolchain for the desired arch
  - compiling, via the Xcompile toolchain, the apps/services to gen the sys
  - compiling the kernel
  - generating the rootfs by creating a disk img ( `rootfs.ext4` ) & recopying the generated sys
- R: for this 1st test, we'll use the defconfig of Buildroot
- this config needs generating a Xcompile toolchain & can take some time
- later on, we'll use the Xcompile toolchains installed, to reduce time taken by about 1 hour
- once generation is complete, we get 2 files in `output/images`:
  - the kernel ( the `Image` )
  - the rootfs ( `rootfs.ext4`, actually a link to `rootfs.ext2` )
- compared to std Raspbian img, we went from 2Go to 60Mo
### Testez votre image
- to test our img with Qemu, we'll use the cmd `qemu-system-aarch64` to emulate a 64bits ARM sys with proc CortexA57 mono-core & an Ethernet network card
- to start on the kernel & disk img we just obtained:
```
qemu-system-aarch64 -M virt \
                    -cpu cortex-a57 \
                    -nographic \
                    -smp 1 \
                    -kernel output/images/Image \
                    -append "root=/dev/vda console=ttyAMA0" \
                    -netdev user,id=eth0 -device virtio-net-device,netdev=eth0 \
                    -drive file=output/images/rootfs.ext4,if=none,format=raw,id=hd0 \
                    -device virtio-blk-device,drive=hd0
```
- on startup end, we shall get the sys `Welcome Banner` ( ex: 'Welcome to Buildroot' )
- we can now compare boot time for a classic Raspbian img & a lightweight Buildroot one: few seconds
- we ca now auth as `root` user without password
- let's quickly check the memory footprints & disk usage of that img
- the `free -h` cmd displays RAM usage, and `df -h` disk usage
- our version uses 15Mo of RAM, and although we generated a 60Mo disk, it's actually using only 2.9Mo
  
## Créez et configurez votre image Linux pour Raspberry PI
- we just created an img using the default config for Buildroot, but it offers an interface for tweaking
- we start here by creating a new Buildroot dir which we'll use to gen the img for RPi & test with Qemu
- `cd ~/Development-tools`
- `cp -R buildroot buildroot-qemu-rpi`
### Prenez en main l'interface de configuration
- we can `preconfigure` Buildroot with the config provided for RPi1 by using the related `defconfig`
- `cd buildroot-qemu-rpi`
- `make raspberrypi_defconfig`
- then, we launch the configuration interface
- `make menuconfig`
- this interface contains a lot of options, only fe ones will be reviewed here
- the 1st menu configures the `target sys` depending on the board used
  - in it, we'll set the arch, executables format, instructions type, ..
  - options are pre-filled via `defconfig`
  - in our case, arch ARM type ARM1175, executables format ELF ( Executable Linux Format )
- the `toolchain` menu allows to configure the Xcompile toolchain
  - by def, Buildroot DLs & compile its own toolchain, which can take quite some time
  - since we installed a toolchain on our Debian, we're gonna tweak things here to use it
  - this menu also allows to configure options related to the Xcompie toolchain
    - type of C li to use: `uClibc` is adapted to embedded sys, or `glibc`
    - the kernel version to be used, to not call non-existing fcns or not compatible ones
    - options related to the C lib ( multilocales support, thread, .. )
    - the utils version for the binaries ( `binutils` )
    - the GCC compiler options
  - R: even if `uClibc` is most adapted for embedded sys, if app needs `glibc`, either switch or tweak the app itself
- the `System` menu allows to tweak the obtained sys
  - hostname & welcome banner
  - storage format for passwords & root password
  - the services startup sys ( Busybox, init, systemd )
  - handling peripherals in /dev & persistent peripherals to create
  - network configuration ( dhcp on interface eth0 )
  - default localisation & local datetime
- the `Kernel` menu allows to tweak the kernel
  - by default, Buildroot DLs the indicated version, applies the `defconfig` ( `bcmrpi` ), then open the interface for kernel configuration ( `menuconfig` ) to adapt, if needed, the params
  - it's also possible to indicate Buildroot that we provide an already-compiled kernel image
- the `Target Packages` menu contains all the apps/libs we can install onto our targt sys
  - these apps are grouped in different menus depending on their category
  - if an app needs other apps to be installed to be instaleld itself, Buildroot indicates the needed deps
- the `Filesystem images` menu allows to choose the final format(s) of our image
  - in our case, we generate an img of an SD card containing an ext4 rootfs, we we could generate an archive ( tar ) of that sys, a livecd image ( SquashFS ), ..
- the `Bootloaders` menu allows to choose the bootloader, the equivalent of `GRUB` on PC ( in the case of a RPi, it's `U-Boot` )
- `Host utilities` menu has some apps are required on the host machine to generate the final image
- the last menu, `Legacy config`, contains the list of options currently selected but no longer supported by Buildroot ( BR 'll refuse to generate the image while these are selected )
- R: using the configs Buildroot provides, we won't have problems with obsolete options, but may happen if using an old config
## Personnalisez la configuration
- before generating the sys img & testing it, we're gonna mod the def config
- to gain time during generation & since we have the official RPi fundation toolchain installed, we may mod the config to use that toolchain for Xcompiling
- we go in the `Toolchain` menu, then choose the type of external chain ( `External toolchain > Toolchain Type` )
- then, we have to select a custom toolchain ( `Custom toolchain > Toolchain` ) that is already installed ( `Pre-installed Toolchain > Toolchain Origin` )
- we can now acces the menu `Toolchain path`, which allows to indicate the dir containing the Xcompile toolchain & the prefix of executables of the GCC suite
- in our case, the path is `/home/user/Development-tools/tools/arm-bcm2708/arm-linux-gnueabihf/` and the prefix is `arm-linux-gnueabihf`
- we then have to indicate the version number of GCC ( `External toolchain gcc version` (4.9x) ), the kernel version ( `External toolchain kernel header series` (4.1.x) ) and the C lib type ( `External toolchain C library` (glibc/eglibc) )
- finally, the toolchain provided for the RPi having C++ support, we nable the related option ( `Toolchain has C++ support` )
- if we try to generate the img immediately, we're gonna get an error during SD card generation
  - by default, the generated `rootfs` has size 60Mo
  - since we're using the official Xcompile toolchain & it uses `glibc` & not `uClibc`, the final img exceeds 60Mo and the copy fails
  - hence, we have to increase the size of the fina limg
  - to do so, we go in the `Filesystem images` menu and increase the size to 200Mo
- now, we can `exit` Buildroot & save our modifications

### Générez votre image
- finally, we generate the img
- `make`
- since the Xcompile chain is already installed, this should be quite faster than when generating the `aarch64` img of the previous chapter
- we obtain a `sdcard.img` file containing the 2 partitions
- the generated files are available in the dir `output/images`:
  - the hw description files `.dtb`
  - the image of the FAT32 partition ( `vboot.vfat` ) that contains among others bootloader & kernel
  - the image of the root partition ( `rootfs.ext2` ) containing the GNU/Linux sys
  - the `rpi-firmware` dir continaing the precompiled binaries provided by the RPI fundation ( start.elf, bootcode.bin, ..) required for startup
  - the `sdcard.img` file is an SD img that we generated
  - `zImage` file corresponds to the kernel
- now we've generated our image, let's test it
  
## Testez l’image obtenue et comparez avec l’image précédente
- We just generated a minimal GNU/Linux img using BUildroot
- in this part, we're gonna test this img with Qemu & verify our 'Hello World' Xcompiled previously Xcompiled works on that sys
### Testez l'image avec QEMU
- we go into our dir containing the img we just generated & execute that img with Qemu
```
cd buildroot-qemu-rpi
qemu-system-arm -kernel ../qemu-rpi-kernel/kernel-qemu-4.9.59-stretch \
                -cpu arm1176 \
                -m 256 \
                -M versatilepb \
                -dtb ../qemu-rpi-kernel/versatile-pb.dtb \
                -no-reboot \
                -append "root=/dev/sda2 panic=1 rootfstype=ext4 rw" \
                -net nic -net user,hostfwd=tcp::5022-:22 \
                -drive file=output/images/sdcard.img,format=raw \
                -nographic
```
- for this test, we use a proc ARM1176 to emulate a RPi1
- the sys we generated in contained in the SD img `sdcard.img` we pass as param
- once the sys startup is complete, we can auth ourselves as `root`
- then, we check the memory footprint of GNU/Linux using `free -m` & disk usage using `df -h`
- the memory footprint should be small ( ex 7Mo )but this img uses more disk space than the `aarch64`, since we use `glibc` for this test ( due to the official toolchain provided by RPi fundation ) and not `uClibc`, yet with 55.5Mo used, we're still quite minimal
- we can now shutdown the mahine
- `halt`
### Transférez un fichier sur l'image de la carte SD
- we Xcompiled in the 1st part a C code eample as 'Hello World'
- we're gonna make sure it works in our Buildroot img as with our older Raspbian one
- we have 2 possibilities to transfer the binary `hello-arm`:
  - copying it over the network using `scp`
  - copying it directly on the SD card
- the `sdcard.img` file contains the SD img with 2 partitions
- to be able to mount these onto dirs, we have to make the associated sys assign devices to those partitions
- this translates to virtually insert the SD card in a SD reader
- to do so, we use the `kpartx` cmd that can associate virtual peripherals to partitions contained in a disk img
- 1st, we install that cmd
- `sudo apt-get install kpartx multipath-tools`
- then we use `kpartx` to create the peripherals associated to each img partition
- `sudo kpartx -a -v /home/user/Development-tools/buildroot-qemu-rpi/output/images/sdcard.img`
- then, we create 2 dirs in `/mnt`, one for the 1st FAT32 partition & another ofr the 2nd EXT4 rootfs
- `sudo mkdir /mnt/boot/ /mnt/system/`
- then, we mount both peripherals created by `kpartx` onto those dir
- `sudo mount -o loop /dev/mapper/loop0p1 /mnt/boot/`
- `sudo mount -o loop /dev/mapper/loop0p2 /mnt/system/`
- we ca now copy the binary `hello-arm` into the dir `/mnt/system/root/`, which 'll allow us to find it later in the personal dir of root once the sys has booted
- `sudo cp ~/Development-tools/examples/hello-arm /mnt/system/root/`
- now, let's virtually eject the SD card
- to do so, we unmount the dirs, then destroy the peripherals `kpartx` created
- and finally, we eject the disk using the `losetup` cmd
- `sudo umount /mnt/boot /mnt/system`
- `sudo kpartx -d /dev/loop0`
- `sudo losetup -d /dev/loop0`
### Exécution de "Bonjour le monde"
- now we have the binary copied onto the SD card, we can boot the img with the previous cmd & reconnect as root, then check if our binary works by executing it
- `./hello-arm`
- we just built & tested our 1st GNU/Linux for embedded on a RPi, so let's go onto a real-life example on how to build a domotic box
