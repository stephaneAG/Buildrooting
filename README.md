# Buildrooting
Repo regarding Buildroot stuff, mostly as reminders

- my question with no answer on official RPi forums: https://forums.raspberrypi.com/viewtopic.php?p=1975704#p1975704

## Useful sources ( aka opended tabs as I write this )

R: some to be added to/from the following recap ( tab 'OpenedTabsMix' ):
- https://docs.google.com/spreadsheets/d/12YQ2Mi7gb-CQd7J8-yZMYKiG3bI7QR9zznBGpvt9v7I/edit#gid=1466322754

R: important infos in the 'Reminders-Software' sheet ( many tabs ):
- https://docs.google.com/spreadsheets/d/1P0MzpoFP-JzDQiC2-yt9tG9NSyNF5TOxuv-8tGfj_-U/edit#gid=1034871738

## Useful for USB Gadget under Buildroot
- https://github.com/raspberrypi/gpioexpander/blob/master/gpioexpand/board/overlay/etc/init.d/S99gpioext
- https://raw.githubusercontent.com/raspberrypi/gpioexpander/master/gpioexpand/board/overlay/etc/init.d/S99gpioext
- https://github.com/raspberrypi/gpioexpander/blob/master/gpioexpand/configs/gpioexpand_defconfig
- https://github.com/hathach/tinyusb

### Useful for Yocto:
- https://www.foobarflies.io/starting-with-yocto/
- R: NOT accelerated I guess .. https://www.foobarflies.io/public-display-mode-embedded-boards/

### Useful for Buildroot
- some infos to extract ?: http://silanus.fr/sin/wp-content/uploads/2016/03/TP3.pdf
- https://docs.google.com/spreadsheets/d/12YQ2Mi7gb-CQd7J8-yZMYKiG3bI7QR9zznBGpvt9v7I/edit#gid=1466322754
- https://bootlin.com/doc/training/buildroot/buildroot-slides.pdf
- https://buildroot.org/downloads/manual/manual.html

### For the Vagrant VM
- nor used but kept: https://stackoverflow.com/questions/15193585/virtual-memory-exhausted-cannot-allocate-memory
- applied to increase cpus & memory available: https://ostechnix.com/how-to-increase-memory-and-cpu-on-vagrant-machine/
- not currently used but kept: https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04

### For Nodejs/chromium/electron
- too simple & not for my needs: https://azukidigital.com/blog/2019/electron-application-on-raspberry-pi/
- worked but I need Electron as well: https://tewarid.github.io/2014/07/23/node.js-on-raspberry-pi-with-buildroot.html
- Electron official quickstart: https://www.electronjs.org/fr/docs/latest/tutorial/quick-start
- adding a prefix to start Electorn app: https://stackoverflow.com/questions/36172442/how-can-i-get-npm-start-at-a-different-directory
- same as above from the docs: https://docs.npmjs.com/cli/v8/using-npm/config/#prefix
- custom build instructions ( to traanslate to Buildroot ?): https://www.electronjs.org/docs/latest/development/build-instructions-linux
- since MagicMirror used Electron & Chromium, grasp tips & tricks from those: https://github.com/MichMich/MagicMirror/blob/master/package.json

### Sudoing stuff
- https://stackoverflow.com/questions/48416261/where-is-su-configuration-in-the-linux-buildroot-menuconfig
- https://askubuntu.com/questions/1332035/user-permissions-for-frame-buffer
- R: modded users.tbl to add groups to user worked, missing video group to access /dev/fb0: https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file-fr
- https://stackoverflow.com/questions/3042304/how-to-determine-what-user-and-group-a-python-script-is-running-as

### reTerminal possibly interesting starting tuts for fast native GUI apps ( to mod for Buildroot )
- https://wiki.seeedstudio.com/reTerminal-build-UI-using-Qt-for-Python/

### To display stuff directly on framebuffer
- working numpy & PILlow from: https://stackoverflow.com/questions/58772943/show-an-image-with-omxiv-direct-from-memory-on-rpi
- https://pythonexamples.org/python-pillow-read-image/
- https://www.geeksforgeeks.org/python-pil-image-frombuffer-method/
- https://www.geeksforgeeks.org/get-directory-of-current-python-script/
- https://avikdas.com/2019/01/23/writing-gui-applications-on-raspberry-pi-without-x.html
- http://raspberrycompote.blogspot.com/2013/04/low-level-graphics-on-raspberry-pi-part_3.html
- http://seenaburns.com/2018/04/04/writing-to-the-framebuffer/

### PS1 & related tweaks:
- https://www.thedroneely.com/posts/tweaking-the-bash-shell/
- https://stackoverflow.com/questions/19199593/how-can-i-add-a-vertical-space-in-terminal-after-each-command
- https://superuser.com/questions/187455/right-align-part-of-prompt
- https://stackoverflow.com/questions/32443522/triangular-background-for-bash-ps1-prompt
- https://www.cyberciti.biz/tips/howto-linux-unix-bash-shell-setup-prompt.html

### WPE possibly worth digging repos
- active repo: https://github.com/WebPlatformForEmbedded/buildroot
- rpi boards: https://github.com/WebPlatformForEmbedded/buildroot/tree/main/board/raspberrypi
- post-image: https://github.com/WebPlatformForEmbedded/buildroot/blob/main/board/raspberrypi/post-image.sh
- post-build: https://github.com/WebPlatformForEmbedded/buildroot/blob/main/board/raspberrypi/post-build.sh

### Tuto's I have long been waiting to follow & test
- uses Arch & chromium: https://webreflection.medium.com/a-minimalistic-64-bit-web-kiosk-for-rpi-3-98e460419b47
- uses WPE: https://samdecrock.medium.com/building-web-applications-for-wpe-webkit-using-node-js-3347146013f3
- https://samdecrock.medium.com/building-wpe-webkit-for-raspberry-pi-3-cdbd7b5cb362
