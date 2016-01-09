# Setting up Rasberry Pi2 2 as a Seedbox with Transmission Remote.

### Prerequisites
In order to create a seedbox you are going to need the following hardware:

* A Raspberry Pi 2
* A Raspberry Pi Case. (Optional)
* A Raspberry Pi power adapter. (2 AMPS!)
* A Raspberry Pi SD Card with Raspbian configured. (See below)
* An external USB hard disk. (Your storage medium for torrents)

### Initial Setup and Pi2 Configuration
Turn on your Pi2 and login as:</br>
>usr: pi
</br>
>pass: raspberry

Become root</br>
`$ sudo su`

You’ll be wanting the Pi to have a static IP address for this so edit your `/etc/network/interfaces` files to make the eth0 section look like this:

```
auto eth0
iface eth0 inet static
	address 192.168.1.23
	netmask 255.255.255.0
	network 192.168.0.0
	broadcast 192.168.0.255
	gateway 192.168.1.1
	dns-search 192.168.1.1
```

My Pi is allocated 192.168.1.23, change yours and your gateway IP (router IP) to suit your network. Reboot and log back in on your new IP and become root again.

Update Raspbian</br>
`$ apt-get update`

Update the Pi2 to the latest packages</br>
`$ aptitude update; aptitude safe-upgrade` 

Run the Raspbian Config tool</br>
`$ raspi-config`

* Expand file system to ensure the entirety of the SD card is used by Raspbian.
* Change the user password.
* In boot options, choose B1.
* In Wait for Network at Boot choose Slow Wait to ensure that Transmission will have the network available at boot.
* You can overclock if you want, but I wouldn't advise it as it can reduce the lifetime of your Pi2.

* Advanced options:
  * Change hostname to "Seedbox" (or leave it as the default raspberrypi, but keep it noted for future reference)
  * Set the memory split to 16 as this is intended to be a headless installation so may as well make the maximum memory available.
  * Enable the SSH server.
  * Lastly update the Config Tool and wait for it to finish.

Reboot</br>
`$ shutdown -r now`

### Setting up your external Hard Drive

Check if the drive is connected with `blkid`

If not, the drive may not have enough power provided by the USB ports. To fix this, add this line to `/boot/config.txt`:</br>
`max_usb_current=1`

Reboot and check again

Run `fdisk -l` to check the drive name (`/dev/sda1` is mine, but yours may be different)

Once you know your drive name you can mount it. This is the tricky part, since by default the drive will be mounted read only. Also, in my case I am mounting an NTFS drive, which is not supported by default in Linux. The instructions I list here is what worked for me. Your results may vary.

Install fuse</br>
`$ apt-get install exfat-fuse exfat-utils`

Install ntfs-3g</br>
`$ apt-get install ntfs-3g`

Mount NTFS partition with read write access</br>
`$ mount -t ntfs-3g /dev/sda1 /home/pi`

Make sure the permissions are set right</br>
`$ chown 775 /home/pi` 

Create two directories for your Downloads and Incomplete Downloads. These directories must be owned by user `pi`</br>
`$ mkdir -p /home/pi/{Downloads,Incomplete}`</br>
`$ chown pi:pi /home/pi/{Downloads,Incomplete}`

### Transmission configuration
Install transmission</br>
`$ apt-get install transmission-daemon`

The configuration file for Transmission is `/etc/transmission-daemon/settings.json`. There you can setup the download directory which we created above and several other options.

To access transmission via web, enable RPC authentication.

```
"download-dir": "/home/pi/Downloads",
[...]
"incomplete-dir": "/home/pi/Incomplete",
[...]
"rpc-authentication-required": true,
"rpc-bind-address": "127.0.0.1",
"rpc-enabled": true,
"rpc-password": "password",
"rpc-port": 9091,
"rpc-url": "/transmission/",
"rpc-username": "username",
```
Change `user` under which Transmission runs and set it up as `pi`. Transmission will change the `password` with a hash algorithm. The rest can be configured with the web interface/remote client.

Change the download and incomplete directories to `/home/pi/Downloads` and `/home/pi/Incomplete`

Open the file `/etc/init.d/transmission-daemon` with your favorite editor and change the `USER` variable on line 13. It must look like `USER=pi`.


Now start transmission and make sure everything is working</br>
`$ sudo service transmission-daemon start`

Connect to http://raspberry-ip-address:9091 and login to your Transmission installation!

You are now done!

Sources:

https://www.convalesco.org/articles/2015/06/08/raspberry-pi-seedbox-with-transmission-and-torguard/

https://docs.google.com/document/d/1yEunzA1DHaYake4jQYlgh4IPoxfcHP5CXLqQR_dSGHs/edit

