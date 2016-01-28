Gateway Software Setup
======================

These instructions should be shared between gateway platforms running Debian.


Software Setup
--------------


1. Experimental: Install MQTT:

        wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key
        sudo apt-key add mosquitto-repo.gpg.key
        cd /etc/apt/sources.list.d/
        sudo wget http://repo.mosquitto.org/debian/mosquitto-jessie.list
        apt-get update
        apt-get install mosquitto

11. Worked on BBB: Install libwebsockets for MQTT.

        wget http://git.warmcat.com/cgi-bin/cgit/libwebsockets/snapshot/libwebsockets-1.5-chrome47-firefox41.tar.gz
        tar xf libwebsockets-1.5-chrome47-firefox41.tar.gz
        cd libwebsockets-1.5-chrome47-firefox41
        mkdir build
        cd build
        cmake ..
        make
        sudo make install

12. Worked on BBB: Install MQTT.

        wget http://mosquitto.org/files/source/mosquitto-1.4.5.tar.gz
        tar xf mosquitto-1.4.5.tar.gz
        cd mosquitto-1.4.5
        sed -i s/WITH_WEBSOCKETS:=no/WITH_WEBSOCKETS:=yes/g config.mk
        make
        sudo make install
        sudo ldconfig
        sudo cp service/upstart/mosquitto.conf /etc/init/
        sudo cp /etc/mosquitto/mosquitto.conf.example /etc/mosquitto/mosquitto.conf
        sudo sed -i 's/#listener/listener 9001\nprotocol websockets/g' /etc/mosquitto/mosquitto.conf
        sudo sed -i 's/#port 1883/listener 1883/g' /etc/mosquitto/mosquitto.conf

12. Worked on BBB: Setup MQTT. Create `/etc/systemd/system/mosquitto.service` with the following contents:

        [Unit]
        Description=Mosquitto MQTT Broker daemon
        ConditionPathExists=/etc/mosquitto/mosquitto.conf
        Requires=network.target

        [Service]
        Type=simple
        User=mosquitto
        ExecStartPre=/bin/rm -f /var/run/mosquitto.pid
        ExecStart=/usr/local/sbin/mosquitto -v -c /etc/mosquitto/mosquitto.conf
        ExecReload=/bin/kill -HUP $MAINPID
        PIDFile=/var/run/mosquitto.pid
        Restart=on-failure

        [Install]
        WantedBy=multi-user.target

    Then:

        sudo useradd mosquitto
        sudo systemctl daemon-reload
        sudo systemctl enable mosquitto

13. Install Python 3.5.

        wget https://www.python.org/ftp/python/3.5.1/Python-3.5.1.tgz
        tar xf Python-3.5.1.tgz
        cd Python-3.5.1
        ./configure
        make sudo make install

14. Install some Python dependencies.

        sudo pip install paho-mqtt
        sudo pip install websocket
        sudo pip3.5 install hbmqtt
        sudo pip3.5 install websockets

13. Install Node.js.

        curl -sL https://deb.nodesource.com/setup_5.x | sudo -E bash -
        sudo apt-get install -y nodejs

14. Enable Node privileged access to BLE so it doesn't need sudo.

        sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)

17. Get rid of locale warnings

        sudo apt-get install locales
        sudo locale-gen en_us.UTF-8
        sudo dpkg-reconfigure locales
            Select "en_US.UTF-8 UTF-8" from list and hit enter
            Select "en_US.UTF-8" and hit enter

18. Add user to dialout (for serial permissions)

        usermod -a -G dialout debian

19. Clean up home directory

        rm -rf /home/debian/*

20. Clone gateway github repository

        git clone https://github.com/lab11/gateway.git
        git clone https://github.com/terraswarm/urban-heartbeat-kit.git

21. Install gateway dependencies.

        pushd gateway/software/ble-gateway-publish && npm i && popd
        pushd gateway/software/ble-gateway-server && npm i && popd
        pushd gateway/software/adv-gateway-ip && npm i && popd
        pushd urban-heartbeat-kit/examples && npm i && popd
        pushd urban-heartbeat-kit/discover/gateway-ssdp && npm i && popd

22. Setup ble-gateway to start on boot.

        sudo cp gateway/systemd/* /etc/systemd/system/
        sudo cp urban-heartbeat-kit/systemd/* /etc/systemd/system/
        sudo systemctl daemon-reload
        sudo systemctl disable lighttpd
        sudo systemctl enable ble-gateway-publish
        sudo systemctl enable ble-gateway-server
        sudo systemctl enable adv-gateway-ip
        sudo systemctl enable gateway-gdp-publish
        sudo systemctl enable adv-gateway-ssdp


Optional: Install Node-RED
--------------------------

2. Install Node-RED.

        sudo npm install -g --unsafe-perm  node-red

    At this point, Node-RED is installed and ready to run. However,
    we want to run Node-RED as a system service so that it starts
    on boot and restarts automatically after encountering issues.

5. Install BBB-specific Node-RED libraries. (GPIO pin access, etc.)

        mkdir -p ~/.node-red
        cd ~/.node-red
        npm install node-red-node-beaglebone

7. Configure Node-RED to run as a service. Create the file
`/etc/systemd/system/node-red.service` with the following contents:

        [Unit]
        Description=Node-RED

        [Service]
        ExecStart=/usr/bin/node-red-pi --max-old-space-size=128 --userDir /home/debian/.node-red
        Restart=always
        StandardOutput=syslog
        StandardError=syslog
        SyslogIdentifier=node-red
        User=debian

        [Install]
        WantedBy=multi-user.target

    Then:

        sudo systemctl daemon-reload
        sudo systemctl enable node-red

6. Start the Node-RED service.

        sudo systemctl start node-red

    Test that Node-RED is running by navigating to `http://<Beaglebone IP>:1880`.

9. You can password-protect your Node-RED instance, add HTTPS support,
and make other configuration changes by modifying
`/home/debian/.node-red/settings.js` and restarting the service with
`sudo systemctl restart node-red`.


Audio
-----

1. Install the microphone tools.

        sudo apt-get install alsa-utils


GDP
---

1. Get the source repo.

        git clone https://repo.eecs.berkeley.edu/git/projects/swarmlab/gdp.git

2. Prep the system.

        cd gdp
        ./adm/gdp-setup.sh

3. Build the client sources.

        ./deb-pkg/package-client.sh 0.3-1
        ./lang/python/deb-pkg/package.sh 0.3-1

4. Install the client.

        sudo dpkg -i gdp-client_0.3-1_armhf.deb
        sudo dpkg -i python-gdp_0.3-1_all.deb
        sudo apt-get -f install



