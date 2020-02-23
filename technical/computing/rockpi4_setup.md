# Configuration of a Rockpi-4 as a navstation

## Image Install

Download armbian image with desktop from:
https://www.armbian.com/rock-pi-4/

Flash on eMMC using uSD card adapter with balenaEtcher following:
https://wiki.radxa.com/Rockpi4/getting_started

Currently, the SPI based [U-boot does not work with MBR partitions](https://forum.radxa.com/t/fail-to-boot-armbian-images-after-spi-bootloader-upgrade/2043) used by Armbian, so it is necessary to use the SOC U-boot instead. Two solutions:
 - If bootloader was previously installed on SPI Flash, erase with radxa utility (something like spi_flash_erase in /usr/local/sbin) available from the standard Radxa/Debian image:
https://wiki.radxa.com/Rockpi4/downloads
 - Connect pins 23 and 25 https://wiki.radxa.com/Rockpi4/dev/spi-install

Connect a USB-Serial (FTDI) cable between a host computer and the SBC:
https://wiki.radxa.com/Rockpi4/dev/serial-console
With the Sparkfun cable:
ORANGE=RX <--> TX=pin 8 + YELLOW=TX <--> RX=pin 10
Configure serial terminal following the instruction above
  + echo line + editing line for Putty

Install eMMC on board + keyboard and mouse + HDMI cable

Note: the serial console to a host computer is critical, because when booting out of the eMMC, Armbian does NOT initially connect to the display! It is then necessary to go through the initial setup steps (password + new user) using the serial console :-(

Boot, follow Armbian setup on the serial console. Finally, lightdm should come up on the monitor. If not, try power cycling the monitor.

sudo apt update
sudo apt upgrade

Add the Radxa PPA following:
https://wiki.radxa.com/Rockpi4/radxa-apt

Note: install heatsink as soon a board in a stable config. The CPU is running easily to the limit (90C) without heatsink, and throttles down.

## Tweaks

### armbianEnv.txt
```
captain@rockpi:~$ cat /boot/armbianEnv.txt 
verbosity=2
console=both
overlay_prefix=rockchip
rootdev=UUID=e9fb6ad8-1bc1-4c64-aef4-401780935e3e
rootfstype=ext4
usbstoragequirks=0x2537:0x1066:u,0x2537:0x1068:u
```
 - `vebosity=2` → seems to make some magic, and prevent the boot sequence to hang on something mysterious (boot time reduced by two from verbosity=1)!
 - `console=both` → allow output in the console

### Services to disable
```
systemd list-jobs
sudo systemctl stop rk3399-bluetooth.service
sudo systemctl disable rk3399-bluetooth.service
# the service below also seems to hang for nothing
sudo systemctl disable NetworkManager-wait-online.service
```

### armbian-config

- System
  - disable Avahi annouce
  - disable ssh root login
  - enable zsh and tmux
- Personal
  - Set Timezone

### Put display to sleep

    sudo apt install xfce4-power-manager 

Configure the delay with the control panel (with my display, blank/sleep/off have the same effect).

### Other

- Settings --> Appearance --> Font --> Custom DPI (144 for my 1600x1200 12in display)

- Settings --> Panel --> Display --> Row Size = 32
- Settings --> Desktop --> Icons --> Icon size = 64 + "Single click to activate items"
- Settings --> Desktop --> Background --> Folder = Pictures + Style = Centered + select desired image

- Enable multi-touch (not sure if it was finally required or not):
` sudo apt install xserver-xorg-input-mtrack`

## OpenCPN

It is necesary to compile OpenCPN from source because not debian for arm64 exist yet.

Instructions at:
https://opencpn.org/wiki/dokuwiki/doku.php?id=opencpn:developer_manual:developer_guide:compiling_linux

```
# Dependencies
sudo apt install build-essential autoconf automake git cmake gdebi

sudo apt install gettext libgps-dev wx-common libwxgtk3.0-dev libglu1-mesa-dev libgtk2.0-dev wx3.0-headers libbz2-dev libtinyxml-dev libportaudio2 portaudio19-dev libcurl4-openssl-dev libexpat1-dev libcairo2-dev libarchive-dev liblzma-dev libexif-dev libelf-dev libsqlite3-dev libjpeg-dev liblz4-dev libsndfile1-dev

# Build
mkdir source
cd source
git clone git://github.com/OpenCPN/OpenCPN.git opencpn

cd opencpn
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr ../
make -j4

# Install the Debian (do not use make install since apt would not be aware of it)
make package
sudo dpkg -i opencpn_5.0.0-1_arm64.deb

# Install the key data using apt
sudo sh -c 'echo "deb http://ppa.launchpad.net/opencpn/opencpn/ubuntu/ bionic main" > /etc/apt/sources.list.d/opencpn-raspbian.list'
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 67E4A52AC865EB40
sudo apt update
sudo apt install opencpn-gshhs-high
sudo apt install opencpn-tcdata
```

## Install the oeSENC plugin

The oeSENC plugin is tricky, because the oeserved on github is too old for arm64: we use instead the armhf one from a Pi distribution.

```
# get code, configure and build
git clone https://github.com/bdbcat/oesenc_pi.git
cd oesenc_pi
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr ../
make -j4
make package # seems to be required even if the debian package is not used
sudo make install

# install the closed source binaries
sudo dpkg --add-architecture armhf
sudo apt install gcc-8-base:armhf libc6:armhf libgcc1:armhf libstdc++6:armhf libusb-0.1:armhf 
cd /usr/bin
sudo rm oeserverd
sudo cp ~/oeserved_pi-armhf .
sudo ln -s oeserved_pi-armhf oeserverd
cd ../lib
sudo cp  ~/libsglarmhf32-2.30.0.0.so .
```