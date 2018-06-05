[![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/fold_left.svg?style=social&label=Follow%20%40CHEF-KOCH)](https://twitter.com/FZeven)
[![Say Thanks!](https://img.shields.io/badge/Say%20Thanks-!-1EAEDB.svg)](https://saythanks.io/to/CHEF-KOCH)
[![Discord](https://img.shields.io/discord/418256415874875402.svg?colorA=7289da&colorB=99aab5&label=Discord&logo=discord&maxAge=60)](https://discord.me/CHEF-KOCH)
[![Tip Me via PayPal](https://img.shields.io/badge/PayPal-tip%20me-green.svg?logo=paypal)](https://www.paypal.me/nvinside)

## Raspberry Pi 3 + Pi-Hole + OpenVPN & DNSCrypt

<p align="center"> 
<img src="https://raw.githubusercontent.com/CHEF-KOCH/CK-s-Raspberry-Pi-3-config/master/Raspberry%20PI.jpg">
</p>

My own guide to use PI 3 with some good programs. <br />

Currently I'm using this [Pi 3 B+ starter kit](https://www.amazon.com/V-Kits-Raspberry-Model-Starter-LATEST/dp/B07BDRD3LP/ref=sr_1_7?s=electronics&ie=UTF8&qid=1528227681&sr=1-7&keywords=Raspberry+Pi+3+model+b%2B). <br />

* The benefit is that you won't need any external software like adblockers (uBlock, AdGuard, etc) any more. Maybe only for cosmetic filter rules. Ads getting blocked before they're getting downloaded which speedups your webpage loading.<br />
* All external devices you plugin onto your Router getting automatically the adblocker lists too, which means you not need to root your device (because efficient adblockers always requiring root or some kind of tunnel which drains your battery).<br />
* OpenVPN and DNSCrypt are included in order to encrypt your internet data traffic and DNS queries. This also will solve any DNS leaks in this case spoofing.
* The guide is beginner friendly with easy install instructions.



### Clean system installation

* Download [Raspbian Lite](https://downloads.raspberrypi.org/raspbian_lite_latest) from Raspberrypi.org and install it onto your microSD card. I use [SD Card Formatter v4.0](https://www.sdcard.org/downloads/formatter_4/) and to format the microSD card [Download USB Image Tool 1.74](http://www.alexpage.de/usb-image-tool/download/) (as alternative you can use [Etcher](https://etcher.io/)) to install Raspbian Lite onto it.


* Optimize Raspberry Pi via `sudo raspi-config`
* Select 2 `Change User Password` to change the default password.
* Select 3 `Boot Options` -> `B1 Desktop` / CLI -> B2 `Console Autologin`
* Select 5 `Interfacing Options` -> P2 `SSH` -> `Yes`
* Select 7 `Advanced Options` -> A3 `Memory Split` -> Enter `16`
* Update Raspbian and the Linux kernel.

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install -y rpi-update
sudo rpi-update
```

* Reboot your Raspberry Pi via `sudo reboot`.



### Install OpenVPN

If the Raspberry Pi is behind a router (NAT) you have to configure port forwarding. The default OpenVPN port is 1194 (UDP), I recommend to use a different port e.g. 11920. Also configure a static local IP address for the Raspberry Pi. Change the Router DNS to the Pi hole given one.

* Install OpenVPN server using the [PiVPN](http://www.pivpn.io/) installer script.

```bash
wget https://git.io/vpn -O openvpn-install.sh
chmod 755 openvpn-install.sh
sudo ./openvpn-install.sh
```



### Setup OpenVPN

* Find the IP of the tun0 interface via `ifconfig tun0 | grep 'inet addr'`.
* This e.g. returns `inet addr:10.8.0.1  P-t-P:10.8.0.1  Mask:255.255.255.0`.
* Edit the file */etc/openvpn/server.conf* via `sudo nano /etc/openvpn/server.conf`.
* Modify push `"dhcp-option DNS 8.8.8.8"` to push `"dhcp-option DNS 10.8.0.1"`.
* You can comment out all other `push "dhcp-option DNS...` entries with `#` in front of it. 
* (_optional_) Change your Port if you like to.
* Close and save the file with *Ctrl+X*, enter *y*, enter.
* Restart OpenVPN via `sudo systemctl restart openvpn`.

Your OpenVPN server.conf file must include the following lines:

```bash
push "dhcp-option DNS 127.0.0.1"
push "dhcp-option DNS 127.0.0.2"
```



### Install Pi-Hole

You can get the latest version of the Pi-Hole script including installation instructions from [here](https://github.com/pi-hole/pi-hole).

The basic command to install PI-Hole is: `sudo curl -sSL https://install.pi-hole.net | bash`.

The following **example isn't needed anymore**, but in case you have troubles ensure dnsmasq.conf is correct configured as shown here:

* Edit */etc/dnsmasq.conf* via: `sudo nano /etc/dnsmasq.conf`.
* Modify `#listen-address=` to: `listen-address=127.0.0.1, 192.168.xxx.xxx, 10.8.0.1`.
* Replace the second IP with your Raspberry Pi local network IP and the third IP is the *tun0* interface.
* Restart DNSMasq via `sudo systemctl restart dnsmasq`.



### Install DNSCrypt-proxy v2

The latest DNSCrypt-proxy releases can be found [here](https://github.com/jedisct1/dnscrypt-proxy/releases). You find the latest Server Resolver list over [here](https://github.com/jedisct1/dnscrypt-proxy/wiki/DNS-server-sources).

* Install the necessary system DNSCrypt package `cd /opt` is our dir where the files are getting dropped into.

```bash
wget https://raw.githubusercontent.com/simonclausen/dnscrypt-autoinstall/master/dnscrypt-autoinstall --no-check-certificate
chmod +x dnscrypt-autoinstall.sh
./dnscrypt-autoinstall.sh
```

After downloading the latest DNSCrypt-proxy version extract the prebuilt binary via `sudo tar -xf dnscrypt-proxy-linux_arm64-2.0.14.tar.gz`
* Rename the extracted folder: `sudo mv linux-arm64 dnscrypt-proxy`.
* Go into the dir with cd: `cd dnscrypt-proxy`
* Now create a configuration file which we're are going to use form the integrated example `sudo cp example-dnscrypt-proxy.toml dnscrypt-proxy.toml` - the .toml file is the DNSCrypt-proxy configuration file.
* Go ahead and edit the configuration file `sudo nano dnscrypt-proxy.toml`, the default port for DNS queries is 53, this is already used by the Pi-Hole default configuration, so we have to edit this to another port like _54_. The `listen addresses` line represents where the port goes over.
* You can change the rest of the configuration how you like, for example `require_dnssec` can be set to true and `listen_addresses` must be set to another port than 53, like `127.0.0.1:54`.
* The last two commands are `sudo ./dnscrypt-proxy -service install` to install the DNSCrypt-proxy service and to start the new service we're going to use `sudo ./dnscrypt-proxy -service start`.



### Setup DNSCrypt

* Add DNSCrypt user via `sudo useradd -r -d /var/dnscrypt -m -s /usr/sbin/nologin dnscrypt`.
* Take a look at the official [DNSCrypt public resolver list](https://dnscrypt.org/dnscrypt-resolvers.html) and select which resolvers you want to use. (Nearby Location, No Logging etc.)
* In this guide I will be using [dnscrypt.nl-ns0](https://dnscrypt.nl/) (DNSCrypt.nl The Netherlands).
* Copy the dnscrypt-proxy.socket adding @resolver-name (from the ‘Name’ column in the list) at the end: `cp dnscrypt-proxy.socket dnscrypt-proxy@dnscrypt.nl-ns0.socket`.
* Edit dnscrypt-proxy@dnscrypt.nl-ns0.socket with `nano dnscrypt-proxy@dnscrypt.nl-ns0.socket`.



### Modify DNSMasq configurations

* Create an additional DNSMasq configuration file: `sudo nano /etc/dnsmasq.d/02-dnscrypt.conf`. If there is already an file, use the existent one via `sudo nano /etc/dnsmasq.conf` (that's the default procedure). 
* Edit the `#listen-address=` to `listen-address=127.0.0.1, 192.168.xxx.xxx, 10.8.0.1`. The second IP (the one starting with 192.168.) is your own Rasperry Pi local network IP address while the third IP is the _tun0_ interface where OpenVPN is listening on. 
* Now we need to create another file, `sudo nano /etc/dnsmasq.d/02-dnscrypt.conf` creates the DNSCrypt-proxy configuration but since our PI-Hole doesn't know (yet) where our PI-Hole is we need to give him our server address `server=127.0.0.1#54` - ensure port 53 is not set here, give him the port we set earlier above (in this test example 54).
* Now we create and edit the Pi-Hole configuration, `sudo nano /etc/dnsmasq.d/01-pihole.conf` and comment out the pre-defined server preference `#server=...`.
* The last step is to change the default setup variables, `sudo nano /etc/pihole/setupVars.conf` we need to comment out `#PIHOLE_DNS_x`and restart our local dnsmasq server via `sudo systemctl restart dnsmasq`.


You're done! 
