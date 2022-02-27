## R: as of 27/02/2022, better grasping Buildroot BUT still so much stuff to learn about, so digging other source ..
## from : https://openclassrooms.com/fr/courses/5281406-creez-un-linux-embarque-pour-la-domotique

# Partie 1 - Mettez en place un environnement de développement pour l’embarqué
## Découvrez la cross-compilation pour l’embarqué
- cours sur RPi3, mais on va aussi voir comment émuler une RPi
### Différence architecture x86/ARM
- x86: arch CISC ( Complex Instruction Set Computer )
- ARM: arch RISC ( Reduced Instruction Set Computer )
- ARM proc: simple instructions of fixed siez & executing in constant cycles num
- RISC: less transistors used
### Xcompile ?
- an exec is compatible with ONE arch
### Pourquoi émuler un processeur ARM sur x86
- less time spent while in dev/cra phase
- ex: to etst on a RPi, we copy the sys on a SD then insert it & test
- virtualisation allows to simulate the hardware
- R: limitation: 256Mo RAM while RPi can have up to 1Go
- in some cases, we can't emulate sutff easily

## Mettez en place votre environnement sous GNU/Linux
### Configuration minimale
- 2 Go RAM, +20Go HDD, 64bit proc
- Debian machine
### Préparez votre système
- to update the Debian distribution:
- `su -` # gain admin rights
- `apt-get update` # update available apps base
- `apt-get upgrade` # update the system
- As we'll need sudo:
- `apt-get install sudo` # install sudo
- `gpasswd -a user sudo` # allow <user>  the right to exec the sudo cmd
- `su - user` # get back the rights of our user account
### Installez votre environnement de compilation et de cross-compilation
- objectif du cours: apprendre à compiler es apps pr un sys embarqué:
  - chaine de compilation classique
  - chaine de X-compilation
- `sudo apt-get install build-essential` # get apps/cmd used in a dev env
- `gcc -v` # make sure gcc cmd is available
- get Xcompile toolchain for RPi, available from Github repo
- `sudo apt-get install git` # start bu getting 'git'
- `mkdir ~/Development-tools` # create a dir in our personal dir
- `cd ~/Development-tools` # go to that dir
- `git clone https://github.com/raspberrypi/tools` # get RPi tools for Xcompile
### Testez votre environnement
- tools/arm-bcm2708/arm-linux-gnueabihf/bin/ has the gcc suite for RPi
- to make sure the Xcompile toolchain is correctly installed:
- `./tools/arm-bcm2708/arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc -v`
- since this Xcompile toolchain is not installed in the sys dirs containing execs
- `echo PATH=\$PATH:~/Development-tools/tools/arm-bcm2708/arm-linux-gnueabihf/bin >> ~/.bashrc`
- `source ~/.bashrc`
- `arm-linux-gnueabihf-gcc -v`
- R: `~/.bashrc` is autoloaded by our cmd interpreter ( bash ) on a new terminal, so can be used to easily add env vars like PATH
### Commencez avec le classique "Bonjour le monde"
- start by compiling this app with classic toolchain
- `gcc -Wall -o hello-x64 hello.c` # hello-x64 from hello.c src code
- `arm-linux-gnueabihf-gcc -Wall -o hello-arm hello.c` # hello-arm
### Étudiez les exécutables obtenus
- to check the diffs between the 2 executable bin files, we use the `file`cmd
- `file hello-64`
- `file hello-arm`
- thx to the `file`cmd, we can determin the file type & extract some infos
- filetype: here, ELF ( Executable and Linkable Format )
- representation: 32 or 64 bits
- endianness: LSB or MSB
- executable type: shared object or executable
- target machine ( arch for which compiled ): x86-64 or ARM
- ABI ( Application Binary Interface ): with which OS the exec is compatible pr low-lvl calls
- liaison type: static or dynamic ( dynamic linked )
- program interpreting this executable
- ( opt ): sha1 print of the executable
- `not stripped`indicates symbols didn't get removed from the executable with `strip`cmd
- Now, we can try running thor programs:
- `./hello-x64` # will error, but we'll see later how to exec ARM code on 86 via Qemu
- `./hello-arm`
### Testez maintenant avec de l'assembleur
- let's see assembler hello world to better illustrates the differences
- `hello-x64.S` # to create with following code, will:
  - Hello world on stdout via SYS_write
  - on x8 arch, we use `int` assembly instruction to exec a sys call by placing params in registers
  - to to the equivalent of a `printf`in C, the registers have to contain:
    - `eax`: number of the sys call to call, here 4 for 'SYS_write'
    - `ebx`: number of the file descriptor in which to write, '1' for stdout
    - `ecx`: addr of thr string to display
    - `edx`: number of chars to write, here the string 'Hello World\n'
  - the 2nd call to 'int' is equivalent to 'exit(0)' in C
  - for this, `eax` must contain 1 ( call of 'SYS_exit' & `ebx` the return code ( 0 )
```
    .text
    .global _start

_start:

        movl    $len,%edx
        movl    $msg,%ecx
        movl    $1,%ebx
        movl    $4,%eax
        int     $0x80

        movl    $0,%ebx
        movl    $1,%eax
        int     $0x80

.data
msg:
        .ascii  "Bonjour le monde!\n"
        len = . - msg
```
  - as reminder, a 'sys call' allows to ask the OS to exec a task ( the ABI )
  - in our case, we ask the kernel to exec a write op or to terminate a prog execution
  - it's the Linux Kernel that does these ops, so we invoke those via 'int' & pass params via proc registers
  - we can now test that code by generating an obj file using the `as`cmd ( provided by gcc ), then generating the executable with the `ld` cmd ( link edition phase )
  - `as -o hello-x64.o hello-x64.S`
  - `ld -o hello-x64 hello-x64.o`
  - `./hello-x64`
  - then we create the `hello-arm.S` & add the following assembler code
  - equivalent of above code, but for ARM arch
  - same global structure, but differs
  - the machine language & the assembler syntax differs for the 2 archs
  - on ARM, to do a sys call, we use `svc` ( SuperVisor Call ), equivalent to 'int'
  - also, the registers are not named the same
  - we use `(r0, r1, r2, r7)` instead of `(ebx, ecx, edx, eax)`
```
.text
	.global _start

_start:
	mov r0, #1
	adr r1, msg
	mov r2, #len
	mov r7, #4
	svc #0

	mov r0, #0
	mov r7, #1
	svc #0

msg:
	.ascii "Bonjour le monde\n"
	len = . - msg
```
  - in some examples, we can fnd the instruction `swi` ( SoftWare Interrupt ) instead of `svc`
  - it's the older name of the same instruction
  - we can generate the executable from this src code using `as` & `ld` of our Xcompile toolchain
  - `arm-linux-gnueabihf-as -o hello-arm.o hello-arm.S`
  - `arm-linux-gnueabihf-ld -o hello-arm hello-arm.o`
  - finally, we can study the executabe using the cmd `file`
  - `file hello-arm`
  - as preiously, ELF format for ARM 32bits, this time static, so no libs deps
  - `printf`from `glibc`is not used & we use `SYS_write` to do stuff on stdout
  - R: generating static execs allows to not be linked to sys libs
  - BUT since the exec holds all the code instead of calling fcns, its size is bigger
  - we can use the option `-static` de `gcc` when colmpiling to gen a static exec
## Utilisez la chaîne de cross-compilation officielle de Debian
### Debian et les chaînes de compilation croisée officielles
- Debian provides an official Xcompile that can be installed via 'apt-get'
- this toolchain can be used to compile apps for embedded sys using Debian
- it's not advised to use it for other distribs like Raspiban or Buildroot (Debian deps)
- to not 'pollute' our sys, it's advised to use our toolchain in a `chroot`env
- `chroot`can be seen an an old times LXC/Docker
- Debian provides a set of cmds that allows to create a new env Debian under Debian
- here's a list of the mains arch officially supported by Debian:
  - arm64: ARM arch, 64bits proc ( RPI3, .. )
  - armel: old ARM arch, 32bits proc ( old NAS, .. )
  - armhf: new ARM arch ( v > 7 ), 32bits proc ( RPi2, .. )
  - mips: MIPS arch grand obudiste ( SGI, .. )
  - mipsel: MIPS petit boudiste ( Loongson-3, .. )
  - powerpc: PPC arch 32bits ( Apple [1994..2006] )
  - ppc64el: PPC arch 64bits ( Motorola G5, .. )
- one of the benefit of using a `chroot`env to install an Xcompile toolchain under Debian is we can delete it easily & all its deps by deleting a dir
### Préparez votre système
- we start by setting up our Debian `chroot` env ( Stretch distrib ) in our work env
- to do so, we install the package `Debootstrap` & use same name cmd to install a new minimal Debian
- `cd ~/Development-tools`
- `sudo apt-get install debootstrap`
- `sudo debootstrap stretch stretch-crossdev`
- the Debootstrap 'll DL & install many Debian packages ( ~650Mo )
- we ca now 'lock in' in our new Debian (chroot-ing ourselves )
- when we'll isse the cmd 'chroot stretch-crossdev', the sys 'll launch a new interpreter ( bash ) that 'll be locked in that dir
- `sudo chroot stretch-crossdev`
- `ls`
- in this new env, we can easily list Xcompile toolchains by looking for GCC versions
- `apt-cache search '^gcc-' | grep architecture`
### Installez une chaîne pour ARM 32 bits
- we start by adding the arch with 'dpkg' cmd, then install the related GCC version
- 1st, an ARM 32bit one: so we add `armhf` & we update the list of available packages
- `dpkg --add-architecture armhf` # arm with FPU ( Floating Point Unit )
- `apt-get update`
- then we continue with the gcc version installation
- `apt-get install gcc-arm-linux-gnueabihf`
- now, to test that toolchain, we 1st have to get out of the chroot env
- `exit` # to get out of a chroot env, we simply end the bash process locked in stretch-crossdev
- then we copy the example created in /root/ of the chroot env ( stretch-crossdev/root/ )
- `sudo cp ~/Development-tools/examples/hello.c stretch-crossdev/root/`
- then we go again in our chroot env
- `sudo chroot stretch-crossdev`
- we make sure the hellp.c file is present in /root/ & compile it
- `arm-linux-gnueabihf-gcc -Wall -o hello-arm32 hello.c`
- R: contrary to previous chapter, the toolchain was installed via pkg manager of Debiban
- the cmds are directly installed in sys apps dirs ( /usr/bin ), no need to mod PATH
- then, we 'll get the infos on the executable generated ( 'file' ) cmd
- `apt-get install file` # as we work in a ne Debian env, we 1st have to install the cmd
- `file hello-arm32` # main diffs: shared obj instead of executable, ABI version, integrity print ( sha1 )
- finally, we can quit the env: `exit`
- Nb: we can retrieve our executable from ~/Development-tools/stretch-crossdev/root/ to transfer it to target
### Installez une chaîne pour ARM 64 bits
- RPi3 has ARM 64bits proc ( SoC Broadcom BCM2837 support 64bits & compatible 32bits )
- to add the nex toolchain, 1st go to the chroot env
- `sudo chroot stretch-crossdev`
- then we add the 'arm64' arch & update the list of available packages
- `dpkg --add-architecture arm64`
- `apt-get update`
- then we install the GCC version available for this arch & compatible with RPi3
- `apt-get install gcc-aarch64-linux-gnu`
- we can now compile the exmaple with this new toolchain to gen a 64bits executable
- `aarch64-linux-gnu-gcc -Wall -o hello-arm64 hello.c` # R double aa
- finally, check the infos of the executable: `file hello-arm64`
- Additional cmds:
  - we can list the different archs added to an env with dpkg: `dpkg --print-foreign-architectures`
  - starting with Debian 9, we'll be able to install a complete dev env & not only gcc
    ( this way we'll have access to the main compilers like g++ & main libs like libssl )
  - to list the aailable envs, we can list pakcages containing the word 'crossbuild'
  - `apt-cache search crossbuild`
  - to install a complete env, ex for 64bits, we just gave to install the following:
  - `apt-get install crossbuild-essential-arm64`
  - WARN: since above cmd installs many more packages, if only needing gcc, use the following instead:
  - `apt-get install gcc-aarch64-linux-gnu`
