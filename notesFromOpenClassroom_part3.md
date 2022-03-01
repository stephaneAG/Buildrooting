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
