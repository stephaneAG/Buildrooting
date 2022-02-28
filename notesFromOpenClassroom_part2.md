# R: from https://openclassrooms.com/fr/courses/5281406-creez-un-linux-embarque-pour-la-domotique/5464231-installez-et-testez-un-linux-embarque
# Partie 2 - Émulez une Raspberry PI avec votre ordinateur
## Installez et testez un Linux embarqué
- Raspbian is a distrib GNU/Linux based on Debian
- 2 versions: lite ( good for servers ) & complete ( with 'pixel' graphic UI ( for desktop )
- admin model is same as Debian's
- most used distro, reason for which we'll install, emulate & test it
- disadvantage: generalist distro, no adapted to particular scenrios / edge cases
- in our case, we'll create a domotic box, aka server hosting one app
- R: there is also general distros for RPi, Arch, Linux ARM, Gentoo ARM, ..
- R: there is as well dedicated distros for RPi like RaspBMC, XBian, OpenElec, ..
### Émulez une architecture ARM sur une architecture x86
- Qemu allows emulating a proc or arh
- As well as emulating an arch x86 on another x86 ( virtualisation ), can do so for diff archs
- for us, we'll emulate ARM on x86
- mutli-platform ( x86 32/64 bits, PPC, Sparc, MIPS, ARM ), at least under GNU/Linux
- multi OS ( GNU/Linux, BSD, OS X, Windows )
- R: the official open source GNU/Linux Hypervisor is composed of a KVM kernel module & Qemu
- it allows to build virtualisation servers or cloud-like virtualisation infrastructures
### Exécutez du code ARM sur une architecture x86
- Before installing & emulating a complete GNU/Linux distro for ARM, let's start by trying helloW's on x86
- to do so, we'll use `qemu-arm-static` that 'll allow to exec a bin compiled for ARM on x86
- we start by installing the package that provides that cmd
- `sudo apt-get install qemu-user-static`
- R: this package installs several programs to emulate diff archs:
  - qemu-aarch64-static : émulation d'une architecture ARM 64 bits
  - qemu-alpha-static : émulation d'une architecture alpha
  - qemu-armeb-static : émulation d'une architecture ARM 32 bits grand boudiste ( Nb: WRONG translate: big endian .. for FR endianisme ou boutisme ) )
  - qemu-arm-static : émulation d'une architecture ARM 32 bits
- 1st, we go to the dir hosting C code then compile 'hello.c' with the official RPi Xcompile toolchain
- `cd ~/Development-tools/examples`
- `arm-linux-gnueabihf-gcc -Wall -o hello-arm hello.c`
- then, we can try executing it with the emulator `qemu-arm-static.`
- `qemu-arm-static ./hello-arm`
- it should error indicating a file `/lib/ld-linux-armhf.so.3` is missing
- we shall then indicate where the libs provided by the toolchain are located
- in our case, in `~/Development-tools/tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/arm-linux-gnueabihf/sysroot/`
- hence, we try again indicating that dir using `-L` option
- `qemu-arm-static -L ~/Development-tools/tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/arm-linux-gnueabihf/sysroot/ ./hello-arm`
- the hello world should exec correctly
- we can then do the same test when compiling the executable in a 'static' manner ( not likning to other libs )
- to do so, we add the option `-static`
- `arm-linux-gnueabihf-gcc -Wall -o hello-arm-static hello.c -static`
- `qemu-arm-static ./hello-arm-static` # should work without prevising libs dir
- R: see the size impact for statc compile: here it's 100 times bigger than using the sys libs (5.8Ko to 613Ko)
### Testez dans votre environnement chroot
- We'll now do the same tests in our Xcompile env 'chroot'
- we start by installing the package ` qemu-user-static` once we entered our env
- `cd ~/Development-tools/`
- `sudo chroot stretch-crossdev`
- `apt-get install qemu-user-static`
- `cd /root`
- we can then install the 32bits ARM version
- `qemu-arm-static -L /usr/arm-linux-gnueabihf ./hello-arm32`
- OR the 64bits version
- `qemu-aarch64-static -L /usr/aarch64-linux-gnu ./hello-arm64`
- R: when installing an official Xcompile toolchain in our chroot env, its provided libs are in `usr/TOOLCHAIN_NAME`
- finally, we can get out of our env
- `exit`

## Émulez une Raspberry PI avec QEMU
### Installez QEMU
- We juste used 'Qemu' to emulate an ARM proc & exec ARM code on a x86 platform
- we'll continue with emulating a complete ARM platform, starting with RPi2
- R: correct emulation of RPi3 is not officially supported as of 06/05/2021
- R: yet, executables for RPi2 'll ork for RPi3 ( if 32bits ), so system & apps generated & tested this way under Qemu shall work nicely on RPi3 we'll later use
- we start by installing 'Qemu' for ARM platform
- `sudo apt-get install qemu-system-arm`
- `qemu-system-arm -machine help` # listing the set of platforms ( machines ) supported by Qemu
- in our case, we'll want `raspi2 Raspberry Pi 2`
- R: a 'machine' allows to define components of a platform & emulate it ( aka proc + components of RPi2 )
- then we list the procs supported by the emulator for that platform
- `qemu-system-arm -machine raspi2 -cpu help`
- R: in our case, the proc we 'd need to emulate a RPi 1 ( ? was 2 ?! ) is `l'arm1176`
- R: Rpi1 / Rpi0: `arm1176`, RPi2: `Cortex-A7`, RPi3: `Cortex-A53` ( see if supported as of 28/02/2022 )
### Émulez Raspbian avec le support officiel de QEMU
- Qemu offers RPi 2 emulation in its latest versions & allows to boot an unmodded img
- 1st, we fetch a Raspbian img:
- `cd ~/Development-tools/`
- `mkdir raspbian`
- `cd raspbian`
- `wget http://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2018-04-19/2018-04-18-raspbian-stretch-lite.zip`
- `unzip 2018-04-18-raspbian-stretch-lite.zip`
- Now, we have to extract the kernel & the file describing the hardware present on the RPi 2
- the Raspbian SD img has 2 partitions:
  - 1st is VFAT with bootloader ( U-boot ), kernel ( kernel7.img ) & hw description files ( .dtb )
  - 2nd contains the img of the GNU/Linux Raspbian sys
- to extract the files we're interested in, we'll mount the Raspbian img in a tmp dir
- `sudo mkdir /mnt/rpi`
- `sudo losetup -f --show -P ~/Development-tools/raspbian/2018-04-18-raspbian-stretch-lite.img`
- `sudo mount /dev/loop0p1 /mnt/rpi`
- `cp /mnt/rpi/kernel7.img /mnt/rpi/bcm2709-rpi-2-b.dtb ~/Development-tools/raspbian/`
- `sudo umount /mnt/rpi`
- `sudo losetup -d /dev/loop0`
- to emulate a RPi with Qemu, e'll use the following cmd:
```
qemu-system-arm -M raspi2 \ # -M for machine to be emulated
                -cpu cortex-a7 \ # to indicate the proc to emulate
                -append "rw earlyprintk loglevel=8 dwc_otg.lpm_enable=0 root=/dev/mmcblk0p2" \ # to pass params to kernel, important one being 'root=', which indicates name of device that has BNU/Linux sys root ( SD 2nd partition )
                -dtb ~/Development-tools/raspbian/bcm2709-rpi-2-b.dtb \ # hw description file
                -drive file=~/Development-tools/raspbian/2018-04-18-raspbian-stretch-lite.img,if=sd,format=raw \ # to emulate a SD reader containing the Raspbian img
                -kernel ~/Development-tools/raspbian/kernel7.img \ #the kernel to be loaded by the bootloader
                -m 1G \ # the RAM size, 1GO for RPi2
                -smp 4 # processor cores, here 4

```
- if we try booting the DL-ed Raspbian using the above cmd, we may end up with a 'kernel panic'
- depending on the Qemu version, this emulation works fine or ends with error while kernel is executing
- this being said, this feature being too much unstable ( as of this writing by original author ), so we'll use another Qemu feature
### Émulez Raspbian avec un noyau modifié
- to emulate an RPi2 without crashing, we may use a modded kernel to exec a Raspbian
- contrary to 'native' emulation, this doesn't allow all components emulation ( ex: GPIOs )
- we start by fetch the kernel, modded to be Qemu compatible
- `cd ~/Development-tools/`
- `git clone https://github.com/dhruvvyas90/qemu-rpi-kernel`
- then ,we get a version of Raspbian compatible with that modded kernel
- `cd raspbian`
- `wget http://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2017-12-01/2017-11-29-raspbian-stretch-lite.zip`
- `unzip 2017-11-29-raspbian-stretch-lite.zip`
- note that we don't use the latest versionof Raspbian here for compatibility reasons with the modded kernel
- we'll later see how to recompile a modded kernel to be able to use the latests version of Raspbian
- now we create a script `start-qemu-rpi.sh` that 'll exe the Raspbian img while emulating the RPi
- `cd ~/Development-tools/`
- `nano start-qemu-rpi.sh`
-  content of that file:
```
#!/bin/bash 
qemu-system-arm -M versatilepb \
                -cpu arm1176 \
                -kernel qemu-rpi-kernel/kernel-qemu-4.14.79-stretch \
                -append "root=/dev/sda2 panic=1 rootfstype=ext4 rw" \
                -hda raspbian/2017-11-29-raspbian-stretch-lite.img \
                -m 256 \
                -dtb qemu-rpi-kernel/versatile-pb.dtb \
                -no-reboot \
                -net nic -net user,hostfwd=tcp::5022-:22
```
- the above script 'll:
  - launch 'qemu-system-arm' & emulate a board 'versatilepb' ( old embedded dev board compatible GNU/Linux )
  - we set the proc ( arm1176, the RPi1 one )
  - we won't emulate an SD card but a HDD & our root partition 'll map to device /dev/sda2
  - finally, the `-net` option 'll create a virtual network card to redirect local port 5022 to img poty 22 to connect over ssh to the emulated board
  - then, we make this script executable & execute it
  - `chmod +x start-qemu-rpi2.sh`
  - `./start-qemu-rpi2.sh``
- a grahpical window shoudl open & a sys booting to a login prompt

## Configurez la base du système
- now we can emulate a RPi with a Raspbian distro, & before going further into embedded dev, we'll review some basics
### Configuration du clavier
- login as pi/raspberry
- Raspbian has the `raspi-config` utility & we can use it to configure the keyboard to AZERTY
- `sudo raspi-config`
- we have network,boot,language/region,input/output,overclock,advanced opts, update
- for kbs, we do Localisation > Change kbd layout > Generic 105-key (Intl) PC > French > default layout > no composite key
### Activation du serveur SSH
- when runnig our image, e configured a port redirection to connect via ssh to the img, so we can use out Debian's terminal instead of Qemu's interface
- by default, ssh is NOT activated: so either use 'raspi-config' or use cli
- using cli, we simply use the `systemctl` cmd to auto activate the ssh service on next sys boot & then start it immediately
- `sudo systemctl enable ssh`
- `sudo systemctl start ssh`
- we should now have a functional ssh service
- R: to exit Qemu's interface, we use Ctrl+Alt
- we now exit the Qemu interface, launch a new terminal & try connecting via ssh onto the Raspbian
- since we redirected local port 5022 to Raspbian's SSH port 22, we open a local ssh connection as pi on port 5022 & 'll be auto-redirected to Raspbian
- `ssh pi@localhost -p 5022`
### Exécutez "Bonjour le monde"
- Let's come back to our Hello World code Xcompiled for ARM arch
- we can now check it 'll work ok on RPi by copying it onto our Raspbian
- since we have a working ssh connection, we can use `scp` to tranfer from Debian to Raspbian
- in a new Debian termlinal, we use:
- `cd /home/user/Development-tools/examples`
- `scp -P 5022 hello-arm pi@localhost:~/`
- then we come back to Qemu's interface & do our test
- `cd`
- `chmod +x hello-arm`
- `./hello-arm`
- now we now our app Xcompiled on Debian x86-64 is compatible with our ARM Raspbian
- once it's done, we can properly shutdown our Raspbian with the cmd
- `sudo shutdown -h now`

## Recompilez votre noyau pour QEMU
### Cross-compilez son noyau pour Raspberry PI
- in previous steps, we executed an old Raspbian cuz the kernel was compatible with only old versions of Raspbian
- to handle the latest Raspbian version, we have to `generate`, hence `Xcompile` a version of the Linux kernel compatible with a particular Raspbian's version
- in this writeup, we'll use Raspbian of 17/04/2018 & generate corresponding Linux kenel img 20180417
- R: another benefit in knowing how to recompile a custom/own kernel is to add patches for specific hw or activate options not available in official version
### Mise en place
- the modified kernel to work with Qemu is provided with a script that allows custom recompile
- we're gonna mod this script so that it DL & compile a kernel version correspondign to the Raspbian img DLed earlier
- 1st, we go into the `tools` dir of `qemu-rpi-kernel ` & edit the script `build-kernel-qemu`
- `cd ~/Development-tools/qemu-rpi-kernel/tools`
- `nano build-kernel-qemu`
- this script fetches by def the source of the Rpi kernel using `git`
- sinev this process is quite long, we start by desactivate the usage of `git`to force the script to directly DL the archive with the kernel's resources
- the git sources version or the archive to be DL corresponds to the variable `COMMIT`, so we also have to mod it to DL version `20180417`
- we mod the 3 following lines at the beginning of the file
  - `COMMIT=6820d0cbec64cfee481b961833feffec8880111e`
  - `COMMIT=raspberrypi-kernel_1.20171029-1`
  - `#COMMIT=""`
- we change the above to be like this:
  - `#COMMIT=921b6254452db87ae0304beaa9833cfdf5083ed8`
  - `#COMMIT=raspberrypi-kernel_1.20171029-1`
  - `COMMIT="raspberrypi-kernel_1.20180417-1"`
- then ,we comment the line `USE_GIT=1` ( to `#USE_GIT=1` ) to disable git support
- by def, the script DLs the archive with the kernel sources each time it's run, not desired for unconnected tests & not loosing time
- we're gonna mod it to verify the presence of the archive, & if exist, ignore the DL phase
- hence we mod the line `wget -c https://github.com/raspberrypi/linux/archive/${COMMIT}.zip -O linux-${COMMIT}.zip`
- to
  - `if [ ! -f linux-${COMMIT}.zip ]`
  - `then`
  - `      wget -c https://github.com/raspberrypi/linux/archive/${COMMIT}.zip -O linux-${COMMIT}.zip`
  - `fi`
- R: even if the script automates DL & compiling the Linux kernel ot make it compatible with a RPi emulated by Qemu, it's nice to know its content to analyze the different steps involved to recompiling  
### Configurez et compilez votre futur noyau
- we can now run the script
- `bash build-kernel-qemu`
- once DL & unzipping of the archive with kernel sources is done, we'l get a screen offering some tweaks for the kernel params
- Tef R: the said-screen is ( seems to be ? ) the `sudo make linux-menuconfig` one ( when using Buildroot )
- hopefully, the kernel has been pre-configured with the params mandatory for a RPi emulated via Qemu  ( Tefnb: aka, some rpi-<n>-qemu-defconfig ? )
- so, we just have to select 'Exit' then 'Save'
- R: it's on this step that we can enable/disable support of some features in the kernel
- ex: Wifi, RJ45, specific file systems, ..
- once we quit the kernel configuration interface, the compiling phase starts ( can take few minutes to a dozen of those )
- once done, the kernel is now compiled & is is the following file: `arch/arm/boot/zImage`
### Testez votre nouveau noyau
- now, let's try to boot our Raspbian with this kernel
- we start by recopying this kernel in the Raspbian dir, via the following cmd:
- `cp linux-raspberrypi-kernel_1.20180417-1/arch/arm/boot/zImage ~/Development-tools/raspbian/kernel-20180417-1`
- then, we start Qemu loading that new kernel
```
cd ~/Development-tools/
qemu-system-arm -kernel raspbian/kernel-20180417-1 \
                -cpu arm1176 \
                -m 256 \
                -M versatilepb \
                -dtb qemu-rpi-kernel/versatile-pb.dtb 
                -no-reboot \
                -append "root=/dev/sda2 panic=1 rootfstype=ext4 rw" \
                -net nic \
                -net user,hostfwd=tcp::5022-:22 \
                -drive file=raspbian/2018-04-18-raspbian-stretch-lite.img,format=raw \
                -nographic
```
- note that we specified, using the `kernel`option, the kerne we just compiled, to have Qemu use it to boot Raspbian
- R: the `-nographic`option allows to load Qemu without opening a new graphical window ( so Qemu execution occurs in the current terminal )  
- we can now open a session with 'pi' user & enter the following cmd to check the kernel infos:
- `uname -a`
- among other things, we get the compiling date & the version of the current kernel
- version sohuld be `4.14.34` & date hould be the one of the kernel compiling
- we now know how to emulate a RPi using Qemu & exec a classic GNU/Linux, Raspbian in our case
- In the next part(s), we'll see how to generate a custom GNU/Linux distro adapted to our needs
