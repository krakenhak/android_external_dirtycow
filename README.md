# CVE-2016-5195
CVE-2016-5195 (dirty cow/dirtycow/dirtyc0w) proof of concept for Android

## recowvery, an exploit tool for flashing recovery on "secure" systems with **unlocked** bootloaders
This means you, LG V20 H918 (T-Mobile)

This repository is set up for building inside an Android OS build environment.  
Please add it to your `local_manifests` folder as `dirtycow.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
	<remote  name="jc" fetch="ssh://git@github.com/jcadduono/" />
	<project path="external/dirtycow" name="android_external_dirtycow" remote="jc" revision="android-6.0" />
	<project path="external/libbootimg" name="android_external_libbootimg" remote="jc" revision="android-6.0" />
</manifest>
```

## Usage:

### Building:
```sh
lunch your_device-eng

make -j5 dirtycow recowvery-applypatch recowvery-app_process recowvery-run-as
```

### Running:
Note: Use `app_process32` on 32-bit targets.  

```sh
adb push dirtycow /data/local/tmp
adb push recowvery-applypatch /data/local/tmp
adb push recowvery-app_process64 /data/local/tmp
adb push recowvery-run-as /data/local/tmp

adb shell

$ cd /data/local/tmp
$ chmod 0777 *
$ ./dirtycow /system/bin/applypatch recowvery-applypatch
"<wait for completion>"
$ ./dirtycow /system/bin/app_process64 recowvery-app_process64
"<wait for completion, your phone will look like it's crashing>"
$ exit

adb logcat -s recowvery
"<wait for it to tell you it was successful>"
"[CTRL+C]"

adb shell reboot recovery
"<wait for phone to boot up again, your recovery will be reflashed to stock>"

adb shell

$ getenforce
"<it should say Permissive, adjust source and build for your device!>"

$ cd /data/local/tmp
$ ./dirtycow /system/bin/run-as recowvery-run-as
$ run-as exec ./recowvery-applypatch boot
"<wait for it to flash your boot image this time>"

$ run-as su
#
"<play around in your somewhat limited root shell full of possibilities>"
```

From your root shell, it's possible to use commands such as:
```sh
dd if=/sdcard/twrp.img of=/dev/block/bootdevice/by-name/recovery
```
If you have a [Team Win Recovery Project](https://twrp.me/) image on your internal storage, this is how you would install a custom recovery.

## How it works:

dirtycow manages to exploit an old bug in the copy-on-write code of the Linux kernel which can trick the system into running a different ELF executable in another "priveleged" executable's place.  
Don't quote me on that, I haven't researched it any more than I needed to.  

Anyways, we can use dirtycow to replace the `/system/bin/applypatch` executable, run by the `install_recovery.sh` script which is run by init in the `u:r:install_recovery:s0` selinux domain.  
While this script is intended for replacing the recovery partition image with the OEM original (based on diff of boot partition) if it is damaged, we can abuse it to install our own recovery images.  

The `u:r:install_recovery:s0` is the only context in Android that is able to read and write to the recovery partition. It has access to only a few other items, however.  
It can read (not write) the boot partition in order to build a full image from a diff file between stock recovery and stock boot.  
It can read and write to `/cache` (files only, not directories) for logging purposes.  
It can run binaries in `/system/bin` under its context with limited permissions. (ex. `applypatch` for applying the diff to create the recovery image, and `sh` for running the `install_recovery.sh` shell script)  

Using these abilities, we have a place to store ramdisk images for modification. (`/cache`)  
We have access to `gzip` for decompressing & compressing the ramdisk.  

recowvery-applypatch will load the boot image from the boot partition, set permissive on the cmdline (currently not necessary), and replace a `.rc` file in the ramdisk by appending it to the cpio.  
The `.rc` file in the ramdisk is run by init, and allows us to tell init to set the device to **SELinux Permissive** mode after the boot sequence completes.  
Once all the modifications to the boot image are made, it's flashed back to the recovery partition.

We can't start the install\_recovery service ourselves, it needs to be run by init's service control. The `u:r:system_server:s0` context is capable of starting init services, so that's where recowvery-app_process64 comes in handy.  
`/system/bin/app_process64` is a zygote executable. It brings up the Android framework and maintains it during system use.  

The `u:r:zygote:s0` context that app\_process64 starts in has permissions to transition to the `u:r:system_server:s0` context for when it brings up the system server.  
We can abuse that privelege by hijacking app\_process64 (which is run as root) with dirtycow and then transition to `u:r:system_server:s0` ourselves to start the install\_recovery service as root in the `u:r:install_recovery:s0` domain.  

With our modified boot image flashed to the recovery partition, we can reboot into recovery mode. Surprise, it's not recovery mode, it's a permissive system!  

We can now use dirtycow to replace the `/system/bin/run-as` execution with our own that is perfectly happy to elevate any command to root, as well as start a root shell for you.  

Once we're root and in permissive mode, we can call recowvery-applypatch again with boot as an argument to tell it to modify the boot image again, but this time flash it to the boot partition.  
You should now be able to start your system in SELinux Permissive mode on every reboot, allowing you to use dirtycow and `run-as` to gain root access (in a shell) whenever you'd like.  

Hope you enjoyed the read, and have fun exploiting!
