# Shiny Server Setup - Raspbian Buster

### Tested On
* Raspberry Pi 4/4GB.
* 16 GB SD card.
* Raspbian Buster Lite.
    * Version: February 2020
    * Release Date: 2020-02-13
    * Kernel version: 4.19
    * Some steps **might** not work on other versions.
* Wired ethernet connection. 
    * Can be changed slightly to make it work for wireless connections.

### Boot to Raspbian
1. Download the latest version of Raspian from https://www.raspberrypi.org/downloads/raspbian/.
1. Install Raspbian to an SD card.
    1. Use a tool such as **balenaEtcher** (https://www.balena.io/etcher/) to write the image into the SD card.
    1. See https://www.raspberrypi.org/documentation/installation/installing-images/ for other methods.
1. (Optional) Enable SSH on the flashed OS. *This setep is not needed if you are directly connecting a monitor and peripherals*.
    1. Add an empty file named ssh to the boot partition of the SD card, using another computer (not the Raspberry pi). See https://www.raspberrypi.org/documentation/remote-access/ssh/ for more details.
1. Enter the SD card to the Raspberry Pi and connect the power.
1. Access the terminal (if directly connected or using SSH). 
    1. To use SSH on Windows, use a tool such as Putty. 
    1. **The initial boot may take some time so try to connect using SSH in a couple of minutes**.
1. Log into Raspian using the default username **pi** and the default password **raspberry**.
1. Run the command `passwd` in the command line to change the password.

### Setting Up the Raspberry Pi
1. Run the command `sudo raspi-config` and select the following actions.
    1. Expand Filesystem (Advanced Options / Expand Filesystem).
    1. Reduce GPU memory to 16MB (Advanced Options / Memory Split).
    1. Disable ''Predictable network interfaces" (Network Options / Network interface names).
1. (Optional) Disable Wifi and Bluetooth.
    1. Run `sudo nano /boot/config.txt`.
    1. Add these lines at the end.
        ```
        dtoverlay=pi3-disable-bt
        dtoverlay=pi3-disable-wifi
        ```
1. Setup a static IP.
    1. Run `sudo nano /etc/dhcpcd.conf`.
    1. Change the following lines according to the IP address that should be set.
        ```bash
        # Example static IP configuration:
        interface eth0
        static ip_address=<<192.168.0.10>>/<<24>>
        #static ip6_address=fd51:42f8:caae:d92e::ff/64
        static routers=<<192.168.0.1>>
        static domain_name_servers=<<192.168.0.1 8.8.8.8 fd51:42f8:caae:d92e::1>>
        ```
