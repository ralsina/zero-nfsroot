# Making Raspberry Pi Zero boot without SD card

<span class="toc"><ul>
<li><a href="#Making-Raspberry-Pi-Zero-boot-without-SD-card" title="Making Raspberry Pi Zero boot without SD card" smoothhashscroll="">Making Raspberry Pi Zero boot without SD card</a><ul>
<li><a href="#Goals" title="Goals" smoothhashscroll="">Goals</a></li>
<li><a href="#On-the-local-network" title="On the local network" smoothhashscroll="">On the local network</a></li>
<li><a href="#On-the-host-machine" title="On the host machine" smoothhashscroll="">On the host machine</a><ul>
<li><a href="#Configure-an-ethernet-bridge" title="Configure an ethernet bridge" smoothhashscroll="">Configure an ethernet bridge</a></li>
</ul>
</li>
<li><a href="#Create-the-NFS-root" title="Create the NFS root" smoothhashscroll="">Create the NFS root</a><ul>
<li><a href="#The-Hard-Way" title="The Hard Way" smoothhashscroll="">The Hard Way</a></li>
<li><a href="#The-Easy-Way" title="The Easy Way" smoothhashscroll="">The Easy Way</a></li>
</ul>
</li>
<li><a href="#The-second-Zero" title="The second Zero" smoothhashscroll="">The second Zero</a></li>
</ul>
</li>
</ul>
</span>

This is **heavily** based on [this page](https://dev.webonomic.nl/how-to-run-or-boot-raspbian-on-a-raspberry-pi-zero-without-an-sd-card) with some tweaks to make it boot 
exactly the way I want it to and configure the network like I wanted. Also, lots of things are "reverse engineered" from [ClusterHAT](hgttp://clusterhat.com) meaning I have one and saw how it works.

**Note:** These are the working notes of how **I** did it. I could have just done it 
but taking notes is good because:

> The difference between screwing around and science is writing it down 
> -- Adam Savage


## Goals

After all this is done, we should have one or more raspberry pi zeros booting

* Off NFS without SD cards
* Using network via USB-gadget-ethernet
* With network configured to be part of the local LAN via a bridge

## On the local network

Configure a range for static IP addresses. How to do that depends on your
setup, but usually implies logging into your router and changing its DHCP
server configuration. I have the network `192.168.0.0/24` and saved
IPs `201 ... 254` for static addresses.

## On the host machine

Again, details on *how* to do these things will vary depending on your 
Linux distribution or whatever operating system you are using (I don't expect
anything that is not Linux will work)

* Install a NFS server
* Install [USBBoot](https://github.com/raspberrypi/usbboot)
* Install bridge-utils
* Connect yourself to the network via ethernet (it can probably be made to work with wifi)
* Enable IP forwarding and set your IPTables FORWARD policy to accept.

### Configure an ethernet bridge

Details may vary but we want an ethernet bridge that includes your ethernet interface.
For example, mine is `enp4s0f3u1u4` and my bridge is `br0`so:

If you are using NetworkManager, it has a GUI to configure it that works just fine.

This is the bridge to which we will also connect the zeros' ethernets so they become part 
of the LAN.

```sh
$ brctl show
bridge name           bridge id              STP enabled         interfaces
br0                   8000.a2d599084fcf      yes                 enp4s0f3u1u4
```

If all went well, you will have an IP address on the `br0` interface and not in your "real" ethernet:

```sh
$ ifconfig -a

br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.72  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 2800:810:505:2c8::1004  prefixlen 128  scopeid 0x0<global>
        inet6 fe80::ef7a:62ed:f9:5ce5  prefixlen 64  scopeid 0x20<link>
        inet6 2800:810:505:2c8:2dff:72aa:2fce:b293  prefixlen 64  scopeid 0x0<global>
        ether a2:d5:99:08:4f:cf  txqueuelen 1000  (Ethernet)
        RX packets 82794  bytes 123703379 (117.9 MiB)
        RX errors 0  dropped 178  overruns 0  frame 0
        TX packets 77001  bytes 49355088 (47.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp4s0f3u1u4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 28:ee:52:1a:8c:20  txqueuelen 1000  (Ethernet)
        RX packets 133110  bytes 125755066 (119.9 MiB)
        RX errors 0  dropped 78  overruns 0  frame 0
        TX packets 66604  bytes 10713246 (10.2 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

## Create the NFS root

There are two ways to do this, the easy way and the hard way. You choose.

### The Hard Way

We will boot a raspberry pi zero, configure it, and then turn its filesystem into
a NFS root. Since we will know **exactly** what we changed we can then use that as
the basis for just copying and modifying it for as many machines as we want, which is the easy way. So, you *could* just skip this section and do what's described in the next one.
You would be, of course, a chicken.

Hello non-feathered readers, let's do this.

* Get raspberry pi OS 32-bit light (or whatever) into a SD card. 
  Doesn't need to be a fast or large SD card, since we will only use it for a few minutes.
  
  If you will try to do this using something other than raspberry pi os then the
  details of *how* to do it will change, but the concepts should still work.

Now we are going to make some changes in the configuration of that SD card before booting
from it.

* Create empty `boot/ssh` file so it has SSH enabled on boot.
* Edit `boot/config.txt` and add this at the bottom:

```
# enable OTG
dtoverlay=dwc2
# set initramfs
initramfs initrd.img followkernel
```

* Edit `boot/cmdline.txt` and add this at the end of the 1st line:

`modules-load=dwc2,g_ether`


* Edit `etc/network/interfaces.d/usb0` to configure its `usb0` interface:

```
auto usb0
allow-hotplug usb0
iface usb0 inet static
  address 192.168.0.201/24
  gateway 192.168.0.1
  dns-nameservers 8.8.8.8 1.1.1.1 9.9.9.9
```

* Enable ssh for the future

`sudo systemctl enable ssh`

At this point we should be able to boot off this SD card, so insert the SD card into the Zero, plug the Zero into your computer via USB, plug some monitor in its HDMI interface and let's see what happens.

The first boot is pretty slow and convoluted because it will do things like resize the filesystem to use the whole SD card and whatnot. But eventually it should finish.

At some point, your computer (not the Zero!) will have a new network interface.
In my case it's called `enp4s0f3u1u1`

**NOTE:** The actual USB port on which you plug it is important, because it determines the
name of the ethernet interface to which your Zero is connected, so ... take notes.

We want that ethernet interface to be added to the bridge, so ... *make it so* 

```sh
$ brctl show
bridge name           bridge id              STP enabled         interfaces
br0                   8000.a2d599084fcf      yes                 enp4s0f3u1u4
                                                                 enp4s0f3u1u1
```

Reboot the Zero, and now it should boot with the static IP we configured and say
something like

"My IP address is 192.168.0.201"

If everything went *particularly* well you should even be able to SSH into it!

```sh
$  ssh pi@192.168.0.201
The authenticity of host '192.168.0.201 (192.168.0.201)' can't be established.
ED25519 key fingerprint is SHA256:gfs9NKRE1y7Oy0lrA3F9dcXg56JEmN0yyFRoAo8o86M.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.0.201' (ED25519) to the list of known hosts.
pi@192.168.0.201's password: 
Linux raspberrypi 5.10.17+ #1414 Fri Apr 30 13:16:27 BST 2021 armv6l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.

SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.

pi@raspberrypi:~$ 
```

Next step is to create a ramdisk that has enough "stuff" (technical term) in it to
boot without the SD card.

Edit `/etc/initramfs-tools/modules` so it has *at least* this:

```
g_ether
libcomposite
u_ether
udc-core
usb_f_rndis
usb_f_ecm
```

Create the ramdisk:

```
$ sudo update-initramfs -c -k `uname -r`
update-initramfs: Generating /boot/initrd.img-5.10.17+
$ sudo mv /boot/initrd.img-5.10.17+ /boot/initrd.img
```


Reboot one last time to make sure everything works.

Now we will take this SD card, and turn it into a NFS root so we can boot our Zero off
it without any onboard storage.

* Take the SD card and put it back in your conmputer.

As root, create a "p1" folder somewhere and copy the contents of the SD card in it preserving permissions and such:

```
# mkdir -p /zeros/p1
# cp -a /run/[whatever]/rootfs/* /zeros/p1/
# cp -a /run/[whatever]/boot/* /zeros/p1/boot/
```

We'll need to configure a few things to make the Zero boot with root on NFS:

* Change `/root/p1/boot/cmdline.txt` to look more or less like this (adjust as needed):

```
console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=192.168.0.72:/zeros/p1 rw elevator=deadline fsck.mode=skip rootwait modules-load=dwc2,g_ether ip=192.168.0.201:192.168.0.1::255.255.255.0:p1:usb0:static rootwait
```

Explanation:

* Set root to be nfs
* Set nfsroot to `{the ip of your NFS server}:/zeros/p1` 
* Don't try to FSCK a NFS server
* Configure `usb0` network interface to be static, IP is `192.168.0.201`, etc.

* Share `/zeros/p1` via NFS with the IP of the zero by adding this in `/etc/exports`:

```
/zeros/p1 192.168.0.201(rw,async,insecure,no_subtree_check,no_root_squash)
```

Then run `exportfs -arv`

* Edit `/zeros/p1/etc/fstab` and remove references to local devices. That usually means 
  that the only uncommented line will be the one about `proc`. Mine looks like this:
  
```
proc            /proc           proc    defaults          0       0
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
```

At this point, your Zero should be able to boot without a SD card if we give it a little help. Plug it in (without SD card, of course) and run this:

```
$ sudo rpiboot -v -d /zeros/p1/boot/
```

It should start telling you how it's sending files to a device. What's happening is that
`rpiboot` is giving your Zero the files it needs to *start* booting, such as the kernel, the ramdisk and so on. After a while it should be fully booted, with network and SSH active.

At this point we have a fully functional and confitured NFS root tree. Let's save it.

As root:

```
# cd /zerros/p1
# tar cfJ ~/pi1.tar.xz .
```

You will now have a fully-configured image in ~/pi1.tar.xz. I have made mine available [here](https://drive.google.com/file/d/19rwrtPAysFs_3XwJBUaAYcnXG6EdF4wi/view?usp=sharing)

### The Easy Way

If you have read "The Hard Way" just smirk condescendingly and move along.

* Download [pi1.tar.xz](https://drive.google.com/file/d/19rwrtPAysFs_3XwJBUaAYcnXG6EdF4wi/view?usp=sharing) 

As root in your machine:

```
# mkdir -p /zeros/p1
# cd /zeros/p1
# tar xvf /wheverver/you/put/it/pi1.tar.xz
```

* Choose a static IP for the Zero (I used 192.168.0.201)

* Edit the following files to make sure they agree with your network configuration:

* `/zeros/p1/boot/cmdline`
* `/zeros/p1/etc/network/interfaces.d/usb0`

* Export `/zeros/p1` via NFS for the IP of the Zero by putting something like this in `/etc/exports`

```
/zeros/p1 192.168.0.201(rw,async,insecure,no_subtree_check,no_root_squash)
```

Then run `exportfs -arv`

At this point, your Zero should be able to boot without a SD card if we give it a little help. Plug it in (without SD card, of course) and run this:

```
$ sudo rpiboot -v -d /zeros/p1/boot/
```

It should start telling you how it's sending files to a device. What's happening is that
`rpiboot` is giving your Zero the files it needs to *start* booting, such as the kernel, the ramdisk and so on. After a while it should be fully booted, but the network won't work.

At some point, your computer (not the Zero!) will have a new network interface.
In my case it's called `enp4s0f3u1u1`

**NOTE:** The actual USB port on which you plug it is important, because it determines the
name of the ethernet interface to which your Zero is connected, so ... take notes.

We want that ethernet interface to be added to the bridge, so ... *make it so* 

```sh
$ brctl show
bridge name           bridge id              STP enabled         interfaces
br0                   8000.a2d599084fcf      yes                 enp4s0f3u1u4
                                                                 enp4s0f3u1u1
```

Reboot the Zero, and now it should boot with the static IP we configured and say
something like

"My IP address is 192.168.0.201"

If everything went *particularly* well you should even be able to SSH into it!

Congratulations, you made it the easy way. WHICH IS GOOD ENOUGH.

## The second Zero

If you have more than one Zero (you should!) then you may want to automate the second one further. Actually, after we are done with this part we'll have something easier than "the easy way".

What's different for the second zero?

* IP is `192.168.0.202` instead of `192.168.0.201`
* Its nfsroot is `/zeros/p2` instead of `/zeros/p1`
* The interface we add to the bridge will be different.

Further, if this was configured in a whole different network and system that's not my own, what would change?

* Default route, DNS servers, network mask are different
* IP address of the NFS server is different
* Path to the nfsroots is different
* Ethernet interface name and bridge name will be different

That's a surprisingly small number of things to change to boot N different systems off
a server.

Nothing else?

Well, we'll need to use one other feature of rpiboot so it works correctly for more than 
one device, but it's easy.

So, we need a simple way to "template" our tarball with the nfsroot and make it accept a few parameters and configure a few files.

**Note:** We know *exactly* what files need templating *because I took notes.* TAKE NOTES WHEN YOU DO STUFF.

### Templated nfsroot

The changes we need to do to add a second, third or whatever many Zeros we want are pretty mechanical. So, let's automate them. Or not, feel free to do things by hand like a farmer.

* Template language: jinja2
* Needed tools: [j2cli](https://github.com/kolypto/j2cli)

Not to go into details but here is the file with the configuration data [config.yaml](https://github.com/ralsina/zero-nfsroot/blob/main/config.yaml) you can edit and the only files that need templating are these:

* `/zeros/pX/boot/cmdline.txt` [templated here](https://github.com/ralsina/zero-nfsroot/blob/main/cmdline.txt.j2)
* `/zeros/pX/etc/network/interfaces.d/usb0` [templated here](https://github.com/ralsina/zero-nfsroot/blob/main/usb0.j2)

So, if you expand the tarball as `/zeros/p3` you can configure it doing something like this:

```
$ j2 cmdline.txt.j2 config.yaml -o /zeros/p3/boot/cmdline.txt
$ j2 usb0.j2 config.yaml -o /zeros/p3/etc/network/interfaces/usb0 
```

Now configure `/etc/exports` and then boot the Zero and do the whole "add interfaces to the bridge" thing.

### Making rpiboot less annoying

Currently to boot, say, raspbery pi #3 we need to go to `/zeros/pi3/boot` and run rpiboot. That is ... not practical.

Luckily, rpiboot supports booting multiple devices independently **as long as we know their USB path.**

Their what? Their USB path. So, USB is shaped like a tree. Here's mine (yes, my computer is complicated):

```
$  lsusb -t
/:  Bus 04.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/1p, 10000M
/:  Bus 03.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/2p, 480M
    |__ Port 1: Dev 2, If 0, Class=Hub, Driver=hub/4p, 480M
        |__ Port 2: Dev 4, If 0, Class=Vendor Specific Class, Driver=, 480M
    |__ Port 2: Dev 3, If 0, Class=Wireless, Driver=btusb, 12M
    |__ Port 2: Dev 3, If 1, Class=Wireless, Driver=btusb, 12M
/:  Bus 02.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/4p, 10000M
    |__ Port 1: Dev 2, If 0, Class=Hub, Driver=hub/4p, 5000M
        |__ Port 4: Dev 8, If 0, Class=Vendor Specific Class, Driver=r8152, 5000M
    |__ Port 2: Dev 7, If 0, Class=Hub, Driver=hub/4p, 5000M
    |__ Port 3: Dev 3, If 0, Class=Hub, Driver=hub/4p, 5000M
        |__ Port 4: Dev 4, If 0, Class=Hub, Driver=hub/4p, 5000M
/:  Bus 01.Port 1: Dev 1, Class=root_hub, Driver=xhci_hcd/4p, 480M
    |__ Port 1: Dev 2, If 0, Class=Hub, Driver=hub/4p, 480M
        |__ Port 1: Dev 110, If 0, Class=Vendor Specific Class, Driver=, 12M
    |__ Port 2: Dev 11, If 0, Class=Hub, Driver=hub/4p, 480M
        |__ Port 1: Dev 20, If 0, Class=Hub, Driver=hub/4p, 480M
            |__ Port 3: Dev 21, If 0, Class=Human Interface Device, Driver=usbhid, 12M
            |__ Port 3: Dev 21, If 1, Class=Human Interface Device, Driver=usbhid, 12M
        |__ Port 4: Dev 22, If 2, Class=Audio, Driver=snd-usb-audio, 12M
        |__ Port 4: Dev 22, If 0, Class=Audio, Driver=snd-usb-audio, 12M
        |__ Port 4: Dev 22, If 3, Class=Human Interface Device, Driver=usbhid, 12M
        |__ Port 4: Dev 22, If 1, Class=Audio, Driver=snd-usb-audio, 12M
        |__ Port 2: Dev 23, If 1, Class=Human Interface Device, Driver=usbhid, 12M
        |__ Port 2: Dev 23, If 2, Class=Human Interface Device, Driver=usbhid, 12M
        |__ Port 2: Dev 23, If 0, Class=Human Interface Device, Driver=usbhid, 12M
    |__ Port 3: Dev 3, If 0, Class=Hub, Driver=hub/4p, 480M
        |__ Port 1: Dev 45, If 0, Class=Hub, Driver=hub/4p, 480M
            |__ Port 1: Dev 92, If 2, Class=Audio, Driver=snd-usb-audio, 480M
            |__ Port 1: Dev 92, If 0, Class=Video, Driver=uvcvideo, 480M
            |__ Port 1: Dev 92, If 3, Class=Audio, Driver=snd-usb-audio, 480M
            |__ Port 1: Dev 92, If 1, Class=Video, Driver=uvcvideo, 480M
            |__ Port 1: Dev 92, If 4, Class=Human Interface Device, Driver=usbhid, 480M
            |__ Port 4: Dev 49, If 0, Class=Mass Storage, Driver=usb-storage, 480M
        |__ Port 4: Dev 52, If 0, Class=Hub, Driver=hub/4p, 480M
    |__ Port 4: Dev 4, If 1, Class=Video, Driver=uvcvideo, 480M
    |__ Port 4: Dev 4, If 0, Class=Video, Driver=uvcvideo, 480M
```

But don't worry, you don't need to understand all that, you just need to run one command and look carefully.

Unplug your Zero, and now boot it running this command:

```
rpiboot -vv -d /zeros/p1/boot/
```

Right before it starts sending files to the Zero it will show something like this:

```
Found device 1 idVendor=0x0a5c idProduct=0x2763
Bus: 1, Device: 113 Path: 1-1.1
Found candidate Compute Module...Loading: /zeros/p1/boot//bootcode.bin
Device located successfully
```

See that 1-1.1 ? That's the path (yours will be different)

Now create a `/zeros/boot` and create in it a link called `1-1.1` pointing to `/zeros/p1/boot` and repeat for every zero you want to boot like this.

```
# mkdir /zeros/boot
# cp /zeros/p1/boot/bootcode.bin /zeros/boot/
# ln -s /zeros/p1/boot/ /zeros/boot/1-1.1
```

It should look like this (with yout own USB paths instead of mine):

```
# ls -l /zeros/boot/
total 52
lrwxrwxrwx 1 root root    15 Jul 24 17:56 1-1.1 -> /zeros/p1/boot/
lrwxrwxrwx 1 root root    15 Jul 24 18:20 1-1.2 -> /zeros/p2/boot/
-rw-r--r-- 1 root root 52456 Jul 24 17:58 bootcode.bin
```

Now, if you run `rpiboot -v -o -d /zeros/boot -l` this will follow the link with the name of the USB path of the device and loop and try again and so on, booting as many Zeros as you need as many times as needed.

You can, of course, make this a systemd service or whatever.