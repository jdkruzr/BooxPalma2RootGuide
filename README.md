# Boox Palma 2 (and Note Air 4C) Root Guide

**INITIAL DISCLAIMER: screwing this up could brick your device, do not attempt unless you thoroughly understand what is happening and why, please read all instructions top to bottom first so you know what to expect, always back up any and all of your important data before trying this**

**CREDITS AND THANKS: huge thanks to Renate from MobileRead.com for the resources here I used to create this guide: https://www.temblast.com/edl.htm**

**ETA 2024-12-14: this guide works equally well for the Note Air 4C (hereafter referred to as NA4C) as it does for the Palma 2 as they use precisely the same SoC.**

OK. On with the show ;)

# What Is Rooting?
Normally when using an Android or iOS device, you as the owner/user of the device cannot perform the device's equivalent of "su," "sudo," "run as Administrator," etc. The manufacturer reserves that right to themselves. This is basically because they don't want to be bothered with the support cost of you doing something that irreversibly breaks your device that they would then have to re-install or perform some other labor-intensive repair to fix. However, some manufacturers are more draconian about this [*stares intently at Apple*] than others. Rooting a Boox device is generally very simple. They always use Qualcomm processors with predictable ways of accessing the low-level hardware so that you can perform the operations necessary.

# Why Root Your Device?
You may not need to at all! It really depends on what you're trying to do. On pen-enabled devices like the Note Air 4C, rooting lets you edit the /onyxconfig/eac_config file which controls ink optimization like Boox enabled for OneNote -- faster/no-lag writing in the app. On the Palma 2, a device which (sadly) has no EMR layer for use with a stylus, rooting still lets you do things like use Titanium Backup to back entire apps and their data up to any location, or run software like nmap with privileges required to do things like putting WiFi interfaces into promiscuous mode so you can gather more data. It's the sort of thing where if you know you need it, you know why you need it.

**ETA 2024-12-29: take note: starting with firmware 4.0+ devices like the Note Air 4C, the above method of enabling handwriting/ink optimization no longer works the way it did because the config file has changed to something that appears not to be directly editable anymore. it is now in /onyxconfig/mmkv/onyx_config and seems to be partially binary, possibly using Tencent's MMKV format, although when I tried an MMKV CLI utility it was unable to read it. more as this story develops, etc. etc.**

**ETA 2026-05-17: Magisk Manager stuck on splash screen on BOOX firmware 4.1.x / 4.2 (Note Air 4C, and reportedly Palma 2 / P6 Pro):** if after following this guide root works (e.g. AdAway can be granted SU and `adb shell su whoami` hangs but apps that already hold the grant work), but the Magisk app itself never gets past the mask splash and `su` from a shell hangs, this is a known ActivityManagerService bug in BOOX's modified `services.jar` on 4.1.x+. The fix is the [`boox-ams-fix`](https://github.com/dynamicfire/boox-ams-fix) Magisk module by `dynamicfire`, which replaces `/system/framework/services.jar` via Magisk's systemless overlay (no partition writes). **Important caveat for Note Air 4C:** the prebuilt `boox-ams-fix` release ships a `services.jar` built for the BOOX Palma / P6 Pro and will *not* work on the Note Air 4C — installing it as-is leaves Magisk hung the same way. The working procedure on NA4C is to pull `/system/framework/services.jar` from the *device itself*, apply the same smali patch the module performs (toggling the AMS check that BOOX added) using `baksmali`/`smali`, repackage it into a copy of the `boox-ams-fix` module zip, and install that device-specific module via `magisk --install-module`. After reboot Magisk launches normally and `su` works from the shell. I confirmed this on `4.1.1-rel_1202_b14f57609` and again after the OTA to `4.2-rel_04282` (the patch must be redone whenever BOOX ships a new `services.jar`).

# Important Caveat on Boox Firmware Updates
There is something important to keep in mind especially if getting frequent beta software updates from Boox is important to you: there are two kinds of software updates. The first are "full updates." These are usually well over a gigabyte in size. They wipe and reinstall all of the system software (don't worry, your data and apps are safe) on all the system partitions of your device's storage. Importantly, this means they do not check to see if anything has been modified first: they don't have to, because they are just nuking and paving what's already there. These are completely fine to install when your device is rooted -- they will unroot you, but you can just root again using the procedure we describe below. The second kind of firmware updates from Boox are incremental updates. These apply changes to the existing software that's on your device instead and are much smaller, on the order of a few hundred megabytes. These **will not install** on a rooted device.

Now, I have tried to get around this on my Go 10.3 by writing the old, unrooted boot partitions back to the device using the same methods you will see below, and in the process soft-bricked my device and had to reinstall everything from recovery, which I must stress is kind of a pain in the butt. I'm not entirely sure what happened here. It could be that I mistook the old boot partitions from a *different* device for the ones from my Go 10.3 and that's how I screwed it up, by writing e.g. the boot partitions from my Palma 2 to my Go 10.3, unsurprisingly causing it to go kaput. (I increasingly think that's the case, honestly.) But it could also be that for whatever reason, this just doesn't work (a firmware write counter? idk, something, impossible to say), and if you want to apply an incremental update you will simply have to go into recovery and restore the firmware from there, possibly losing your data in the process (which would be fine, because you backed up anything you couldn't afford to lose before you started, right? 😉).

# K, How Does This Work?
So, first, these instructions are for Windows. For macOS instructions, see [below](#macos-instructions).

You will need the following:
* The ["EDL" utility](https://www.temblast.com/edl.htm) which is used to interface with Qualcomm devices like your Palma 2 on a lower level
* A ["loader" file](https://github.com/bkerler/Loaders/blob/main/lenovo_motorola/0000000000000000_bdaf51b59ba21d8a_fhprg.bin) for EDL which permits it to communicate with the device correctly
* The [Zadig utility](https://zadig.akeo.ie/) which installs a device driver used for your 
* The Android [platform-tools](https://developer.android.com/tools/releases/platform-tools#downloads) package for Windows which sends the Palma's Android OS commands
* The Magisk APK, which you will want to use [this link](https://github.com/topjohnwu/magisk/releases) **on your device** to download and install on your device -- Magisk is the modern way of handling root/superuser access on your Android device

Broadly, you are going to do this:
* Reboot your device to EDL mode
* Use EDL to grab both boot partition images from the Palma 2 / NA4C
* Reboot your device to regular Android mode
* Send both of the boot images to your Palma 2 / NA4C with ADB
* Use the Magisk app to patch each image to permit root access
* Pull the patched boot images back to your computer with ADB
* Reboot your device to EDL mode again
* Use EDL to send the Magisk-patched boot images to the appropriate boot partitions
* Reboot your device to regular Android mode
* Open Magisk and make any changes to settings you want to make

Sounds good? Let's get started.

# Step by Step
* Put your "platform-tools" folder from Google somewhere easy for you to get to from the command line. I just put it in my home folder. If you do that, you will be able to run it from the command line (Win-R, `cmd`, Enter) with something like `platform-tools-latest-windows\platform-tools\adb`
* Go to the Settings menu on your Palma 2 / NA4C (pull down the menu from the upper right corner, select the gear icon that shows up in the upper right), scroll down to "More Settings", turn "USB Debug Mode" on
* Connect your Palma 2 / NA4C to your computer. On the command line, try `platform-tools-latest-windows\platform-tools\adb devices` What you're looking for is something like this:
```
PS C:\Users\jtd\Downloads> ..\platform-tools-latest-windows\platform-tools\adb devices
List of devices attached
f69d9611        device
```
This step is a little fiddly. At first your device will probably not connect. You do have to go through "approving" the computer on the Palma 2 / NA4C which will pop up after a few tries of "adb devices" after turning USB Debug Mode on. It's not super consistent but it will work eventually.

* Once you can see the Palma 2 / NA4C with `adb devices,` do `adb reboot edl`. It will look on the device like nothing has happened. That's fine, don't worry, it has. It might also go almost entirely black. That's also fine.
* Download and run Zadig (found above). Go to "Options" and pick "List All Devices." From the dropdown, look at your list of USB devices. You should see "Quectel QDLoader 9008." Select that. Click "install driver."
* Download the loader file and EDL, from the list above. Notice that the loader file has a really long, complicated name. To make your life easier, rename it to something  like palma2.bin. On the command line, try `.\edl /lpalma2.bin` (yes, just like that -- assuming the edl.exe file is in your current directory and so is the palma2.bin file you renamed from the loader download -- the EDL program uses this weird command line switch format where you do not want a space between the l from /l and the name of the loader file.) You should see something like this:

```
PS C:\Users\jtd\Downloads> .\edl /lpalma2.bin
Found EDL 9008, handshaking... nope, resetting... version 2.1
HWID: 0013f0e100000000, JTAG: 0013f0e1, OEM: 0000, Model: 0000
Hash: d40eee56f3194665-574109a39267724a-e7944134cd53cb76-7e293d3c40497955-bc8a4519ff992b03-1fadc6355015ac87 (x3)
Sending palma2.bin 100% ok, starting... ok, waiting for Firehose... ok
```

* Now try `.\edl /u /g` . You should see it print the partition table.
* Pull the A slot boot partition from the device: `.\edl /u /r /pboot_a palma_boota_unrooted_4.0.img`
* Same for the B slot: `.\edl /u /r /pboot_b palma_bootb_unrooted_4.0.img`
* Modern Android devices have two "boot slots" they can use, which sort of gives you the ability to dual-boot various versions of the operating system. You don't need to worry about that specifically, but I have no idea if the boot partition slots are materially different from each other on Boox, so to be safe, we'll pull both partitions like we just did, modify both of them, and write them both to the device.
* Reboot the device back to Android with `.\edl /z`
* Use ADB to push both of the boot images to your device: `adb push palma_boota_unrooted_4.0.img /sdcard` and `adb push palma_bootb_unrooted_4.0.img /sdcard`.
* You installed Magisk from Github as shown above already, right? Great! Open Magisk and click the "Install" button in the upper right. Pick "select and patch a file." You will do this twice, once for boota.img and once for bootb.img. Each time it will produce a new .img boot partition file ending with a random set of characters. Make sure you keep track of which one is which, because again, I have no idea if it matters for Boox! You should probably rename them to something like `palma_magisk-boota.img` and `palma_magisk-bootb.img."
* Assuming you did that, do an `adb pull` of the new Magisk files back to your computer.
* Back to EDL mode. `adb reboot edl`.
* Now we push the modified boot images back on to your device's storage in the right places. `.\edl /u /w /pboot_a palma_magisk-boota.img` and `.\edl /u /w /pboot_b palma_magisk-bootb.img`, followed by `.\edl /z` to reboot back to Android again.
* You're running rooted! 🎉
* You can double check by opening Magisk again. One thing I would recommend is to go into the Magisk settings and block anything work or bank related from being able to see that you're rooted.

Enjoy! 

# macOS Instructions

Use the [same loader](https://github.com/bkerler/Loaders/blob/main/lenovo_motorola/0000000000000000_bdaf51b59ba21d8a_fhprg.bin) as mentioned in the Windows instructions. Unlike the original Palma which could be rooted solely with EDL and no custom loader, this is required. If it is not included, you may see the error below[^1].

First, install ADB. You can do this with Homebrew:

```
brew install android-platform-tools
```

Then install EDL via instructions [on their README](https://github.com/bkerler/edl#installation).

> [!NOTE]
> This is confirmed working with both latest Python 3.13 installed through `pyenv` or Homebrew. Was not able to get EDL installed with default macOS Python version located at `/usr/bin/python`. 

> [!NOTE]
> You may need to use this workaround to install the `pylzma` dependency:
> 
> ```sh
> CFLAGS=-Wno-int-conversion pip install pylzma
> ```

Then follow same remaining steps as in the Windows instructions, adjusted for the *nix versions of ADB and EDL:

```sh
adb devices
adb reboot edl
edl --loader=/path/to/loader.bin r boot_a,boot_b boot_a.img,boot_b.img
adb push ./boot_a.img /storage/emulated/0/Download
adb push ./boot_b.img /storage/emulated/0/Download

# Install Magisk on Palma, patch each image with Magisk, then run the following
# for both boot_a.img and boot_b.img:
adb pull /storage/emulated/0/Download/magisk_patched-{hash}.img
mv magisk_patch-{hash.img} magisk_patch-boot_{a,b}.img

# Then run:
edl --loader=/path/to/loader.bin w boot_a ./magisk_patched-boot_a.img
edl --loader=/path/to/loader.bin w boot_b ./magisk_patched-boot_b.img
edl --loader=/path/to/loader.bin reset
```

### Footnotes

[^1]: See example stack trace above. EDL may continue to show this error even when the loader is provided in subsequent commands until a full device restart - see thread [here](https://github.com/bkerler/edl/issues/508) for similar reports. This can be done on the Palma by holding down the lock and volume buttons.

```
Qualcomm Sahara / Firehose Client V3.62 (c) B.Kerler 2018-2024.
main - Using loader /Users/user/Downloads/palma.bin ...
main - Waiting for the device
main - Device detected :)
main - Mode detected: sahara
Traceback (most recent call last):
  File "/Users/user/.local/state/pyenv/versions/3.13.1/bin/edl", line 4, in <module>
    __import__('pkg_resources').run_script('edlclient==3.62', 'edl')
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/user/.local/state/pyenv/versions/3.13.1/lib/python3.13/site-packages/pkg_resources/__init__.py", line 747, in run_script
    self.require(requires)[0].run_script(script_name, ns)
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^
  File "/Users/user/.local/state/pyenv/versions/3.13.1/lib/python3.13/site-packages/pkg_resources/__init__.py", line 1734, in run_script
    exec(code, namespace, namespace)
    ~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/user/.local/state/pyenv/versions/3.13.1/lib/python3.13/site-packages/edlclient-3.62-py3.13.egg/EGG-INFO/scripts/edl", line 393, in <module>
    base.run()
    ~~~~~~~~^^
  File "/Users/user/.local/state/pyenv/versions/3.13.1/lib/python3.13/site-packages/edlclient-3.62-py3.13.egg/EGG-INFO/scripts/edl", line 293, in run
    version = conninfo.get("data").version
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: type object 'req' has no attribute 'version'
```
