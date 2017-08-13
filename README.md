## Raspberry Pi 3 + Pi-Hole + OpenVPN & DNSCrypt

My own guide to use PI 3 with some good programs. <br />
Currently I'm using this [Pi 3 B starter kit](https://www.amazon.com/Raspberry-Essentials-Kit-board-Connectivity/dp/B01LWVVMUI/ref=sr_1_4?ie=UTF8&qid=1502666099&sr=8-4&keywords=raspberry+pi+3+starter+kit). <br />

* The benefit is that you not need any external software like adblockers anymore (maybe only for cosmetic blocking). <br />
* All external devices you plugin onto your Router getting automatically the adblocker lists too, which means you not need to root your device (because efficient adblockers always requiring root or some kind of tunnel which drains your battery).<br />
* OpenVPN and DNSCrypt are included for your maximum security which avoids e.g. DNS leaks.
* Noob friendly setup instructions.


### Clean system installation

* Download [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/) from Raspberrypi.org and install it onto your microSD card. I use [SD Card Formatter v5.0](https://www.sdcard.org/downloads/formatter_4/) to format the microSD card and [Download USB Image Tool 1.74](http://www.alexpage.de/usb-image-tool/download/) (as alternative you can use [Etcher](https://etcher.io/) to install Raspbian Jessie Lite onto it.


* Optimize Raspberry Pi via `sudo raspi-config`
* Select 2 `Change User Password` to change the default password.
* Select 3 `Boot Options` -> `B1 Desktop` / CLI -> B2 `Console Autologin`
* Select 5 `Interfacing Options` -> P2 `SSH` -> `Yes`
* Select 7 `Advanced Options` -> A3 `Memory Split` -> Enter `16`
* Update Raspbian and the Linux kernel.

```
sudo apt update && sudo apt -y upgrade
sudo apt install -y rpi-update
sudo rpi-update
```

* Reboot your Raspberry Pi via `sudo reboot`


### Install OpenVPN

If the Raspberry Pi is behind a router (NAT) you have to configure port forwarding. The default OpenVPN port is 1194 (UDP), I recommend to use a different port e.g. 11920. Also configure a static local IP address for the Raspberry Pi. Change the Router DNS to the Pi hole given one.

* Install OpenVPN server using the [PiVPN](http://www.pivpn.io/) installer script.

`curl -L https://install.pivpn.io | bash` <br />



### Setup OpenVPN


* Find the IP of the tun0 interface via `ifconfig tun0 | grep 'inet addr'`
* This e.g. returns `inet addr:10.8.0.1  P-t-P:10.8.0.1  Mask:255.255.255.0`
* Edit the file */etc/openvpn/server.conf* via `sudo nano /etc/openvpn/server.conf`
* Modify push `"dhcp-option DNS 8.8.8.8"` to push `"dhcp-option DNS 10.8.0.1"`
* Close and save the file with *Ctrl+X*, enter *y*, enter.
* Restart OpenVPN via `sudo systemctl restart openvpn`.



### Install Pi-Hole

* Edit */etc/dnsmasq.conf*.
* Modify `#listen-address=` to: `listen-address=127.0.0.1, 192.168.xxx.xxx, 10.8.0.1`.
* Replace the second IP with your Raspberry Pi local network IP and the third IP is the *tun0* interface.
* Restart DNSMasq via `sudo systemctl restart dnsmasq`.



### Install DNSCrypt

* Install necessary system packages.

```bash
sudo apt -y install build-essential tcpdump dnsutils libsodium-dev
sudo apt -y install locate bash-completion libsystemd-dev pkg-config
```

* Build DNSCrypt from source.

```bash
mkdir -p dnsproxy
cd dnsproxy
wget https://download.dnscrypt.org/dnscrypt-proxy/LATEST.tar.bz2
tar -xf LATEST.tar.bz2
cd dnscrypt-proxy[tab completion]
sudo ldconfig
./configure --with-systemd
make
sudo make install
```


### Setup DNSCrypt

* Add DNSCrypt user via `sudo useradd -r -d /var/dnscrypt -m -s /usr/sbin/nologin dnscrypt`.
* Take a look at the official [DNSCrypt public resolver list](https://dnscrypt.org/dnscrypt-resolvers.html) and select which resolvers you want to use. (Nearby Location, No Logging etc.)
* In this guide I will be using [dnscrypt.nl-ns0](https://dnscrypt.nl/) (DNSCrypt.nl The Netherlands).
* Copy the dnscrypt-proxy.socket adding @resolver-name (from the ‘Name’ column in the list) at the end: `cp dnscrypt-proxy.socket dnscrypt-proxy@dnscrypt.nl-ns0.socket`.
* Edit dnscrypt-proxy@dnscrypt.nl-ns0.socket with `nano dnscrypt-proxy@dnscrypt.nl-ns0.socket`.

```bash
[Unit]
Description=dnscrypt-proxy dnscrypt.nl-ns0 listening socket

[Socket]
ListenStream=127.10.10.1:11153
ListenDatagram=127.10.10.1:11153

[Install]
WantedBy=sockets.target
```

* Close and save the file with *Ctrl+X*, enter *y*, enter.
* To complete the configuration we need to edit the dnscrypt-proxy.service as well, so we cp again: `cp dnscrypt-proxy.service dnscrypt-proxy@.service`.

```bash
[Unit]
Description=DNSCrypt client proxy
Documentation=man:dnscrypt-proxy(8)
Requires=dnscrypt-proxy@%i.socket
After=network.target
Before=nss-lookup.target

[Install]
Also=dnscrypt-proxy@dnscrypt.nl-ns0.socket
WantedBy=multi-user.target

[Service]
Type=simple
NonBlocking=true

# Fill in the resolver name with one from dnscrypt-resolvers.csv file
# It is also recommended to create a dedicated system user, for example _dnscrypt
# Additional features, such as ephemeral keys and plugins, can be enabled here as well
ExecStart=/usr/local/sbin/dnscrypt-proxy \
        --resolver-name=%i \
        --user=dnscrypt

Restart=always
```

* Close and save the file with *Ctrl+X*, enter *y*, enter.
* Copy all related files to the *systemd* folder: `sudo cp ./dnscrypt-proxy@* /lib/systemd/system/`
* Enable all related files as shown:

```bash
sudo systemctl daemon-reload
sudo systemctl enable dnscrypt-proxy@.service
sudo systemctl enable dnscrypt-proxy@dnscrypt.nl-ns0.socket
```
* Reboot your Raspberry Pi -> `sudo reboot`.
* After rebooting, check the status if everything is running ok. All should be in green: `sudo systemctl status -l dnscrypt-proxy@\*`.


### Modify DNSMasq configurations

* Create a additional DNSMasq configuration file: `sudo nano /etc/dnsmasq.d/02-dnscrypt.conf`.
* Enter the following in the file: `server=127.10.10.1#11153`.
* Edit 01-pihole.conf: `sudo nano /etc/dnsmasq.d/01-pihole.conf`.
* Comment (#) out all server references, which means everything which looks like: `#server=...`.
* Edit setupVars.conf: `sudo nano /etc/pihole/setupVars.conf`.
* Comment (#) out all piholeDNS references: `#piholeDNS1=...`.
* Restart DNSMasq: `sudo systemctl restart dnsmasq`.
* Reboot Raspberry Pi the last time: `sudo reboot`.






