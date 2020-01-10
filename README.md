# RaspberryPi-Air-Quality-Display
Simple Setup to build a raspberry pi based air quality monitor

## For now this is just a copy of [Xose Pérez](https://github.com/xoseperez)'s [Gist](https://gist.github.com/xoseperez/e23334910fb45b0424b35c422760cb87)

## It will be updated/modified to include the following things:

* Using a CCS811-BME280 air quality monitor for measuring [CCS811-BME280](https://github.com/DzRepo/RaspberryPI-CCS811-BME280-Google-Cloud)
* Running InfluxDB for storing historical values
* setup [InfluxDB](https://www.influxdata.com/blog/how-to-send-sensor-data-to-influxdb-from-an-arduino-uno/) to receive data from external sensors
* Running Grafana for showing pretty graphs
* Running Telegraf for collecting internet uptime and system stats like HHD. [Uptime Dashboard](https://grafana.com/grafana/dashboards/2690) For monitoring the home internet connection
* Using WS2812b LED strip to display temp, humidity and CO2 [Link](https://github.com/jgarff/rpi_ws281x)
* Trigger mobile notifications on [CO2 Thresholds](https://www.lufft.com/blog/fuenf-gruende-warum-die-ueberwachung-der-co2-konzentration-eine-gute-idee-ist/)

# Original from [here](https://gist.github.com/xoseperez/e23334910fb45b0424b35c422760cb87)

# Raspberry Pi 3 IoT Home Server

## Presentation

http://tinkerman.cat/rpi3_iot_server.pdf (Catalan)

## Get the latest image and flash the SD card

* download the latest image

```
$ wget --output-document=raspbian.img.zip  https://downloads.raspberrypi.org/raspbian_lite_latest
```

* locate the destination volume

```
$ unzip -p raspbian.img.zip | sudo dd of=/dev/mmcblk0 bs=4M conv=fsync
```

* mount the SD card
* windows users might want to install http://www.paragon-drivers.com/extfs-windows/ to read/write EXT4 partitions
* locate the `boot` and `rootfs` partitions (on bare Windows machines only `boot` partition is visible)
* enable ssh by default:

```
$ cd /media/$USER/boot
$ touch ssh
```

* configure the wifi connection (optional):

```
$ cd /media/$USER/boot
$ sudo vi wpa_supplicant.conf

country=ES
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="<ssid1>"
    psk="<pass1>"
    id_str="Home"
    priority=1
}

# More than one network configuration can be added
# This is an open network (no pass)
network={
    ssid="<ssid2>"
    key_mgmt=NONE
    id_str="Open"
    priority=2
}

```

* configure the network interfaces (optional, linux host only):

```
$ cd /media/$USER/rootfs/etc/network
$ sudo vi interfaces

auto eth0
iface eth0 inet static
    address 192.168.1.13
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8

auto wlan0
iface wlan0 inet dhcp
    wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
    address 192.168.1.200/24
#    gateway 192.168.1.1
#    dns-nameservers 8.8.8.8
```



* unmount the boot and rootfs partitions
* remove the SD card from your computer
* insert it into the RPi
* boot the RPi and wait 90 seconds
* then connect your computer and the RPi with an ethernet cable
* set your computer to use an IP in the range 192.168.1.0/24 (not the 15!)
* log to the RPi using user `pi`, password `raspberry`:

```
$ ssh -o IdentitiesOnly=yes pi@192.168.1.200
```

* from now on you can work using the eth or you can findout the IP of the wlan0 interface:

```
$ ifconfig wlan0
```

* if you are connected via wlan0, you might want to disable eth0 temporarily:

```
$ sudo ifdown eth0
```


## First steps

* Check connectivity:

```
$ ping google.com
```

* Delete routes if necessary:

```
$ sudo route del -net 0.0.0.0 gw 192.168.1.1 netmask 0.0.0.0 dev eth0
```

* Upgrade packages:

```
$ sudo apt-get update
$ sudo apt-get upgrade
```

* Update VIM editor (personal preference):

```
$ sudo apt-get install vim
$ sudo select-editor
```

* Configure options (enable I2C, SSH, expand filesystem, change hostname, localisation options, do not reboot yet)

```
$ sudo raspi-config
```

* Make us of TMPFS (strongly recommended)

```
$sudo vi /etc/fstab

(...)
tmpfs    /tmp               tmpfs   defaults,noatime,nosuid,size=30m                    0 0
tmpfs    /var/tmp           tmpfs   defaults,noatime,nosuid,size=30m                    0 0
tmpfs    /var/log           tmpfs   defaults,noatime,nosuid,mode=0755,size=30m          0 0
tmpfs    /var/spool/mqueue  tmpfs   defaults,noatime,nosuid,mode=0700,gid=1001,size=30m 0 0
```

* Create a new user (recommended)

```
$ sudo rm /etc/ssh/ssh_host* 
$ sudo ssh-keygen -A
$ sudo adduser xose
$ sudo adduser xose sudo
$ sudo reboot
...
$ sudo deluser --remove-home pi
```

* ...or change the pass for the `pi` user and reboot:

```
$ passwd
$ sudo reboot
```

## Mosquitto

* Add the repository and install Mosquitto:

```
$ wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key
$ sudo apt-key add mosquitto-repo.gpg.key
$ cd /etc/apt/sources.list.d/
$ sudo wget http://repo.mosquitto.org/debian/mosquitto-stretch.list
$ cd
$ sudo apt-get update
$ sudo apt-get install mosquitto mosquitto-clients
```

* Secure the connection:

```
$ sudo mosquitto_passwd -c /etc/mosquitto/passwd <user>
$ sudo vi /etc/mosquitto/conf.d/default.conf

user mosquitto
allow_anonymous false
password_file /etc/mosquitto/passwd
```

* Use bonjour/zeroconf to advertise the service (optional):

```
$ sudo apt-get install avahi-utils
$ sudo vi /etc/avahi/services/mosquitto.service

<?xml version="1.0" standalone='no'?><!--*-nxml-*-->
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
 <name replace-wildcards="yes">Mosquitto MQTT server on %h</name>
  <service>
   <type>_mqtt._tcp</type>
   <port>1883</port>
  </service>
</service-group>
```

* Autostart service

```
$ sudo systemctl stop mosquitto
$ sudo update-rc.d mosquitto remove
$ sudo rm /etc/init.d/mosquitto
$ sudo vi /etc/systemd/system/mosquitto.service

[Unit]
Description=Mosquitto MQTT Broker daemon  
ConditionPathExists=/etc/mosquitto/mosquitto.conf  
AfteR=network.target
Requires=network.target

[Service]
Type=simple  
ExecStart=/usr/sbin/mosquitto -c /etc/mosquitto/mosquitto.conf   
Restart=always

[Install]
WantedBy=multi-user.target  

$ sudo systemctl daemon-reload
$ sudo systemctl enable mosquitto
$ sudo systemctl start mosquitto.service
```

## InfluxDB

* Add the repository

```
$ curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
$ source /etc/os-release
$ test $VERSION_ID = "7" && echo "deb https://repos.influxdata.com/debian wheezy stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
$ test $VERSION_ID = "8" && echo "deb https://repos.influxdata.com/debian jessie stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
$ test $VERSION_ID = "9" && echo "deb https://repos.influxdata.com/debian stretch stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
$ sudo apt-get update
```

* Install

```
$ sudo apt-get install libfontconfig1
$ sudo apt-get -f install
$ sudo apt-get install influxdb
```

* Start service

```
$ sudo service influxdb start
```

* Configure console to use proper timestamps

```
$ vi ~/.bash_aliases

(...)
alias idb='influx -precision "rfc3339"'
```

* Install Telegraf (optional, system metrics on InfluxDB)

```
$ sudo apt-get install telegraf
$ sudo service telegraf start
```

* Check the database

```
$ source ~/.bash_aliases
$ idb
> SHOW DATABASES
name: databases
name
----
_internal
telegraf
> USE telegraf
Using database telegraf
> SHOW MEASUREMENTS
name: measurements
name
----
cpu
disk
diskio
kernel
mem
processes
swap
system
> SELECT * FROM disk;
name: disk
time                 device    free        fstype host        inodes_free inodes_total inodes_used mode path  total       used       used_percent
----                 ------    ----        ------ ----        ----------- ------------ ----------- ---- ----  -----       ----       ------------
2018-02-15T16:00:00Z mmcblk0p1 21017600    vfat   raspberrypi 0           0            0           rw   /boot 42856960    21839360   50.95872409055612
2018-02-15T16:00:00Z root      28520734720 ext4   raspberrypi 1811742     1881152      69410       rw   /     31349534720 1522057216 5.066297497391156
(...)
```

* Define retention policies (example)

```
> USE telegraf
> ALTER RETENTION POLICY "autogen" ON "telegraf" DURATION 4w REPLICATION 1 DEFAULT;
> CREATE RETENTION POLICY "forever" ON "telegraf" DURATION INF REPLICATION 1;
> SHOW RETENTION POLICIES;
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 672h0m0s 168h0m0s           1        true
forever 0s       168h0m0s           1        false
```

* Define continuous queries (example)

```
> CREATE CONTINUOUS QUERY "disk_free_1h" ON "telegraf" RESAMPLE EVERY 1h BEGIN SELECT min("free") INTO "forever"."disk" FROM "autogen"."disk" WHERE device = 'root' GROUP BY time(1h) END
```

* Dump RPi temperature to InfluxDB (optional)

```
$ sudo apt-get install bc
$ cd 
$ mkdir -p bin
$ vi bin/cpu_temp

#!/bin/bash
cpu=$(</sys/class/thermal/thermal_zone0/temp)
hostname=`hostname`
database=telegraf
cpu=`echo "scale=2;$cpu / 1000" | bc`
curl -i -XPOST "http://localhost:8086/write?db=$database" --data-binary "temperature,host=$hostname,name=cpu,units=celsius value=$cpu"

$ chmod +x local/cpu_temp
$ crontab -e

*/1 * * * * $HOME/bin/cpu_temp
```

## Node-RED

* Install & start

```
bash <(curl -sL https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/update-nodejs-and-nodered)
node-red-start
```

* Security (uncomment adminAuth part in settings.js replacing username and password)

```
cd ~/.node-red
sudo npm install -g node-red-admin
node-red-admin hash-pw
vi settings.js
```

* Using PM2 to monitor the service (recommended, notice the messages):

```
sudo systemctl disable nodered.service
sudo npm install -g pm2
pm2 start node-red
pm2 save
pm2 startup
```

## Grafana

* Check the latest version visiting  https://github.com/fg2it/grafana-on-raspberry/releases

* Download it and install it

```
cd ~
wget --output-document=grafana.deb https://github.com/fg2it/grafana-on-raspberry/releases/download/v4.6.2/grafana_4.6.2_armhf.deb
sudo dpkg -i grafana.deb
sudo apt-get install -f
```

* Install plugins (optional)

```
sudo grafana-cli plugins install grafana-clock-panel
sudo grafana-cli plugins install natel-discrete-panel
sudo grafana-cli plugins install briangann-gauge-panel
sudo grafana-cli plugins install vonage-status-panel
sudo grafana-cli plugins install neocat-cal-heatmap-panel
#sudo grafana-cli plugins install natel-plotly-panel
```

* Enable service (so it autostarts)

```
sudo systemctl enable grafana-server
```

* Start service

```
sudo systemctl start grafana-server
```

* Open the web interface (`http://192.168.1.200:3000`) and configure a data source (local InfluxDB database `telegraf`) and a dashboard (example).

## Nginx

* Install Nginx

```
$ sudo usermod -a -G video $USER
$ sudo apt-get update
$ sudo apt-get install nginx
```

* Install certbot for SSL certificiates (strongly recommended)

```
$ echo "deb http://ftp.debian.org/debian jessie-backports main contrib" | sudo tee /etc/apt/sources.list.d/backports.list
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 8B48AD6246925553
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7638D0442B90D010
$ sudo apt-get update
$ sudo apt-get install certbot -t jessie-backports
```

* Install certificates (recommended)

```
$ sudo certbot certonly --webroot -w /var/www/html -d <domain_name>
$ sudo certbot certificates
```

* Autorenew certificates (recommended)

```
$ sudo crontab -e

@weekly /usr/bin/certbot renew --quiet --no-self-upgrade -w /var/www/html/
@daily service nginx reload
```

* Configure service

```
$ sudo vi /etc/nginx/sites-available/proxy

server {

    # general server parameters 
    listen                      443;
    server_name                 santpol.tinkerman.cat;
    access_log                  /var/log/nginx/<domain_name>.access.log;
    #error_log                   /var/log/nginx/error.log debug;

    # SSL configuration
    ssl                         on;
    ssl_certificate             /etc/letsencrypt/live/<domain_name>/fullchain.pem;
    ssl_certificate_key         /etc/letsencrypt/live/<domain_name>/privkey.pem;
    ssl_session_cache           builtin:1000  shared:SSL:10m;
    ssl_protocols               TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers                 HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
    ssl_prefer_server_ciphers   on;
    
    location /node-red/ {

      # header pass through configuration
      proxy_set_header        Host $host;      
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;

      # extra debug headers      
      add_header              X-Forwarded-For $proxy_add_x_forwarded_for;

      # actual proxying configuration
      proxy_ssl_session_reuse on;
      proxy_pass              http://localhost:1880/;
      proxy_redirect          default;
      proxy_read_timeout      90;
   
    }
    
    location /grafana/ {

      # header pass through configuration
      proxy_set_header        Host $host;      
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;

      # extra debug headers      
      add_header              X-Forwarded-For $proxy_add_x_forwarded_for;

      # actual proxying configuration
      proxy_ssl_session_reuse on;
      proxy_pass              http://localhost:3000/;
      proxy_redirect          default;
      proxy_read_timeout      90;

    }

}

$ sudo ln -s /etc/nginx/sites-available/proxy /etc/nginx/sites-enabled/proxy
```

* Check and reload changes

```
$ sudo nginx -t
$ sudo service nginx reload
```

* Use bonjour/zeroconf to advertise the service (optional):

```
$ sudo vi /etc/avahi/services/nginx.service

<?xml version="1.0" standalone='no'?><!--*-nxml-*-->
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">
<service-group>
 <name replace-wildcards="yes">Nginx server on %h</name>
  <service>
   <type>_http._tcp</type>
   <port>80</port>
  </service>
  <service>
   <type>_https._tcp</type>
   <port>443</port>
  </service>
</service-group>
```
