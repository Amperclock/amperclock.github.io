---
layout: post
title:  "Creating a seedbox and a NAS with a RaspberryPi"
date:   2021-01-23 11:00:00 +0100
categories: howto
tags: NAS Raspberry Pi Seedbox Creation torrent vpn openvpn
---

![banner](/assets/raspberry-seedbox_banner.png)

In this post, we'll see how to create a seedbox for your torrents, and how to create a samba network share to make them available to your local network. Torrents needs users in order to work properly: if no one is seeding the file, you can't download it. So it's a good practice to seed during a certain period of time the file that you just downloaded, especially when there's only a few seeders.

A seedbox is a server that is dedicated to download and share files via a Peer-to-peer (P2P) network. Additionally, it should have a way to recover those downloaded files (in our case, the samba network share). This server is always on, which means that we don't need to remember to seed the files we just downloaded for the rest of the community, the seedbox will take care of it.

We will also see 

## 1) Necessary material
- A Raspberry pi (I'll use a 3, but a 4 is strongly recommended)
- An SD Card of at least 4GB
- A USB HDD (any size will do, but keep in mind that your downloaded files will end up on it)

For this project, a Raspberry pi 4 is way better then the 3. Although it comes with greater processing power, what really interest us is the better IO:

| Raspberry | USB | Network |
|-----------|:---:|:-------:|
| Version 3 | 2.0 | 100 Mbps|
| Version 4 | 3.0 | 1 Gps   |

If you have a optic fiber connection, using a 4 over a 3 will allow you to pass from 12,5 MB/s to 128 MB/s (theoretical). You can find more informations about the technical specifications on the raspberry [3] and [4] product pages.

[3]: https://www.raspberrypi.org/products/raspberry-pi-3-model-b/
[4]: https://www.raspberrypi.org/products/raspberry-pi-4-model-b/specifications/


## 2) Installing Raspberry Pi OS (Raspbian)

Let's start by downloading the Raspberry pi OS (previously called Raspbian): 
Go to the [download page] and choose the Raspberry Pi OS **Lite**. We don't need any graphical interface or any tools with a GUI for this project.

Once downloaded, burn it on the SD card:
{% highlight bash %}
$ dd bs=4M if=2021-01-11-raspios-buster-armhf.img of=/dev/sdX conv=fsync
{% endhighlight %}

Insert the SD card into the Raspberry, connect a screen with HDMI, and power it up by connecting the USB power cable. Once the startup sequence is finished, you are invited to log in. Use the default credentials :
{% highlight bash %}
user    pi
pass    raspberry
{% endhighlight %}

Use the rapsberry utility to change your keyboard layout (if needed) and enter your Wifi settings (more informations about the raspi-config [here]):
{% highlight bash %}
$ raspi-config
{% endhighlight %}

For security reasons, change the defaults passwords:
{% highlight bash %}
$ su   # Change user to root 
 #Enter the password of the pi user (rapberry)
$ passwd    #To change the root password
$ passwd pi #to change the pi user password
{% endhighlight %}

Don't forget to make sure that all your packages are up to date, and install any pending updates: 
{% highlight bash %}
$ apt update && apt upgrade
{% endhighlight %}

[download page]: https://www.raspberrypi.org/software/operating-systems/
[here]: https://www.raspberrypi.org/documentation/configuration/raspi-config.md

## 3) Configure and enable SSH server
An SSH server comes preinstalled on the version Lite of RaspberryPiOS. We'll add a bit of configuration:

Disable root login (for security reasons):
{% highlight bash %}
$ vim /etc/ssh/ssh_config
add     PermitRootLogin No
{% endhighlight %}

Then, enable the ssh service (previously named sshd) to have it started at boot:
{% highlight bash %}
$ systemctl enable ssh
$ systemctl status ssh #check it status
{% endhighlight %}

## 4) Storage
We can create the directory that transmission will use to store the files:
{% highlight bash %}$ mkdir /mnt/drive{% endhighlight %}

don't forget to add the proper rights so the daemon can write in it:
{% highlight bash %}
$ chown debian-transmission /mnt/drive
$ chgrp debian-transmission /mnt/drive
$ chmod -R 755 /mnt/drive
{% endhighlight %}

To simplify everything, we can edit fstab file to auto mount the USB disk on boot. First, let's get the UUID of the drive (to have more infos about the UUID, you can look at [this wiki page]).
{% highlight bash %}
$ lsblk -f
#Find your USB disk, and write down the UUID
{% endhighlight %}

Now, we need to edit the fstab to add our drive. At the end of the file, add a line according to this table:

| Option | UUID | Mount point | Filesystem | Options | Dump | Pass |
|--------||------|-------------|------|---------|------|------|
|Explanations| The UUID of your drive|The directory to mount the filesystem|The filesystem type|Leave it to default|Leave 0| Leave 2|
|Example| [MYUUID]| /mnt/drive| ext4 | Defaults | O |

{% highlight bash %}
$ vim /etc/fstab
# Add your line according to the previous table, save and quit
{% endhighlight %}

For a comprehensive explanation on every fstab options, visit [the wiki]

[this wiki page]: https://wiki.debian.org/Part-UUID#Via_UUIDs
[the wiki]: https://wiki.debian.org/fstab

## 5) Install and configure transmission-daemon:

Install the daemon with `apt`:
{% highlight bash %}apt install transmission-daemon{% endhighlight %}

And stop it:
{% highlight bash %}systemctl stop transmission-deamon{% endhighlight %}

Once it's stopped, we can start editing the json config file:
{% highlight bash %}
$ vim /etc/transmission-daemon/settings.json
Change download-dir     : /mnt/drive
Change incomplete-dir   : /mnt/drive/
Change rpc-whitelist    : add *
Change rpc-username     : "user" #Or what ever you want it to be
Change rpc-password     : "yourpassword" in plain text #Will be encrypted on restart
Change rpc-url          : /transmission/

{% endhighlight %}
All infos about every parameters are available here on the [git].

Now we can restart the daemon and enable it:
{% highlight bash %}
$ systemctl restart transmission-daemon
$ systemctl enable transmission-daemon
{% endhighlight %}

We can now access the RPC
URL : http://raspberryIP:9091/transmission/web/

**NOTE:** When the download is completed, you **need** to manually Pause it and Restart it in order to start seeding.

[git]: https://github.com/transmission/transmission/wiki/Editing-Configuration-Files


## 6) Create the network share
Install samba:
{% highlight bash %}
$ apt install samba
{% endhighlight %}

Then, create a user for authentication when accessing the share.
**NOTE:** You can skip this operation if, like me, you prefer to use the "pi" user.
{% highlight bash %}
$ useradd myuser
$ passwd myuser
{% endhighlight %}

Don't forget to add the user to the `debian-transmission` group:
{% highlight bash %}usermod -a -G debian-transmission myuser{% endhighlight %}

Now, we can add the user to the smbpasswd file to use if for authentication:
{% highlight bash %}
$ smbpasswd -a myuser
#Specify the same password, or an other one.
{% endhighlight %}

We can edit the Samba config file in order to add our directory to share:
{% highlight bash %}
$ vim /etc/samba/smb.conf
#At the end of the file, add
[shared]
comment = My fabulous torrent collection
path = /mnt/drive
public = yes
writable = no
read only = yes
guest ok = yes
{% endhighlight %}

**NOTE:** This configuration is to allow anyone (user anonymous) to read the share, but not to write in it. You can find more informations into the  [smb.conf documentation]

[smb.conf documentation]: https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html

## 7) Install and configure openvpn
At this point, you already have a working seedbox, with a network share to access your downloaded files. We can take it a step further by configuring a VPN client.

Start by installing openvpn:
{% highlight bash %}
$ apt install openvpn
{% endhighlight %}

Once installed, grab the .ovpn that your vpn provider gave you (in my case [nordvpn]), and start openvpn with it
{% highlight bash %}
$ openvpn file.ovpn
{% endhighlight %}

Don't forget to make sure that the route is correctly added:
{% highlight bash %}
$ ip route
#0.0.0.0/1 via 10.7.0.1 dev tun0 
{% endhighlight %}

To make is more convenient, we'll configure Openvpn to automatically start the VPN tunnel at boot. According to this [openvpn's documentation]:
When openvpn is installed from *"DEB package on Linux, the installer will set up an initscript. When executed, the initscript will scan for .conf configuration files in /etc/openvpn, and if found, will start up a separate OpenVPN daemon for each file."*

Edit the openvpn configuration file:
{% highlight bash %}
$ /etc/default/openvpn
#Uncomment the line AUTOSTART="all"
{% endhighlight %}

Copy your .ovpn file into the openvpn's directory:
{% highlight bash %}
$ cp /where/ever/is/you/file.ovpn /etc/openvpn
#And change the name of it to client.conf
$ mv /etc/openvpn/yourfile.ovpn /etc/openvpn/client.conf
{% endhighlight %}

This step is only if you need to enter your credentials when connecting to the vpn server:
{% highlight bash %}
$ vim /etc/openvpn/yourfile.conf
#Add ".creds" at the end of the line auth-user-pass
#Save and quit
$ vim /etc/openvpn/.creds
#Write the username on the first line
#and the password on the second line
#Save and quit
$ chmod 400 .creds
#To only allow root to read the file
{% endhighlight %}

Enable openvpn to have it started on boot:
{% highlight bash %}
systemctl enable openvpn
{% endhighlight %}

And set a delay before the transmission-daemon starts (in order for the VPN tunnel to be turned on):
{% highlight bash %}
$ vim /etc/systemd/system/multi-user.target.wants
#After the [Service], add :
ExecStartPre=/bin/sleep 30
{% endhighlight %}

At this point the seedbox is ready. Just restart the Raspberry Pi and check that you can access to the web interface.  

[openvpn's documentation]: https://openvpn.net/community-resources/configuring-openvpn-to-run-automatically-on-system-startup/

[nordvpn]: https://nordvpn.com/fr/ovpn/