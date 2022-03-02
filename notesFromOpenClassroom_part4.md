## R: from https://openclassrooms.com/fr/courses/5281406-creez-un-linux-embarque-pour-la-domotique/5464341-initiez-vous-a-la-domotique-avec-domoticz
## Initiez-vous à la domotique avec Domoticz
### Qu'est-ce la domotique ?
- from greek 'domus' ( FR domicile ) & 'tique' ( technique )
- specifality of building with techniques to control, automate & program the habitat
- merges electronics, computer science, telecommunication & automatisms
- the network can be wired or wireless
### Exemple d'architecture
- a domotic env is mainly composed of 3 elements:
  - `connected ojects`: switches, sensors, ..
  - `box or bridge`: the element that controls our objects ( a board hosting domotic app )
  - `management interface`: mostly webpage hosted on the domotic box
### Le logiciel Domoticz
- open-source domotic solution
- dashboard, floorplans, switches, scenes, temperatures, wheather, utility, setup
- we can now the position of our objects & their states

## Déployez Domoticz sur un Linux embarqué
### Préparez votre image
- we're gonna conceive a GNU/Linux hosting the domoticz app & transform a RPi into low-cost domotic server
- we start by a 1st test using Qemu before going to a real RPi3
- as usual, we copy the dir 'Buildroot' to 'buildroot-qemu-rpi-domoticz' & prepare a test env
- `open-source domotic solution`
- `cp -R buildroot buildroot-qemu-rpi-domoticz`
- since we created a Buildroot config using our preinstalled Xcompile toolchain, we reuse it
- we copy the '.config' file containing the previous config ( after the make menuconfig ) in our new env
- `cp buildroot-qemu-rpi/.config buildroot-qemu-rpi-domoticz/`
- then we launch the configuration interface
- `cd buildroot-qemu-rpi-domoticz`
- `make menuconfig`
### Ajoutez l'application Domoticz
- the corresponding package is in 'Target Package > Miscellaneous'
- as indicated, the 'domoticz' app is available, but we need lua, gcc > 4.8 & few options ( C++, nptl, .. )
- since our gcc version is valid, we have to add Lua from 'Target Package > Interpreter languages an scripting'
- we can now enable the 'domoticz' package
- R: it needs `glibc` & not `uClibc` 
### Générez et testez votre image
- we save our config & launch the img generation
- `make`
- once we verified that we obtained an sd image, w can boot it using Qemu
- additionally to redirecting the port 22 ( ssh  of the img to local port 5022, we'll redirect domoticz port 8080 to local port 8080, to check domoticz web interface
```
cd ~/Development-tools/
qemu-system-arm \
       -kernel qemu-rpi-kernel/kernel-qemu-4.9.59-stretch \
       -cpu arm1176 \
       -m 256 \
       -M versatilepb \
       -dtb qemu-rpi-kernel/versatile-pb.dtb \
       -no-reboot -append "root=/dev/sda2 panic=1 rootfstype=ext4 rw" \
       -net nic -net user,hostfwd=tcp::5022-:22,hostfwd=tcp::8080-:8080 \
       -drive file=buildroot-qemu-rpi-domoticz/output/images/sdcard.img,format=raw \
       -nographic
```
- once startup is complete, we shoiuld have 2 services running, in their 'ok' state
- Mosquitto: mesaage sys used by domoticz
- Domoticz: instance of domoticz for domotic
### Connectez-vous à Domoticz
- now we have to verify the service works fine
- it provides access to a web interface on port 8080
- the cmd `netstat -altn | grep 8080` should indicate a process is listening on that port for TCP protocol
- the cmd `ps aux | grep domoticz` should indicate that the binary `/opt/domoticz/domoticz` is currently being executed
- studying the size of our img `df -h`, we get 80.5Mo, and `free -m` gives us RAM usage of 20Mo
- finally, we check the http server hosting the domoticz interface answers correctly
- to do so, we open a web browser ( ex Firefox ) on our Debian & connect to `http://localhost:8080`
- since we asked Qemu to redirect local port 8080 to our img during startup, it's the same as connecting to the domoticz service hosted by our img
- the homepage should indicate no sensor added
- before proceeding to generating an img for the RPi3, we shutdown this VM
- `halt`
 
## Testez Domoticz sur une véritable Raspberry PI
### Préparez votre image
- we prepare a dir `buildroot-rpi3-armhf-domoticz` to gen our img for RPi3
- `cd ~/Development-tools/`
- `cp -R buildroot buildroot-rpi3-armhf-domoticz`
- we start by applying the default params for RPi3 via the file `raspberrypi3_64_defconfig`
- `cd buildroot-rpi3-armhf-domoticz`
- `make raspberrypi3_64_defconfig`
- then start the Buildroot configuration interface
- `make menuconfig`
- R: we'll be using the 64bits version, not 32bits, of GNU/Linux
### Ajoutez Domoticz à votre image
- by default, our img is configured to use `uClibc`, so we switch it for `glibc` in `Toolchain` menu
- R: since we generate an img for RPi3 64bits, we won't modify the Xcompile toolchain, so that BR generates it
- R: if we had generated a 32bit version, we may have reused to Xcompile toolchain we installed & gain ~1 hour of generation time
- as in previous chapter, we enable Lua & DOmoticz
### Configuration de l'accès réseau
- our domotic box 'll be accessible from a web interface & the RPi won't be connected to any screen
- to allow remote access to the system, we enable the package `OpenSSH`
- the RPi3 has a RJ45 Ethernet interface but also a wifi one
- to be able to use wifi to connect our RPi to the network, we'll need the packages `iw` & `wpa_supplicant` as well as the firmware that handles wifi, plus creating the access infos
- to enable firmware ( driver ) support of wifi card, `Target Package > Hardware Handling > Firmware`
- in there, we enable `Install DTB overlays` and the `rpi-wifi-firmware` option
### Finalisez la configuration
- since or domotic box 'll be installed in a private network, it's ok to leave an OpenSSH access non-protected
- to give a password to the `root` user, we go in `System Configuration`
- last step before generation, so the sys handles correctly the RPi hardware: we enable support of dynamic peripherals via `mdev`
- in `System Configuration > /dev management > Dynamic using devtmpfs + mdev`
- R: `mdev` has 2 roles:
  - create peripherals in /dev during RPi boot depending on the available hw ( initial population )
  - handle new peripherals dunamically ( dynamic updates )
- we can now quit & save our config
- to activate peripherals handling mdev on our sys, we need to tweak the following file
- `board/raspberrypi3/post-build.sh`
- we add at its end
```
cp package/busybox/S10mdev ${TARGET_DIR}/etc/init.d/S10mdev
chmod 755 ${TARGET_DIR}/etc/init.d/S10mdev
cp package/busybox/mdev.conf ${TARGET_DIR}/etc/mdev.conf
```
- the `post-build.sh` script is executed by BR before generating the sd card img
- it's very useful to automate actions not handled by BR ( modding config files, .. )
- in our case, we recopy the example script for auto startup of mdev in the `/etc/init.d` dir, so that it's enabled during sys startup & thus activate handling of dynamic peripherals
### Générez et transférez l'image sur une carte SD
- since BR config is now done, let's got to the img generation step with the classical `make`
- let's now recopy this img onto an SD card
- we use `sudo dmesg | grep sd` to get the name of the peripheral corresponding to the SD card
- then we actually recopy the img onto the SD
- `sudo dd if=~/Development-tools/buildroot-rpi3-armhf-domoticz/output/images/sdcard.img of=/dev/sdb`
- once the copy is over, we mount both partitions in `/mnt/boot` & `/mnt/system` & check the siez of the sys via `du` cmd
```
sudo mount /dev/sdb1 /mnt/boot
sudo mount /dev/sdb2 /mnt/system
sudo du -sh /mnt/boot/ /mnt/system/
```
### Modifiez la configuration réseau
- since our 2 partitions are mounted, we modify the network config & allow the RPi to connect to a network
- we start by modifying the interfaces configuration & add `wlan0` for the wifi network
- `sudo nano /mnt/system/etc/network/interfaces`
```
auto wlan0
iface wlan0 inet dhcp
    wireless-essid <ssid>
    pre-up wpa_supplicant -B w -D wext -i wlan0 -c /etc/wpa_supplicant.conf -dd
    post-down killall -q wpa_supplicant
```
- then we tweak the `wpa_supplicant.conf` file to add params to the wifi network
- `sudo vim /mnt/system/etc/wpa_supplicant.conf`
```
network={
    ssid="<ssid>"
    scan_ssid=1
    proto=WPA RSN
    key_mgmt=WPA-PSK
    pairwise=CCMP TKIP
    group=CCMP TKIP
    psk="<wpa_passphrase>"
}
```
- then, we have to modify the configuration of the ssh server to authorize connection to the sys as root
- `sudo vim /mnt/system/etc/ssh/sshd_config`
- in it, we tweak the line `PermitRootLogin yes`
- the sys is now ready: it 'll connect to the network during boot & 'll allow ssh connection
- we can now safey unmount the SD & unplug it from laptop
- `sudo umount /mnt/boot /mnt/system`
### Le test final
- we insert the SD into the RPi, connect a screen & plug the power plug
- if all is well, we should boot like in Qemu & display ip address on end
- we get this ip addr & connect using ssh to make sure domoticz works
- `ssh root@ADRESSE_IP_DE_VOTRE_RPI`
- `ps aux | grep domoticz`
- the Rpi is now fully functional, so to terminate this class, we'll see how domoticz works

## Ajoutez des sondes dans Domoticz
### Processus d'ajout d'une sonde dans Domoticz
- domoticz offers possibility to handle sensors & data via web interface
- adding new hardware is in 3 steps:
  - 1: adding & config of hw
  - 2: adding peripherals provided by hw & get /store data
  - 3: selecting perihperals to be displayed in dashboard
- let's continue with 2 examples:
  - 1st 'll allow monitoring RPi by getting usage infos
  - 2nd 'll integrate data ( weather forecast ) for a city
### Ajoutez une sonde système sous Domoticz
- we go in web interface `http://IP_DE_VOTRE_RPI:8080`
- `Setup > Hardware > type: motherboard sensors > Add`
- `Setup > Devices > Add`
- `<star>` allows to add stuff to the dashboard
### Ajoutez une sonde météo sous Domoticz
- create account on weather underground ( https://www.domoticz.com/wiki/Virtual_weather_devices )
- `Setup > Device > type: weather underground > Add`
### Allez plus loin
- our domotic bx is fully functional, but we didn't add physical sensors
- 2 projects to create sensors for an installation:
  - RFLink: http://www.rflink.nl/blog2/
  - MySensors: https://www.mysensors.org/

## Entraînez-vous en construisant un Linux 64 bits hébergeant Domoticz
### À vous de jouer !
- 
