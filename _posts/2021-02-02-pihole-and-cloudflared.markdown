---
layout: post
title:  "Removing adds with piHole and securing DNS requests with DoH"
date:   2021-02-02 14:30:00 +0100
categories: howto
tags: DNS Raspberry Pi piHole Cloudflare cloudflared DoH DNS-Over-HTTPS
---

![banner](/assets/pihole_cloudflared.png)

DNS Is one of those things that is use without any thought by normal users. Being able to remember websites names instead of their IP addresses is a must when it comes to browsing easily the web. The problem is that DNS was created more than [35 years ago], with different priorities in mind. To this day, DNS (by default) remains unencrypted: you can clearly see the resolving requests made by a user by sniffing the traffic. We'll see how the introduction of the *relatively* recent cloudflare's DNS servers and the DNS-Over-HTTP (DoH) protocol can help us use DNS safely.

Oh, and when it comes to piHole, I think we can all agreed that ads are annoying. Tied to the aging buisness model of the internet, they are everywhere and are constantly plaguing my browsing and viewing experience. So we'll take care if that too.

![picture](/assets/articles/wireshark_dns.png)

[35 years ago]: https://en.wikipedia.org/wiki/Domain_Name_System

## 1) Necessary material
- A Raspberry pi (I'll use a 3, but a Zero is more tailored for this kind of application)
- An SD Card of at least 4GB

piHole is usually installed on a Raspberry Pi Zero thanks to the fact that it is lightweight. Because I don't have one *yet*, I'll be installing it on my old Raspberry pi 3.

## 2) Installing Raspberry Pi OS (Raspbian)

I'll copy/paste this part from my previous post. Yes, I'm that lazy.

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

Use the rapsberry utility to change your keyboard layout if needed (more informations about the raspi-config [here]):
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

Change your hostname in order to better identify the pi on your network:
{% highlight bash %}
$ hostnamectl set-hostname piDNS
{% endhighlight %}

[download page]: https://www.raspberrypi.org/software/operating-systems/
[here]: https://www.raspberrypi.org/documentation/configuration/raspi-config.md

## 3) Installing piHole

The installation of piHole is pretty straight forward, thanks to the scrip they provide. To run it, simply execute:
{% highlight bash %}
$ curl -sSL https://install.pi-hole.net | bash
{% endhighlight %}

Then, just keep following the installation Wizard. Nothing's too complicated to understand. I've changed my privacy option to all anonymize; I don't want to log the browsing activity of my users, and it's much safer is someone succeed to hack into the piHole, they won't have much to see.

At the end of the installation process, you will receive a temporary password to access to the administration page of the piHole. Don't forget to change it with:
{% highlight bash %}
$ pihole -a -p
{% endhighlight %}

I *STRONGLY* recommend you to setup and SSL certificate and to have some HTTPS, or somebody could retrieve your password by sniffing the network.
Everything is explained [here].

Now, you can access the piHole's web interface by going to `http://[piHoleIP]/admin` (or `https://[piHoleIP]/admin` if you've followed the recommendation).

To use it, connect to you router (or anything to your DHCP), and change the DNS server address to the one of you piHole. You can also go directly to you device, and change the DNS configuration to use your piHole.

**SIDENOTE**: Some of the most brilliant router manufacturers \*cough\* **netgear for exemple** will *still* serve their own IP address as DNS in the DHCP answer, and querry the piHole themselves, making an unnecessary hop (and raising privacy concern).

![whathehellnetgear](/assets/articles/whathehellnetgear.png)

The solution to this is to use your own DHCP server, and serve whatever DNS ip you want, like the one from the piHole **OR** change the DNS settings on each of your devices.

[here]: https://discourse.pi-hole.net/t/enabling-https-for-your-pi-hole-web-interface/5771

## 4) Installing cloudflared to enable DoH

Alright, so now that we got rid of the ads, let's encrypt our DNS requests, so our Internet Provider wont eavesdrop the domains we looking up.

**NOTE**: All the following commands are as `root`. We gonna use nano since vim is not installed by default on raspberryPiOS

We'll be using a tool created by CloudFlare (in addition to using their excellent DNS servers). Let's download and install it:
{% highlight bash %}
$ wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-arm.tgz
$ tar -xvzf cloudflared-stable-linux-arm.tgz
$ cp ./cloudflared /usr/local/bin
$ chmod +x /usr/local/bin/cloudflared
$ cloudflared -v
{% endhighlight %}

We'll create user for the daemon to run:
{% highlight bash %}
$ useradd -s /usr/sbin/nologin -r -M cloudflared
{% endhighlight %}

Then modify the configuration file of cloudflared. You can change the --upstream by any DNS server that supports DoH:
{% highlight bash %}
$ nano /etc/default/cloudflared

#Add:
CLOUDFLARED_OPTS=--port 5053 --upstream https://1.1.1.1/dns-query --upstream https://1.0.0.1/dns-query
{% endhighlight %}

Update the permissions for the configuration file and cloudflared binary to allow access for the cloudflared user:
{% highlight bash %}
$ chown cloudflared:cloudflared /etc/default/cloudflared
$ chown cloudflared:cloudflared /usr/local/bin/cloudflared
{% endhighlight %}

Then create the systemd script by copying the following into /etc/systemd/system/cloudflared.service. This will control the running of the service and allow it to run on startup:
{% highlight bash %}
$ nano /etc/systemd/system/cloudflared.service

#Add:
[Unit]
Description=cloudflared DNS over HTTPS proxy
After=syslog.target network-online.target

[Service]
Type=simple
User=cloudflared
EnvironmentFile=/etc/default/cloudflared
ExecStart=/usr/local/bin/cloudflared proxy-dns $CLOUDFLARED_OPTS
Restart=on-failure
RestartSec=10
KillMode=process

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Enable the systemd service to run on startup, then start the service, check its status and try it:
{% highlight bash %}
$ systemctl enable cloudflared
$ systemctl start cloudflared
$ systemctl status cloudflared

dig @127.0.0.1 -p 5053 amperclock.com
{% endhighlight %}

{% highlight text %}
; <<>> DiG 9.11.5-P4-5.1+deb10u2-Raspbian <<>> @127.0.0.1 -p 5053 amperclock.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53310
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;amperclock.com.			IN	A

;; AUTHORITY SECTION:
amperclock.com.		300	IN	SOA	dns108.ovh.net. tech.ovh.net. 2021020206 86400 3600 3600000 300

;; Query time: 201 msec
;; SERVER: 127.0.0.1#5053(127.0.0.1)
;; WHEN: Wed Feb 03 22:00:34 GMT 2021
;; MSG SIZE  rcvd: 119
{% endhighlight %}

Success ! We now have a local DNS server that transmit our DNS queries via HTTPS and answering on the port 5053. We can now configure ou piHole to use it as an Upstream DNS server. Got the **web interface**, **log-in**, then go to **settings**, and to the **DNS** tab. Now, uncheck any DNS server that you previously had, and add the custom one `127.0.0.1#5053`. Don't forget to save.

![piholeupstream](/assets/articles/piholeupstream.png)

But we still need to make sure that our `cloudflared` service is up-to-date. Since we didn't installed it via our package manager, we need to update it manually.

For that, just run the following commands:
{% highlight bash %}
wget https://bin.equinox.io/c/VdrWdbjqyF/cloudflared-stable-linux-arm.tgz
tar -xvzf cloudflared-stable-linux-arm.tgz
$ systemctl stop cloudflared
$ cp ./cloudflared /usr/local/bin
$ chmod +x /usr/local/bin/cloudflared
$ systemctl start cloudflared
cloudflared -v
$ systemctl status cloudflared
{% endhighlight %}

You can also set-up automatic update by following the [official piHole documentation].

[official piHole documentation]: https://docs.pi-hole.net/guides/dns/cloudflared/#automating-cloudflared-updates