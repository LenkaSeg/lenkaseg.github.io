---
layout: post
title: Running a Fedora container on Arch Linux
comments: true
---

To be able to run a Pagure instance locally I need to run it on Fedora. But 
I want to keep Arch Linux on my computer. Thus I will make a Fedora container 
from an Arch Linux host.

There is this article: [Container technologies in Fedora: systemd-nspawn](https://fedoramagazine.org/container-technologies-fedora-systemd-nspawn/)
in Fedora magazine. I think systemd-nspawn is what I could use.

There is also this gist repo: [Bootstrap systemd-nspawn Fedora container from an Arch
Linux host](https://gist.github.com/jdnavarro/65c8284cc67a6ae2c5ee).
I think I could use that too.

`yaourt -Sy yum`

I added bootstrap repos to `/etc/yum.repos.d/boot.repo`,
that is to create a directory yum.repos.d and a file boot.repo with this content:

~~~~
[fedora]
name=Fedora $releasever - $basearch
failovermethod=priority
#baseurl=http://download.fedoraproject.org/pub/fedora/linux/releases/$releasever/Everything/$basearch/os/
metalink=https://mirrors.fedoraproject.org/metalink?repo=fedora-$releasever&arch=$basearch
enabled=1
gpgcheck=0
repo_gpgcheck=0

[updates]
name=Fedora $releasever - $basearch - Updates
failovermethod=priority
#baseurl=http://download.fedoraproject.org/pub/fedora/linux/updates/$releasever/Everything/$basearch/
metalink=https://mirrors.fedoraproject.org/metalink?repo=updates-released-f$releasever&arch=$basearch
enabled=1
gpgcheck=0
repo_gpgcheck=0
~~~~

Bootstrap:

`yum --installroot=/var/lib/machines/fedora install systemd passwd dnf fedora-release NetworkManager`

Set root password:

`systemd-nspawn -M fedora`

`passwd`

Start and login into the container:

`machinectl start fedora`

`machinectl login fedora`

And the Fedora container is working!

Update: Or at least mostly working. There is some problem starting the machine the next time just
with `machinectl start fedora`. It results in some Permission denied stuff. Solution: start the
container like with systemd-nspawn and path to the machine:

`sudo systemd-nspawn -b -D /var/lib/machines/fedora`

Update: How to access files located in the fedora container from arch:
Just go as sudo to /var/lib/machines/fedora and all the path till the desired place and copy
from there to arch. Fairly easy. Thanks Pingou.
