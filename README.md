# Setting up Rasberry Pi 2 as a Seedbox with Transmission Remote.

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

You’ll be wanting the Pi to have a static IP address for this so edit your `/etc/network/interfaces` files to make the eth0 section to look like this:

```
iface eth0 inet static
address 192.168.1.6
netmask 255.255.255.0
network 192.168.1.0
broadcast 192.168.1.255
gateway 192.168.1.1
```

To find your address, netmask and broadcast, run</br>
`$ ifconfig`</br>
To find gateway (router ip) and destination, run</br>
`$ netstat -nr`

Leave everything else untouched.

Remove any existing leases</br>
`$ rm /var/lib/dhcp/*`

My Pi is allocated 192.168.1.6, change yours and your gateway IP (router IP) to suit your network. Reboot and log back in on your new IP and become root again.

Update Raspbian to the latest packages</br>
`$ aptitude update; aptitude safe-upgrade`

Update Raspbian</br>
`$ apt-get update`

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
`$ reboot`

### Setting up your external Hard Drive
Become root</br>
`$ sudo su`

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

Make a directory in which to mount the USB drive</br>
`$ sudo mkdir /mnt/usbstorage`

Make pi the owner of the mounted drive and make its permissions read, write and execute for it</br>
`$ sudo chown -R pi:pi /mnt/usbstorage`</br>
`$ sudo chmod -R 775 /mnt/usbstorage`

Set all future permissions for the mount point to pi user and group</br>
`$ sudo setfacl -Rdm g:pi:rwx /mnt/usbstorage`</br>
`$ sudo setfacl -Rm g:pi:rwx /mnt/usbstorage`

Get the UUID of the drive:</br>
`$ ls -l /dev/disk/by-uuid/`

lrwxrwxrwx 1 root root 10 Jan 11 06:46 220EB2EB0EB2B6DF -> ../../sda1</br>
Note down the value of the UUID --> 220EB2EB0EB2B6DF

For all drive types mount the usb with this command, -o insures pi is the owner which should avoid permission issues</br>
`$ sudo mount -o uid=pi,gid=pi /dev/sda1 /mnt/usbstorage`

Now, we will configure RasPi to do this after every reboot:</br>
Take a backup of current fstab and then edit</br>
`$ cp /etc/fstab /etc/fstab.backup`</br>
`$ nano /etc/fstab`

Add the mount information at the bottom of the fstab file (remember to replace the UUID with your own):</br>
For NTFS:</br>
>UUID=220EB2EB0EB2B6DF /mnt/usbstorage ntfs nofail,uid=pi,gid=pi 0 0

For exFAT:</br>
>UUID=220EB2EB0EB2B6DF /mnt/usbstorage exfat nofail,uid=pi,gid=pi 0 0

For ext4:</br>
>UUID=220EB2EB0EB2B6DF /mnt/usbstorage ext4 nofail,uid=pi,gid=pi 0 0

Now test the fstab file works
`$ sudo mount -a`

Create two directories for your Downloads and Incomplete Downloads. These directories must be owned by user `pi`</br>
`$ mkdir -p /mnt/usbstorage/Transmission/{Downloads,Incomplete}`</br>
`$ chown pi:pi /mnt/usbstorage/Transmission/{Downloads,Incomplete}`

### Transmission configuration
Install transmission</br>
`$ apt-get install transmission-daemon`

Stop the daemon in order to make changes to the config file</br>
`$ service transmission-daemon stop`

The configuration file for Transmission is `/etc/transmission-daemon/settings.json`. There you can setup the download directory which we created above and several other options.

To access transmission via web, enable RPC authentication.

```
"download-dir": "/mnt/usbstorage/Transmission/Downloads",
[...]
"incomplete-dir": "/mnt/usbstorage/Transmission/Incomplete",
[...]
"rpc-authentication-required": true,
"rpc-bind-address": "127.0.0.1",
"rpc-enabled": true,
"rpc-password": "password",
"rpc-port": 9091,
"rpc-url": "/transmission/",
"rpc-username": "pi",
"rpc-whitelist": "*",
"rpc-whitelist-enabled": false,
```
Change `user` under which Transmission runs and set it up as `pi`. Transmission will change the `password` with a hash algorithm. The rest can be configured with the web interface/remote client. Ensure that the `rpc-whitelist-enabled` is set to false or you will not be able to log in remotly.

Now start transmission and make sure everything is working</br>
`$ service transmission-daemon start`

Connect to http://raspberry-ip-address:9091 and login to your Transmission installation!

OPTIONAL:</br>
If you get permission errors do the following:

Open the file `/etc/init.d/transmission-daemon` with your favorite editor and change the `USER`. It must look like `USER=pi`.

Open the file `/lib/systemd/system/transmission-daemon.service` and change the `User`. It must look like `User=pi`.

You are now done!

Sources:

https://www.convalesco.org/articles/2015/06/08/raspberry-pi-seedbox-with-transmission-and-torguard/

https://docs.google.com/document/d/1yEunzA1DHaYake4jQYlgh4IPoxfcHP5CXLqQR_dSGHs/edit

http://www.techjawab.com/2013/06/how-to-setup-mount-auto-mount-usb-hard.html

http://www.htpcguides.com/properly-mount-usb-storage-raspberry-pi/
