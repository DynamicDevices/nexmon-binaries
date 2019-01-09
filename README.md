
# Overview

I need to run Raspberry Pi variants in "monitor" mode with BalenaOS. The WiFi drivers/firmware don't support this out of the box.

Luckily there's a project called 'nexmon' which supports patching the binaries. This needs to be done on the device itself.

The process is further complicated for me as with BalenaOS I am running in a Docker container which needs specific BalenaOS Linux kernel headers for building the needed module.

What follows is an attempt to capture the process. It's all very hand-rolled at the moment...

The output binaries I have built so far are available for you to use from this repo. You can skip forward to Installation to see where they go.

# Bring up Balena device

Use one of the Balena quickstart guides to create an application and to download a base image for your device

https://www.balena.io/docs/learn/getting-started/raspberrypi3/cpp/

I'm going to assume you are using a Raspberry Pi 3 and the current *development* Balena image (*not* production) which is v2.29.0+rev1 at the time of writing.

Follow the instructions to download the device image, burn it to SD card and get to the point where the device is connected to the application in BalenaCloud.

(Given that we're going to be fiddling about with the WiFi drivers you're probably going to want to be connecting through the cloud).

You're going to need a "base" application running on the device as the "main" target. So make sure you follow right the way through the getting-started example so that 'balena-cpp-hello-world' is deployed and running.

(At the time of writing the Dockerfile.template in this example uses FROM resin/xxx-debian:stretch)

# Setup for nexmon build

Connect up to the running container on the target (not the host os) either via the BalenaCloud termina window or using balena-cli.

Install some basics we need from the build. The following is based on the nexmon instructions

@see https://github.com/seemoo-lab/nexmon#build-patches-for-bcm43430a1-on-the-rpi3zero-w-or-bcm434355c0-on-the-rpi3-using-raspbian-recommended

`mount -o remount,rw /lib/modules`

`apt-get update && apt-get upgrade`

`apt install raspberrypi-kernel-headers build-essential git libgmp3-dev gawk qpdf bison flex make ssh libfl-dev xxd`

Change to the root home directory

`cd`

Grab the nexmon project

Either with

`git clone https://github.com/seemoo-lab/nexmon.git`

Or I've forked the project to fix an RPiv3 issue I was seeing so you could use my fork at

`git clone https://github.com/DynamicDevices/nexmon.git`

We need to replace the default RPi kernel headers with the ones that were used to build the running Balena kernel.

There's an example out of tree module project which contains a build script we can use to determine where the appropriate sources are

@see https://github.com/balena-io-projects/kernel-module-build/blob/master/build.sh

This results in a path of the following format. If you're not using v2.29.0+rev1 *development* you'll need to change this (if production then 'dev' changes 'prod')

`curl https://resin-production-img-cloudformation.s3.amazonaws.com/images/raspberrypi3/2.29.0%2Brev1.dev/kernel_modules_headers.tar.gz -o /usr/src/kernel_modules_headers.tar.gz`

`cd /usr/src && tar xzvf kernel_modules_headers.tar.gz && mv kernel_modules_headers linux-headers-4.14.79`

`ln -s /usr/src/linux-headers-4.14.79 /lib/modules/4.14.79/build`

# Build the WiFi firmware

This is again taken from the nexmon instructions

`cd ~/nexmon/buildtools/isl-0.10 && ./configure && make && make install && ln -s /usr/local/lib/libisl.so /usr/lib/arm-linux-gnueabihf/libisl.so.10`

`cd ~/nexmon && . ./setup_env.sh && make`

* For RPiv3 use

`cd ~/nexmon/patches/bcm43430a1/7_45_41_46/nexmon  && make`

* Or for RPiv3+ use (and replace the v3 driver references to 'bcm43430a1/7_45_41_46'  below with 'bcm43455c0/7_45_154')

`cd ~/nexmon/patches/bcm43455c0/7_45_154/nexmon  && make`

If the above step fails with a linking error try again with

`cd ~/nexmon/patches/bcm43430a1/7_45_41_46/nexmon && make clean && make`

# Build the nexmon utility

`cd ~/nexmon/utilities/nexutil && make`

# Persist the build artifacts

There are different ways of persisting the build artifacts so they can be used to bring up the WiFi interface in monitor mode.

Here we'll copy them to a known partition (/data), locate that on the host OS and move the kernel module and firmware binary to where they need to be for Linux to load them automatically

Copy the configuration utility

`cp ~/nexmon/utilities/nexutil/nexutil /data`

Copy the firmware binary

`mkdir /data/brcm && cp ~/nexmon/patches/bcm43430a1/7_45_41_46/nexmon/brcmfmac43430-sdio.bin /data/brcm/`

Copy the kernel module

`cp ~/nexmon/patches/bcm43430a1/7_45_41_46/nexmon/brcmfmac_4.14.y-nexmon/brcmfmac.ko /data/`


Now use the BalenaCloud terminal interface or the Balena client to connect into the *Host OS* rather than the main container.

Show a list of the running Balena containers with

`balena ps`

This will look something like

```
root@3702575:~# balena ps
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS                                 PORTS               NAMES
18917afb2d5f        0bde209699ca                       "/usr/bin/entry.sh /…"   17 hours ago        Up About a minute                                          main_740651_724383
b420598e99f8        balena/armv7hf-supervisor:v9.0.1   "./entry.sh"             2 weeks ago         Up About a minute (health: starting)                       resin_supervisor
```

The 'main' container is the one we care about. Check the container ID, in this case starting with 18...

Now inspect that container looking for the mount points

`balena inspect 18`

You'll see a lot of information with a number of mount points. Locate the /data configuration which will look something like this

```
 {
                "Type": "volume",
                "Name": "1336122_resin-data",
                "Source": "/var/lib/docker/volumes/1336122_resin-data/_data",
                "Destination": "/data",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
 },
```

From the about you can see that the '/data' mount in the main container maps to '/var/lib/docker/volumes/1336122_resin-data/_data' on the host OS.

Alternatively you could use find, something like this

`cd / && find -name brcmfmac.ko`

Having located the 'data' folder on the host OS we now want to copy the files to the required locations

Make sure the host OS filesystem is writable

`mount -o remount,rw /`

Backup the original WiFi firmware

`mv /lib/firmware/brcm/brcmfmac43430-sdio.bin /lib/firmware/brcm/brcmfmac43430-sdio.bin.org`

Copy the new firmware to the expected location

`cp brcm/brcmfmac43430-sdio.bin /lib/firmware/brcm`

Backup the original WiFi driver

`mv /lib/modules/4.14.79/kernel/drivers/net/wireless/broadcom/brcm80211/brcmfmac/brcmfmac.ko /lib/modules/4.14.79/kernel/drivers/net/wireless/broadcom/brcm80211/brcmfmac/brcmfmac.ko.org`

Copy the new driver to the expected location

`cp brcmfmac.ko /lib/modules/4.14.79/kernel/drivers/net/wireless/broadcom/brcm80211/brcmfmac`

Copy the configuration utility to the expected location

`cp nexutil /usr/bin`

Make sure changes are persisted and reboot - the new WiFi driver should now be loaded!

`sync`

`reboot`

# Check the interface supports monitor mode

Run `iw phy` to show capabilities of the WiFi device

You might need to first install the tool with

`apt install iw`

You'll see something like the the following output which should now contain 'monitor' as a supported mode

```
	Supported interface modes:
		 * IBSS
		 * managed
		 * AP
		 * monitor
		 * P2P-client
		 * P2P-GO
		 * P2P-device
```

To add a monitor mode device:

`iw phy phy0 interface add mon0 type monitor`

`iw dev`

```
phy#0
	Interface mon0
		ifindex 10
		wdev 0x2
		addr b8:27:eb:4b:84:25
		type monitor
		channel 1 (2412 MHz), width: 20 MHz, center1: 2412 MHz
		txpower 31.00 dBm
	Interface wlan0
		ifindex 3
		wdev 0x1
		addr b8:27:eb:4b:84:25
		ssid TP-LINK_20D2
		type managed
		channel 1 (2412 MHz), width: 20 MHz, center1: 2412 MHz
		txpower 31.00 dBm
```

`ifconfig wlan0 down`

NB. We seem to get some strange errors with the RPiV3+ if we don't first take down the wlan0 interface

`ifconfig mon0 up`

```
root@3702575:/# ifconfig mon0
mon0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        unspec B8-27-EB-4B-84-25-00-00-00-00-00-00-00-00-00-00  txqueuelen 1000  (UNSPEC)
        RX packets 426  bytes 57349 (56.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

And you're now ready to use your packet monitoring tool of choice...
