# SolidHals Shitty guide to AOSP bringup on your android device

## No promises that these notes are accurate or even good advice.
## This information may become out of date quickly depending on how AOSP changes
## As usual, you will be flashing stuff to your device. I am not responsible if you brick your device. If you are not comfortable with flashing your device, and the risks associated, don't try any of this.

# NOTICE: if you skip reading this, you *will* have a bad time

Building AOSP for your device is *NOT* easy. Once it is built, getting everything booting, let alone getting a fully functional device is very very hard.
You will need extensive knowledge of linux operating systems and kernels, and be ready to read a lot aboutthe special ways in which android works.
Even with this knowledge, you will likely fail. Be ready to sink months of work into this.
If you do succeed, congratulations! Please share your sources and roms with folks around the xda fourms, lineage, or your other favorite rom. 

This is not a step by step guide, I cannot predict what things will/won't work for you. These are just some things I find useful, and you may too. There is a good chance your device differs in interesting ways to the ones I have experience with.


### This "guide" will be lineageOS focused, as that is the rom I am choosing to work with. Other roms are likely similar.

## Collect information on your device
This is not all of the information you will need, but it will get you started
Make sure to read this list thuroughly as not having some of these items means this is nearly impossible.

- Can you unlock your devices bootloader?
   - without an unlocked bootloader, this is impossible. If you don't know if your bootloader is unlocked, it probably isn't.
   - check on the xda fourms, a few devices have special exploits to unlock their bootloaders.
   - *DO NOT DOWNLOAD RANDOM PROGRAMS FROM OTHER SITES, they are likely viruses. If you needed this reminder, you likely shouldn't try to build a rom*
- does your devices manufacturer comply with GPL and post kernel source?
  - if they don't, you are better off looking into getting a GSI working instead.
      - since GKIs are not standard yet, you have little to no chance of getting a full rom working. Sorry.
      - *I am not a lawyer* but you can try emailing/calling/mailing their support or legal department asking for them to release the source as part of GPL. Most likely this will be ignored.
- What version of android did your device ship with, what version are you trying to make work?
  - if it is old, there is a good chance your manufacturer no longer supports the latest android.
  - this can make things difficult, as we rely often rely on the kernel source from the manufacturer to get off the ground
- cpu chipset, this is useful for finding similar devices
- does it support A/B ota, or not? 
  - https://source.android.com/devices/tech/ota/nonab 
  - https://source.android.com/devices/tech/ota/ab
- how are the partitions laid out? Do you know what dynamic partitions are?
  - https://source.android.com/devices/tech/ota/dynamic_partitions
  - https://source.android.com/devices/bootloader/partitions
  

## getting help
unfortunately, there is often no obvious "correct" answer to any issues you may face in this process
best you can do is read the error logs, read the code that creates them. The section below about useful tools will help you with this
even with the logs and the source code, it might take an experienced person quite a bit of time to figure out your specific issue. Don't be surprised when folks on irc or mailing lists are hesitant to offer up hours of their own time to debug your issue.
If you hit a road block, start reading any documentation or blog posts about the part of the code base giving you issue.

## get some useful tools
I highly, highly recommend `ripgrep` instead of grep. this will save you hours of time.
make sure you know how to use `find` as well
any error codes you get are somewhere in the giant AOSP code base, and these will help you find the source file you care about

## find a similar device to start from
- using the cpu chipset is a good way to do this.
ex: the sm8250 
https://github.com/LineageOS/android_device_xiaomi_sm8250-common
https://github.com/LineageOS/android_device_oneplus_sm8250-common
https://github.com/LineageOS/android_device_asus_sm8250-common

## make the repos
your device will at the minimum need its own:
- common repo
- device repo
- kernel repo

common and device repos can be forked and renamed from a similar device
please *don't* just copy the files manually and commit them in a new repo
this means they lose all of their git history, which you and others may find useful
please *do* fork and rename instead

your kernel repo likely will be a fresh new repo, place the source from your manufacturer in it

## get all of the sources
follow the beginning of this to get setup: https://wiki.lineageos.org/devices/bacon/build
then when you get to breakfast, go into `.repo/local_manifesets`
add your repos, and any supporting repos to your `.repo/local_manifesets/local_manifests.xml`
create the file if it doesn't exist.

example `local_manifests.xml`:
```
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
<project name="SolidHal/android_device_samsung_x1q" path="device/samsung/x1q" remote="github" revision="lineage-18.1" />
<project name="SolidHal/android_device_samsung_sm8250-common" path="device/samsung/sm8250-common" remote="github" revision="lineage-18.1" />
<project name="LineageOS/android_device_samsung_slsi_sepolicy" path="device/samsung_slsi/sepolicy" remote="github" />
<project name="LineageOS/android_hardware_samsung" path="hardware/samsung" remote="github" />
<project name="SolidHal/android_kernel_samsung_sm8250" path="kernel/samsung/sm8250" remote="github" revision="stock-11" />
</manifest>
```

now do another `repo sync` and then do `breakfast (your device)`

## get familiar with the common and device repos
some interesting things in the two repos:
#### common
- Android.mk
  - initial makefile for the common repo, usually has a line at the top that says which device codes this supports
  - you will likely need to change that line to include your devices code name
- common.mk (might be named device.mk or <cpu-chipset>.mk)
  - usually calls sub makefiles and lists which packages to include
- BoardConfigCommon.mk
  - usually defines build variables
- manifest and matrix xml files
  - https://source.android.com/devices/architecture/vintf/objects
  - https://source.android.com/devices/architecture/vintf/comp-matrices
  
#### device
- Android.mk
- AndroidProducts.mk
- BoardConfig.mk
- device.mk
- setup-makefiles.mk

## get your functional reference
- pull all of the partitions off of your known working device. you will want copies of these locally before you start overwriting them.
  `ripgrep`-ing and `find`-ing through these file systems will be very useful
- this will require root, or the ability to disassemble and mount the factory images
- use `file` on the images to get hints about what you need to do
- you may need to use `simg2img`
- you may need the mount option `-o ro`
- on the device, run commands like `mount` and `ls /dev/block/bootdevice/by-name/`
- explore the root of the device file system to see how each partition is mounted

## get the recovery image booting

a good place to start is to get a booting recovery image, since this shows your kernel has some basic functionality.
look over those files listed in the above section, you will need to modify them to match your device
specifically you should look at the kernel variables in the BoardConfigCommon.mk/BoardConfig.mk
you may need to modify your devices kernel config in the kernel source

you can look at the stock recovery image by doing:
```
abootimg -x recovery.img
```
which gives you the kernel, which can give you hints about the cmdling params you should use
and the initrd.img
which you can further take apart by doing something like:
```
gunzip -c initrd.img | cpio -i
```

after running `brunch` you can find your recovery.img in `out/target/product/<device_name>/`

this can be difficult to debug, but if you have either a rom or recovery you have adb root priviledges on you can still get some logs by doing the following:
1) try booting to your recovery image
2) when it fails, or if it takes more than ~2 minutes boot instead to your rom or your known good recovery
3) pull the last_kmsg `adb shell cat /proc/last_kmsg > last_kmsg`

look at that log to get hints about why it is failing.
Something I found useful to look for is `Booting Linux` which brings you to the start of each kernel boot attempt

some tips: 
- look at what recovery init you are using. Pulling the fstab and recovery init from the known working recovery is likely a good idea.
- get it mounting the correct partitions
- this is a combination of modifying the `.fstab` file, located either in your common or device repos and the `PARTITION` settings in the
  `DeviceConfigCommon.mk` and `BoardConfig.mk`
- get adb working, you should be able to do `Advanced > Enable ADB` and have functional ADB. You may have to modify the `.prop` files in the `common` and `device` repos. Look for `.prop` files in your stock file systems.

## make sure the the recovery image is actually working
- can format userdata and cache
- can use ADB

## get vendors building
- look at the muppets repo, create a vendor/common directory for your device
- modify the device/common .mk files to use it
- `proprietary_files.txt` defines all of the precompiled blobs and libraries you device uses
- pull those files either from a device, or from a stock image and put them in your vendor/common directory
- `setup-makefiles.sh` in the device repo can be ran to configure the vendor proprietary repo makefiles automagically
- you likely will want to pull all init scripts into the main common/device repos instead of the proprietary repos

## build and test your image, a lot
your first build won't boot
make sure you are running selinux permissive in the kernel cmdline for now

make sure you check out the section on adb logcat below with some tips to get it working.

if you can't get adb working and need logs:
since you have a functional recovery, you have some debugging options
1) last_kmsg
2) /sys/fs/pstore/
parse these logs for failures. Start with mount failures, as those will cause lots of others
make sure the init scripts aren't erroring out
next make sure keymaster and gatekeeper aren't causing issues

some tips for looking at these logs:
- `exited with status 1` is a bad thing, figure out what service is failing and why. Do you have the correct prebuilt files? Are your .mk files configured properly?
  it is also possible that you have some service lying around from your reference device, and your device does not support it
- `init: Control message: Could not find`
- `init: Could not start service`
- `times before boot completed`
- like before, `Booting Linux` is your sign of a new boot
- the log buffers can be small in size, meaning if you get stuck in boot for a long time the actual failure may get dropped from the buffer
  to work around this, you can force the device off after ~20 to 30 seconds to comb through the start of boot


## Get your build booting
if you get this far, congratulations, most of the hardest work is over. You now have access to a bunch of android debugging tools like `logcat` that will make the next steps easier.

now debug:
- hardware functionality
- selinux functionality, make sure you remove selinux permissive

see below for some tips on this...

## adb logcat

this is one of the most important hurdles to overcome, as once you have the adb daemon (adbd) running you
can start eaily getting all of the logs instead of just the `last_kmsg` fragment

even if your device doesn't boot, you can run
```
adb wait-for-device logcat
```
which will dump all of your logs as soon as a device is available

### Things to try if adb doesn't work

Look for `start adbd` in your devices `.rc` files. There will likely be multiple.
Make sure your devices definition of `sys.usb.config` in the `.prop` files matches one of them.
### IMPORTANT the strings after the `=` must match exactly in the `.prop` and the `.rc`
ex: 
in the `.prop` files
```
sys.usb.config=mtp,conn_gadget,adb
```
and in the `.rc`
```
on property:sys.usb.config=mtp,conn_gadget,adb
```
As a general tip for android init scripts, you can use the following to print something in `last_kmsg`
```
    write /dev/kmsg "SOLIDHAL starting adbd"
```

## Compare your build to stock

the easiest way to compare a build vendor, odm, system, etc. partition to a stock one is to use git
for these instructions, I will use the vendor partition as an example

*NOTE* some files may end up in a different partition in the built image than in the stock image, this is usually alright for things like libraries

1) copy the stock vendor files to a directory we will call `stock`
2) copy the built vendor files to a directory we will all `built`
   - on lineage, these are available at `out/target/product/<dev_name>/vendor`
3) run `git init` in `stock`
4) copy `stock/.git` to `built/.git`
5) `git status`
6) `git ls-files -d` is useful for seeing just deleted files, but looking at the modified files is good to do as well


#### simpler, but with drawbacks

this can only really be used to compare small directories with a few differences, otherwise `diff` gets un-helpful

get a one item per line list of everything in a directory
the `|` tells ls to change its output to one item per line
```
ls -a | cat > file_list.txt
```
run that in your stock and built directory and diff the two lists

## share your work
Congratulations, you successfully brought up an AOSP rom for your device. This is no small feat.
Reward yourself in whatever way you find appropriate.
Brag about it to your friends, maybe they will say "oh wow, cool? why did you waste your time on this??" :D
Share your sources and roms with folks around the xda fourms, lineage, or your other favorite rom. I'm sure others will appreciate it.

if you have any suggestions to improve this guide, please submit a PR.

