# MinnowBoard Notes and Resources
1. Pictures
2. Case
3. Hadoop Cluster Build

## MinnowBoard Notes
* radical 10.0.0.8 MAC 002082A22092F724E
* farout 10.0.0.14 MAC 002082A22092F92F2
* heyman 10.0.0.7 MAC 002082A22092F32CC /dev/cu.usbserial-DN018PF7

* webmin https://10.0.0._:10000/

## Console
Serial console screen size: 100x31
ls /dev/tty.* ls /dev/cu.* ctrl a ctrl \ to end ps kill // finds sessions
screen /dev/cu.usbserial-DN018PF7 115200 –L

## Key Commands
* sudo nbtscan 10.0.0.1/24 # check for hosts
* i2cdetect -l
* i2cdetect -y -r 7

## Initial Setup
* sudo passwd root
* sudo passwd -u root
* sudo passwd -l root  // disable
* sudo hostname [host]
* sudo adduser [host] sudo
* sudo apt-get update
* sudo apt install wireless-tools
* sudo apt install network-manager
* nmcli d wifi list
* nmcli con up id "[ssid]"
* nmcli dev wifi con "[ssid]" password [password] name "[ssid]"
* sudo ifconfig wlx7cdd90b1132f up
* sudo iwlist wlx7cdd90b1132f scan
* sudo iwconfig wlx7cdd90b1132f essid [ssid]
* sudo apt install openssh-server
* systemctl start sshd
* systemctl restart sshd
* sudo apt install vsftpd
* sudo vsftpd restart
* sudo wget http://prdownloads.sourceforge.net/webadmin/webmin_1.820_all.deb sudo dpkg --install webmin_1.820_all.deb
* edit /etc/nsswitch.conf > hosts: files dns > hosts: files wins dns
* sudo apt-get install winbind

## Create SSH Keys
* ssh-keygen –t rsa # keys drop into /home/~user/.ssh/id_rsa
* add contents of id_rsa.pub file into authorized_keys file in /home/~user/.ssh.
* check permissions chmod 600 *
* ssh-keygen -t rsa -P "" -f id_rsa
* cat ~/.ssh/id_rsa.pub | ssh minnow@radical "mkdir -p ~/.ssh && cat >> ~/.ssh/ authorized_keys"

## Dupe SD Card
1.a. sudo fdisk -l # check volume names (eg, /dev/mmcblk0)
1.b. sudo umount [devname]
1.c. sudo dd if=[devname1] of=[devname2] bs=4096 conv=notrunc,noerror or
1.d. sudo dd if=[devname1] of=~/sd-card-copy.img
2. sudo dd if=~/sd-card-copy.img of=[devname2]

## Board Interfaces
* CR1225 or BR1225 battery for real time clock
* J2 connector (fan) 2.54mm pitch pin 1 +5VSB pin 2 GND 
* J5 jumper +pin 1 5VSB pin 2 GND
* J6 LED SATA activity pin 1 +V1P8S pin 2 GND
* J7 jumper (NVRAM Reset) SD card write protect jumper
* D1 LED power indicator
* D2 LED on/off status indicator controllable via GPIO
* Type D micro-HDMI connector CEC connected
* SoC GPIO_S0_SC_58 
* RJ45 Realtek RTL8111GS-CG chipset 10/100/1000 Ethernet
* LSE Layout
** I2S MCLK (audio interface) 
** High Speed UART1 (/dev/ttyS4) 
** High Speed UART2 (/dev/ttyS5) 
** GPIO Mapping Linux Pin EXP_I2C_SCL (I2C #6) 501 * EXP_I2C_SDA (I2C #6) 500 EXP_GPIO1 365
** EXP_GPIO2 366
** EXP_GPIO3 367
** EXP_GPIO4 368
** EXP_GPIO5 362
** EXP_GPIO6 361
** EXP_GPIO7 363
** EXP_GPIO8 364
** TLV320AIC3104 Low-Power Stereo Audio Codec for Portable Audio and Telephony
