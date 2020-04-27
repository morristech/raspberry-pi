# Shiny Server Setup on Raspbian Buster

This document is based on https://withr.github.io/install-shiny-server-on-raspberry-pi/ and https://community.rstudio.com/t/setting-up-your-own-shiny-server-rstudio-server-on-a-raspberry-pi-3b/18982.

### Tested On
* Raspberry Pi 4/4GB.
* 16 GB SD card.
* Raspbian Buster Lite
    * Version: February 2020
    * Release Date: 2020-02-13
    * Kernel version: 4.19
    * Some steps **might** not work on other versions.
* Wired ethernet connection. 
    * Can be changed slightly to make it work for wireless connections.
* R
   * Version: 4.0.0 (Arbor Day)
   * Release Date: 2020-04-24

### Boot to Raspbian
1. Download the latest version of Raspian from https://www.raspberrypi.org/downloads/raspbian/.
1. Install Raspbian to an SD card.
    1. Use a tool such as **balenaEtcher** (https://www.balena.io/etcher/) to write the image into the SD card.
    1. See https://www.raspberrypi.org/documentation/installation/installing-images/ for other methods.
1. (Optional) Enable SSH on the flashed OS. *This step is not needed if you are directly connecting a monitor and peripherals*.
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
    1. Disable "Predictable network interfaces" (Network Options / Network interface names).
1. (Optional) Disable Wifi and Bluetooth.
    1. Run `sudo nano /boot/config.txt`.
    1. Add these lines at the end.
        ```
        dtoverlay=pi3-disable-bt
        dtoverlay=pi3-disable-wifi
        ```
1. Setup a static IP. *This step is not needed if static IPs are set on the switch/router.*
    1. Run `sudo nano /etc/dhcpcd.conf`.
    1. Change the following lines according to the IP address that should be set.
        ```
        # Example static IP configuration:
        interface eth0
        static ip_address=192.168.0.10/24
        #static ip6_address=fd51:42f8:caae:d92e::ff/64
        static routers=192.168.0.1
        static domain_name_servers=192.168.0.1 8.8.8.8 fd51:42f8:caae:d92e::1
        ```
1. Enable all the repositories.
      1. Run `sudo nano /etc/apt/sources.list`
      1. Uncomment the line with the repositories.
1. Update packages
      1. Run `sudo apt update && sudo apt full-upgrade`.
1. Set up swap memory (used 3GB in this case).
   1. Run the following commands.
      ```
      sudo /bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=3072
      sudo /sbin/mkswap /var/swap.1
      sudo /sbin/swapon /var/swap.1
      sudo sh -c 'echo "/var/swap.1 swap swap defaults 0 0 " >> /etc/fstab'
      ```
1. Reduce the unnecessay use of the swap memory.
      1. Run `sudo nano /etc/sysctl.conf`.
      1. Ad this line at the end `vm.swappiness=10`.

### Setting Up HTML and SQL Servers

1. Install the HTML server (Nginx is used here).
      1. Run the following commands.
         ```
         sudo apt install nginx
         sudo chown -R www-data:pi /var/www/html/
         sudo chmod -R 770 /var/www/html/
         ```
1. Install the SQL server (PostgreSQL is used here).
      1. Run the following command.
         ```
         sudo apt install postgresql libpq-dev postgresql-client postgresql-client-common
         ```
      1. Check the version installed. Run `pg_config --version`.
1. Configure PostgreSQL.
      1. Run the command `sudo nano /etc/postgresql/11/main/pg_hba.conf`. Use the correct version here.
      1. Modify the "local" line to `local all all md5`.
      1. Add the following lines
         ```
         host all all 192.168.0.0/24 trust
         host all all 0.0.0.0/0 password
         ```
      1. Run the command `sudo nano /etc/postgresql/11/main/postgresql.conf`.
      1. Add the line `listen_addresses='*'`.
1. Restart postgresql service.
      1. Run `sudo systemctl restart postgresql`.
1. Create a user named **pi** and a database named **pi**.
      1. Run ```sudo su postgres```.
      1. Run `createuser pi -P --interactive`. Then enter a password.
      1. Run `psql`.
      1. Run 
         ```
         create database pi;
         \q
         exit
         ```

### Install R
Most compilations can take a long time to run.
1. Install dependencies for R, shiny-server and RStudio. Run the following code.
      ```
      sudo apt-get install -y gfortran libreadline6-dev libx11-dev libxt-dev \
       libpng-dev libjpeg-dev libcairo2-dev xvfb \
       libbz2-dev libzstd-dev liblzma-dev \
       libcurl4-openssl-dev libssl-dev libxml2-dev \
       texinfo texlive texlive-fonts-extra \
       screen wget openjdk-8-jdk git
      ```
1. Download and extract the source files. Use the link for the latest version on CRAN.
      ```
      cd /usr/local/src
      sudo wget https://cran.rstudio.com/src/base/R-4/R-4.0.0.tar.gz
      sudo su
      tar zxvf R-4.0.0.tar.gz      
      ```
1. If using an R version >=4.0.0, install PCRE2. Select the latest version from https://ftp.pcre.org/pub/pcre/.
      ```
      wget https://ftp.pcre.org/pub/pcre/pcre2-10.34.tar.gz
      tar zxvf pcre2-10.34.tar.gz
      cd pcre2-10.34
      ./configure
      make
      make install
      cd ..
      rm -rf pcre2-10.34*
      ```
1. Install R.
      ```
      cd R-4.0.0
      ./configure --enable-R-shlib
      make
      make install
      cd ..
      rm -rf R-4.0.0*
      exit
      cd
      ```
      
### Install Shiny-Server
1. Install R package dependencies for shiny-server.
      ```
      sudo su - -c "R -e \"install.packages('later', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('fs', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('Rcpp', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('httpuv', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('mime', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('jsonlite', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('digest', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('htmltools', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('xtable', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('R6', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('Cairo', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('sourcetools', repos='http://cran.rstudio.com/')\""
      sudo su - -c "R -e \"install.packages('shiny', repos='http://cran.rstudio.com/')\""
      ```
1. Install cmake from source. Select the latest version from https://cmake.org/files/.
      ```
      wget https://cmake.org/files/v3.17/cmake-3.17.1.tar.gz
      tar xzf cmake-3.17.1.tar.gz
      cd cmake-3.17.1
      ./configure
      make
      sudo make install
      cd
      rm -rf cmake-3.17.1*
      ```
1. Install shiny-server
      ```
      git clone https://github.com/rstudio/shiny-server.git
      cd shiny-server
      DIR=`pwd`
      PATH=$DIR/bin:$PATH
      mkdir tmp
      cd tmp
      PYTHON=`which python`
      sudo cmake -DCMAKE_INSTALL_PREFIX=/usr/local -DPYTHON="$PYTHON" ../
      sudo make
      mkdir ../build
      sed -i '8s/.*/NODE_SHA256=a865e69914c568fcb28be7a1bf970236725a06a8fc66530799300181d2584a49/' ../external/node/install-node.sh # node-v12.15.0-linux-armv7l.tar.xz
      sed -i 's/linux-x64.tar.xz/linux-armv7l.tar.xz/' ../external/node/install-node.sh
      sed -i 's/https:\/\/github.com\/jcheng5\/node-centos6\/releases\/download\//https:\/\/nodejs.org\/dist\//' ../external/node/install-node.sh
      (cd .. && sudo ./external/node/install-node.sh)
      (cd .. && ./bin/npm --python="${PYTHON}" install --no-optional)
      (cd .. && ./bin/npm --python="${PYTHON}" rebuild)
      sudo make install
      ```
1. Configure shiny-server
      ```
      cd
      sudo ln -s /usr/local/shiny-server/bin/shiny-server /usr/bin/shiny-server
      sudo useradd -r -m shiny
      sudo mkdir -p /var/log/shiny-server
      sudo mkdir -p /srv/shiny-server
      sudo mkdir -p /var/lib/shiny-server
      sudo chown shiny /var/log/shiny-server
      sudo mkdir -p /etc/shiny-server
      cd
      sudo wget \
      https://raw.github.com/rstudio/shiny-server/master/config/upstart/shiny-server.conf \
      -O /etc/init/shiny-server.conf
      sudo chmod 777 -R /srv      
      ```
1. Configure shiny-server auto start.
      1. Run `sudo nano /lib/systemd/system/shiny-server.service`.
      1. Add the following lines and save.
         ```
             #!/usr/bin/env bash
             [Unit]
             Description=ShinyServer
             [Service]
             Type=simple
             ExecStart=/usr/bin/shiny-server
             Restart=always
             # Environment="LANG=en_US.UTF-8"
             ExecReload=/bin/kill -HUP $MAINPID
             ExecStopPost=/bin/sleep 5
             RestartSec=1
             [Install]
             WantedBy=multi-user.target

         ```
      1. Start the server.
         ```
         sudo chown shiny /lib/systemd/system/shiny-server.service
         sudo systemctl daemon-reload
         sudo systemctl enable shiny-server
         sudo systemctl start shiny-server
         ```
      1. Set up user permission.
         ```
         sudo groupadd shiny-apps
         sudo usermod -aG shiny-apps pi
         sudo usermod -aG shiny-apps shiny
         cd /srv/shiny-server
         sudo chown -R pi:shiny-apps .
         sudo chmod g+w .
         sudo chmod g+s .
         ```

### Install RStudio Server
1. Install Rust and Cargo. 
      1. Run `sudo curl https://sh.rustup.rs -sSf | sh` and select ''Proceed with installation'' (1).
      1. Run `source $HOME/.cargo/env`.
1. Install sentry-cli. Run the following commands.
      ```
      sudo git clone https://github.com/getsentry/sentry-cli.git
      sudo chown -R pi sentry-cli
      cd sentry-cli # Took 55m 45s
      cargo build
      cd
      sudo rm -rf sentry-cli
      ```
1. Clone RStudio from GitHub.
      1. Run `sudo git clone https://github.com/rstudio/rstudio.git`.
      1. To avoid installing crashpad, run `sudo nano /home/pi/rstudio/dependencies/common/install-common`.
      1. Comment the following lines and save.
      ```
      # ./install-crashpad
      # sudo ./install-crashpad
      ```
1. Install system dependencies.
      ```
      sudo su
      git clone https://github.com/raspberrypi-ui/pam
      cd pam/
      ./configure
      make
      make install
      cd rstudio/dependencies/common      
      ./install-common --exclude-qt-sdk
      ```
1. Configure Java.
      ```
      java -Xms1800m
      ```
1. Install gwt compiler.
      1. Run the following commands.
         ```
         cd /home/pi/rstudio
         wget http://dl.google.com/closure-compiler/compiler-latest.zip
         unzip compiler-latest.zip
         ```
      1. Replace all files if prompted.
      1. Run the following commands.
         ```
         rm COPYING README.md compiler-latest.zip
         mv closure-compiler-v20*.jar /home/pi/rstudio/src/gwt/tools/compiler/compiler.jar
         ```
1. Install RStudio
   ```
   mkdir build
   cd build
   cmake .. -DRSTUDIO_TARGET=Server -DCMAKE_BUILD_TYPE=Release
   make install
   ```

### Setting Up Dynamic DNS
This step is needed if you don't have a static public IP (most residential Internet connections does not have a static IP). As a solution, a dynamic DNS can be set. No-IP is used here (https://www.noip.com).
1. Setup an account with No-IP.
1. Select a domain name of choice.
1. Run the following commands on the Raspberry Pi.
      ```
      mkdir /home/pi/noip
      cd /home/pi/noip
      wget https://www.noip.com/client/linux/noip-duc-linux.tar.gz
      tar vzxf noip-duc-linux.tar.gz
      cd noip-2.1.9-1
      sudo make
      sudo make install
      ```
1. Login with your No-IP account username and password.
1. 
