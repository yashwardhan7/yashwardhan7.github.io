---
layout: post
title:  "Setting up a new RaspberryPi"
date:   2016-06-20
categories: raspberrypi headless
excerpt_separator: <!--more-->
---
I have a few headless RaspberryPi setup at my home running various tasks.
Here are the minimal setup steps I follow when setting up a new Pi:
<!--more-->

1. [**Download** a Raspbian image](https://www.raspberrypi.org/downloads/raspbian/)
  * get the Lite version to set-up headless
2. Get an 8+ GB microSD card and **flash** the Raspbian image on it
  * Windows: use [Win32DiskImager](https://www.raspberrypi.org/documentation/installation/installing-images/windows.md)
  * Mac: use [dd](https://www.raspberrypi.org/documentation/installation/installing-images/mac.md) or [ApplePi-Baker](http://www.tweaking4all.com/hardware/raspberry-pi/macosx-apple-pi-baker/)
  * Linux: use [dd](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md)
3. Insert the microSD card in the RaspberryPi, connect ethernet cable to it and **boot**.
4. Figure out Pi's IP address and **SSH** to it (look at [router's browser frontend](http://192.168.1.1) to find the IP address)
  * `ssh pi@ipadress` and use password `raspberry`
5. **Setup wifi** (replace `wifi_name` and `wifi_password` in below command with correct values)
  * `echo -e "network={\n\tssid=\"wifi_name\"\n\tpsk=\"wifi_password\"\n}" | sudo tee --append /etc/wpa_supplicant/wpa_supplicant.conf`
6. Run `passwd` and **change the default password** for user `pi`
7. **Change hostname** (replace `pi_hostname` in below commands with correct value)
  * `echo pi_hostname | sudo tee --append /etc/hostname`
  * `sudo sed -i -- 's/raspberrypi/pi_hostname/g' /etc/hosts`
  * `sudo /etc/init.d/hostname.sh`
8. Upgrade to **latest OS**
  * `sudo apt-get update && sudo apt-get upgrade`
9. `sudo raspi-config` and
  * Expand filesystem to use the full SD card
  * Configure pi to boot into terminal
  * Change internationalization options: locale, timezone, keyboard layout, wifi country
10. Enable watchdog to **auto-reboot on hang**
  * `sudo apt-get install watchdog`
  * `sudo modprobe bcm2708_wdog`
  * `echo bcm2708_wdog | sudo tee --append /etc/modules`
  * `sudo update-rc.d watchdog defaults`
  * `sudo sed -i -- $'s/#max-load-1\t/max-load-1\t/g' /etc/watchdog.conf`
  * `sudo sed -i -- 's/#watchdog-device/watchdog-device/g' /etc/watchdog.conf`
  * `sudo service watchdog start`
11. Enable fail2ban to **guard against network attacks**
  * `sudo apt-get install fail2ban`
  * `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`
  * `sudo service fail2ban restart`
12. Reboot
  * `sudo shutdown -r now`
