# Software

How to prepare your BeagleBone to use as BBBmini.

* Debian 8.6 jessie
* GCC 4.9
* Kernel 4.4 PREEMPT RT
* BBBmini devicetree loaded at startup.

## Prepare microSD with your Linux host computer
1. Download Debian image [https://rcn-ee.net/rootfs/bb.org/testing/2016-10-02/console/BBB-blank-debian-8.6-console-armhf-2016-10-02-2gb.img.xz](https://rcn-ee.net/rootfs/bb.org/testing/2016-10-02/console/BBB-blank-debian-8.6-console-armhf-2016-10-02-2gb.img.xz)可以更新到最新的BBB debian 不带图形界面系统 小于4G.
2. Decompress image: `unxz BBB-blank-debian-8.6-console-armhf-2016-10-02-2gb.img.xz`
3. Use `lsblk` to find the address `/dev/sdX` of your microSD device, such as `/dev/sdc`. `/dev/sdX` should point to your microSD device, not partition, be careful here to make sure you don't wipe the wrong device or partition!!! Unplug your microSD card and run lsblk again then plug it back in, if you want to verify it's the correct device.
4. Copy image to microSD card (>= 2GB), the process can take 15-30 minutes depending on the speed of your microSD card: `sudo dd bs=4M if=./BBB-blank-debian-8.6-console-armhf-2016-10-02-2gb.img of=/dev/sdX status=progress`
5. `sync` and remove microSD 

## Install Debian to your BeagleBone eMMC
1. Plug prepared microSD into BeagleBone 参考BBB 官方的文档 修改
2. While holding down the boot button, apply power to the board. If there is a newer Debian installed, holding down the boot button is not necessary.
3. Wait some minutes until Debian is installed (all four LEDs turned on).
4. Remove power.
5. Remove microSD.
6. Apply power again.
7. Connect to the BeagleBone `ssh debian@beaglebone`
8. Password `temppwd`
9. Update software: `sudo apt update && sudo apt upgrade -y`
10. Install software: `sudo apt install -y bb-cape-overlays cpufrequtils g++ pkg-config gawk git make screen python python-dev python-lxml python-pip python-future`
11. Set link to pkg-config: `sudo ln -s pkg-config /usr/bin/arm-linux-gnueabihf-pkg-config`
12. Update scripts: `cd /opt/scripts && sudo git pull`
13. Expend partition: `sudo /opt/scripts/tools/grow_partition.sh`
14. Install RT Kernel: `sudo /opt/scripts/tools/update_kernel.sh --bone-rt-kernel --lts-4_4`
15. Add BBBmini DTB: `sudo sed -i 's/#dtb=$/dtb=am335x-boneblack-bbbmini.dtb/' /boot/uEnv.txt`
16. Add ADC DTBO: `sudo sed -i 's/#cape_enable=bone_capemgr.enable_partno=/cape_enable=bone_capemgr.enable_partno=BB-ADC/g' /boot/uEnv.txt`
17. Set clock to fixed 1GHz `sudo sed -i 's/GOVERNOR="ondemand"/GOVERNOR="performance"/g' /etc/init.d/cpufrequtils`
18. Reboot system: `sudo reboot`
19. Login again: `ssh debian@beaglebone`
20. Your BeagleBone is now ready to use.

## Compile ArduPilot native on BeagleBone
1. `cd ~`
2. `git clone https://github.com/ardupilot/ardupilot.git`
3. `cd ardupilot`
4.  * for ArduCopter `git checkout Copter-3.6.7`
    * for ArduPlane `git checkout ArduPlane-3.9.6` or `git checkout ArduPlane-beta` 
    * for ArduRover `git checkout Rover-3.5.0`
    * for ArduSub `git checkout ArduSub-stable` or `git checkout ArduSub-beta`
5. `git submodule update --init --recursive`
6. `./waf configure --board=bbbmini`
7. `./waf` (take about 1h20m)
8. `cp build/bbbmini/bin/* /home/debian/`

## Cross compile ArduPilot (faster) with Ubuntu computer

Get the source code:

1. `cd ~`
2. `git clone https://github.com/ardupilot/ardupilot.git`
3. `cd ardupilot`
4. `./Tools/scripts/install-prereqs-ubuntu.sh`
5.  * for ArduCopter `git checkout Copter-3.6.7`
    * for ArduPlane `git checkout ArduPlane-3.9.6` or `git checkout ArduPlane-beta` 
    * for ArduRover `git checkout Rover-3.5.0`
    * for ArduSub `git checkout ArduSub-stable` or `git checkout ArduSub-beta`
6. `git submodule update --init --recursive`
7. `./waf configure --board=bbbmini`
8. `./waf`
9. `scp build/bbbmini/bin/* debian@beaglebone:/home/debian/`

## Run ArduPilot
Now you can check your hardware [here.](../checkhardware/checkhardware.md)

ArduCopter:
`sudo /home/debian/arducopter` (plus parameter) 

ArduPlane:
`sudo /home/debian/arduplane` (plus parameter) 

ArduRover:
`sudo /home/debian/ardurover` (plus parameter) 

ArduSub:
`sudo /home/debian/ardusub` (plus parameter) 

Parameter mapping:

start parameter | ArduPilot serial port 
------------ | -------------
-A | SERIAL0
-B | SERIAL3
-C | SERIAL1
-D | SERIAL2
-E | SERIAL4
-F | SERIAL5

Check http://ardupilot.org/copter/docs/parameters.html#serial0-baud-serial0-baud-rate to set the right value for `SERIALx_BAUD` and `SERIALx_PROTOCOL`

To connect a MAVLink groundstation with IP 192.168.178.26 add `-C udp:192.168.178.26:14550`

To use MAVLink via radio connected to UART4 add `-C /dev/ttyO4`. 

If there is a GPS connected to UART5 add `-B /dev/ttyO5`. 

Example: MAVLink groundstation with IP 192.168.178.26 on port 14550 and GPS connected to `/dev/ttyO5` UART5.

`sudo /home/debian/arducopter -C udp:192.168.178.26:14550 -B /dev/ttyO5`

Example: MAVLink groundstation via radio connected to UART4 and GPS connected to `/dev/ttyO5` UART5.

`sudo /home/debian/arducopter -B /dev/ttyO5 -C /dev/ttyO4`

## Automatic start ArduPilot after boot

If ArduPilot should start automatically at boot time follow the instructions below:

1. Connect to your BeagleBone via ssh with `ssh debian@beaglebone`
2. Edit `/etc/rc.local` with `sudo nano /etc/rc.local`
3. Modify file to (use your ArduPilot file and parameter):
```
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

/bin/sleep 10
/home/debian/arducopter -B /dev/ttyO5 -C /dev/ttyO4 > /home/debian/arducopter.log &

exit 0
```
4. Save file: `Strg + o` + Enter
5. Exit nano: `Strg + x`
6. Reboot BegaleBone with `sudo reboot`
