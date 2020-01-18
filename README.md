# Compiling TWRP For ZTE Nubia Z11 Mini (NX529J)

## Usual steps to get ready for compiling

### Packages Required For Compiling

Note: These packages are for Ubuntu 16.04. Newer versions of Ubuntu might require updated package versions.

Install the extra packages required for compiling.

    sudo apt-get install bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf lib32ncurses5-dev lib32readline6-dev lib32z1-dev lib32z-dev libc6-dev-i386 libesd0-dev libgl1-mesa-dev liblz4-tool libncurses5-dev libsdl1.2-dev libwxgtk3.0-dev libxml2 libxml2-utils lzop maven pngcrush schedtool squashfs-tools unzip x11proto-core-dev xsltproc zip zlib1g-dev

### Repo

Get the `repo` tool/script that android uses as a wrapper for dealing with multiple git repos at the same time.

    mkdir ~/bin
    curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
    chmod +x ~/bin/repo
    cat "export PATH=~/bin:$PATH" >> ~/.bashrc
    source ~/.bashrc

With these commands you create a directory called bin in your home folder and download the repo tool into it.
You then set it as exeutable and add the bin dir to the .bashrc file so you to call the `repo` command from anywhere.

## Repo sync sources

Since we are only compiling TWRP and not a full ROM we will get a minimal checkout.
We are using Omnirom as the TWRP source is there.
The Z11 Mini comes with Android 5.1 so we will get branch `twrp-5.1`.

    mkdir -p ~/android/twrp
    cd ~/android/twrp
    repo init -u git://github.com/lj50036/platform_manifest_twrp_omni.git --depth=1 -b twrp-5.1
    repo --time sync --no-clone-bundle -j8

You can change the number of threads (simultaneous downloads) by changing `j8` (twice number of CPU cores is a good start).

- `--time` will output the time taken at the end.
- `--depth=1` means only get the last commit and not the entire history which should reduce the amount downloaded.
- `--no-clone-bundle` will reduce the download further by disabling git bundles that aren't affected by the `-c` option.

This took 15mins to download on my 15MB/s connection.
The .repo dir was about 1.5 GB in size and the full twrp dir (including the 1.5GB .repo) was about 6.6 GB.


## Extracting Recovery/System img Files

This is a bit circular in a guide that is about compiling from source.
However, if you used someone else's TWRP to get root (or used some other method such as Kingroot) then you can do the following to extract system image files.

    adb shell
    su
    cd /dev/block
    #You probably don't have find at this point unless you installed busybox so we have to use recursive ls and grep
    #find -name by-name
    ls -lR|grep by-name
    #Search the output of the above command for partitions
    ls -l ./platform/7824900.sdhci/by-name|grep -e system -e recovery -e ' boot'

This gives us the partitions:

    boot -> /dev/block/mmcblk0p21
    recovery -> /dev/block/mmcblk0p22
    recovery2 -> /dev/block/mmcblk0p23
    system -> /dev/block/mmcblk0p26

Copy the partitions to img files on your sdcard:

    dd if=/dev/block/mmcblk0p21 of=/sdcard/boot.img
    dd if=/dev/block/mmcblk0p22 of=/sdcard/recovery.img
    dd if=/dev/block/mmcblk0p23 of=/sdcard/recovery2.img
    dd if=/dev/block/mmcblk0p26 of=/sdcard/system.img

Copy to your PC and remove them from your device to save space:

    #Copy
    adb pull /sdcard/boot.img
    adb pull /sdcard/recovery.img
    adb pull /sdcard/recovery2.img
    adb pull /sdcard/system.img
    #Remove
    adb shell rm /sdcard/boot.img
    adb shell rm /sdcard/recovery.img
    adb shell rm /sdcard/recovery2.img
    adb shell rm /sdcard/system.img

Copy the `build.prop` to your PC:

    adb pull /system/build.prop

## Device sources

In order to compile a ROM we need to populate these trees.

    ~/android/twrp/device/[vendor]/[devicename]
    ~/android/twrp/kernel/[vendor]/[devicename]
    ~/android/twrp/vendor/[vendor]/[devicename]

You can find the [vendor] and [devicename] in the build.prop you extracted.

    ro.product.manufacturer=nubia
    ro.product.device=NX529J

In our case [vendor] is nubia and [devicename] is NX529J.

It isn't critical to have kernel source to compile recovery as it is enough to just have the device tree *if* we include a pre-built kernel from our device.

### Create The Device Tree from Prebuilt Files

We can use scripts to create the device tree from the files extracted from our device.
This step is a bit annoying though as the minimal TWRP is Omnirom based and the script is from CyanogenMod/LineageOS.

***
**WE NEED ANOTHER PROJECT CONTAINING otatools for this to work**

We want only one branch and no history to make the clone as small as possible. I am using branch cm-12.1 as the stock ROM is Lollipop based but I don't think it matters.

    git clone --depth 1 --single-branch --branch cm-12.1 git@github.com:LineageOS/android_build.git

The .git dir is only 2.5 MB! Much better than getting a full ROM source!
***

An alternative is to try from here https://github.com/pbatard/bootimg-tools 

To start you need to compile some tools from the ROM (I am using octos ROM in this example) and put the directory that contains unpackbootimg on the PATH.

    cd ~/android/octos
    make -j4 otatools
    find out -name unpackbootimg
    #/home/buildbot/android/octos/out/host/linux-x86/bin/unpackbootimg
    export PATH=/home/buildbot/android/octos/out/host/linux-x86/bin:$PATH

Now run the command to generate the files (this assumes you have got the boot.img from your device - see below)

    cd  ~/android/octos/build/tools/device
    ./mkvendor.sh  nubia NX529J ~/android/boot.img

This will create a git repo in `~/android/octos/device/nubia/NX529J/` and add a bunch of files and commit them.

The most important things here is that it has added `recovery.fstab` and `BoardConfig.mk` which can be hard to create correctly.
We can now move this to our twrp tree.

    mv ~/android/octos/device/nubia ~/android/twrp/device/

To continue with this tree we need to add a `vendorsetup.sh` and change some files to be omnirom specific instead of CyanogenMod.
See also
https://docs.omnirom.org/Porting_Omni_To_Your_Device
https://docs.omnirom.org/Team_Win_Recovery_Project#TWRP-ifying

    cd ~/android/twrp/device/nubia/NX529J/
    echo "add_lunch_combo omni_NX529J-userdebug" > vendorsetup.sh

The system.prop file is empty in the template so delete it. We can always add it back if we ever need it.

    rm system.prop

Rename these 2 files so they are Omni specific and not CyanogenMod specific. 

    mv cm.mk full_NX529J.mk    
    mv device_NX529J.mk omni_NX529J.mk

Since we renamed the file we have to change the references too. CHange PRODUCT_MAKEFILES in AndroidProducts.mk to match.

    sed -i 's/device/omni/' AndroidProducts.mk

Change the PRODUCT_NAME in the omni_NX529J.mk so it says omni instead of full.

    sed -i 's/full_NX529J/omni_NX529J/' omni_NX529J.mk    

The reason why we rename device.mk to omni_NX529J.mk is that the build system looks for a make file based on the contents of the `vendorsetup.sh`.

e.g. our contains `add_lunch_combo omni_NX529J-userdebug` so the make file is `omni_NX529J.mk`.

Since we are only building TWRP and not a full ROM we don't need GPS. Delete the lines that contain gps (e.g. `device/common/gps/gps_us_supl.mk).

SImilarly we can delete the `common_full_phone.mk` lines in `full_NX529J.mk as they are CyanogenMod references.

## BoardConfig.mk

This is a crucial file and deserves its own section!

TWRP flags are typically at the bottom and you need at least the `TW_THEME` that defines your screen size.

    echo "# TWRP" >> BoardConfig.mk
    echo "TW_THEME := portrait_hdpi" >> BoardConfig.mk

You can get the `TARGET_BOARD_PLATFORM` from the build.prop as it is called `ro.board.platform`.
You can also get it from adb with

    cat /proc/cpuinfo|grep Hardware

Here are the outputs to compare

    ro.board.platform=msm8952
    Hardware	: Qualcomm Technologies, Inc MSM8952

So in the BoardConfig.mk we put 

    TARGET_BOARD_PLATFORM := msm8952

#####
Caused build failures
From the build.prop you can check for `ro.product.cpu.abi` or `ro.product.cpu.abilist` to get the `TARGET_CPU_ABI`.
We have 

    ro.product.cpu.abi=arm64-v8a
    ro.product.cpu.abilist=arm64-v8a,armeabi-v7a,armeabi

So we put

    TARGET_ARCH := arm64
    TARGET_CPU_ABI := arm64-v8a

We can also put these values. As a list of valid `TARGET_ARCH_VARIANTs` come from `find build/core/combo/arch -name "*.mk"|sort` 

    TARGET_ARCH_VARIANT := armv8-a
    TARGET_CPU_VARIANT := generic

#####

The partition sizes in the BoardConfig.mk are defined as follows.
> BOARD_BOOTIMAGE_PARTITION_SIZE: the number of bytes allocated to the kernel image partition.

> BOARD_RECOVERYIMAGE_PARTITION_SIZE: the number of bytes allocated to the recovery image partition.

> BOARD_SYSTEMIMAGE_PARTITION_SIZE: the number of bytes allocated to the Android system filesystem partition.

> BOARD_USERDATAIMAGE_PARTITION_SIZE: the number of bytes allocated to the Android data filesystem partition.

Below shows how you get them from adb.

We are doing this with basic android tools but if you already have root you might as well install busybox and get a better set of commands to make your life easier.

    adb shell
    su
    cd /dev
    ls -lR|grep by-name|grep block

For me this outputs `./block/platform/7824900.sdhci/by-name:` Then do:

    ls -l ./block/platform/7824900.sdhci/by-name|grep recovery

For me this outputs

    lrwxrwxrwx root     root              1970-06-22 19:11 recovery -> /dev/block/mmcblk0p22

The key part is the word at the end. You then get the number of blocks for that.

    cat /proc/partitions|grep mmcblk0p22  

For you this will have output

    179       22      40960 mmcblk0p22

This says there are 40960 blocks.
Each block is 1024 in size.

    40960 x 1024 = 41943040

The number should match the size of you recovery.img you extracted from your phone `ls -l recovery.img`.

Therefore in the BoardConfig.mk you put

    BOARD_BOOTIMAGE_PARTITION_SIZE := 41943040
    BOARD_RECOVERYIMAGE_PARTITION_SIZE := 41943040

NB: Verifying with BusyBox you can do:

    fdisk -l /dev/block/mmcblk0p22

Which returns the same value as before:

    Disk /dev/block/mmcblk0p22: 40 MB, 41943040 bytes

We can now do the same for the system and data partitions.

    ls -l ./block/platform/7824900.sdhci/by-name|grep system
    ls -l ./block/platform/7824900.sdhci/by-name|grep data
    cat /proc/partitions|grep mmcblk0p26
    cat /proc/partitions|grep mmcblk0p41

This gave us

    179       26    2621440 mmcblk0p26                      
    259        9   57335791 mmcblk0p41

Which translates to

    2621440 * 1024 = 2684354560
    57335791 * 1024 = 58711849984

Therefore we put the following in BoardConfig.mk

    BOARD_SYSTEMIMAGE_PARTITION_SIZE := 2684354560
    BOARD_USERDATAIMAGE_PARTITION_SIZE := 58711849984

We can leave this one as is `BOARD_FLASH_BLOCK_SIZE := 131072`

## Compile

Now we have everything we need to compile.

    cd ~/android/twrp
    source ./build/envsetup.sh
    lunch
    time mka recoveryimage

## Test On Your Phone

You don't want to flash it right away and overwrite the existing recovery just in case there are bugs!

What you can do is load it temporarily by using `fastboot boot` instead of `fastboot flash`.

Here we are using `sudo` and forcing the vendor id as this version of fastboot doesn't recognise the phone as it is a new phone and also not the latest version of fastboot.

    adb reboot bootloader
    sudo fastboot -i 0x19d2 boot out/target/product/NX529J/recovery.img

If you are happy with it you can replace your existing recovery with

    adb reboot bootloader
    sudo fastboot -i 0x19d2 flash recovery out/target/product/NX529J/recovery.img

**NB:** The latest version of fastboot now recognises the phone so you can skip the `-i 0x19d2` if you are up to date.

If after testing it on your device you get the following error you will need to use a different process to unpack the kernel from the boot.img file.

    ~/android/twrp $ sudo fastboot boot out/target/product/NX529J/recovery.img
    downloading 'boot.img'...
    OKAY [  0.394s]
    booting...
    FAILED (remote: dtb not found)

We need a separate project (an alternative is Carliv Image Kitchen - https://forum.xda-developers.com/android/development/tool-cika-carliv-image-kitchen-android-t3013658).

    cd ~/android
    git clone git@github.com:xiaolu/mkbootimg_tools.git
    cd mkbootimg_tools

From here we run the script passing in our recovery.img or boot.img and and output folder.

    ./mkboot ~/android/recovery.img ~/android/recoveryextract

The key files in the output folder are dt.img, img_info, kernel. You can move the dt.img and kernel (overwrite the other generated one) to your project but we need to extract info from the img_info file to put in the BoardConfig.mk.

The line starting `cmd_line=` replaces `BOARD_KERNEL_CMDLINE` in the BoardConfig.mk. e.g.

    cmd_line='console=null androidboot.console=ttyHSL0 androidboot.hardware=qcom msm_rtb.filter=0x237 ehci-hcd.park=3 androidboot.bootdevice=7824900.sdhci lpm_levels.sleep_disabled=1 earlyprintk'
    #becomes
    BOARD_KERNEL_CMDLINE := console=null androidboot.console=ttyHSL0 androidboot.hardware=qcom msm_rtb.filter=0x237 ehci-hcd.park=3 androidboot.bootdevice=7824900.sdhci lpm_levels.sleep_disabled=1 earlyprintk

Take the value from the `ramdisk_offset` and the `tags_offset` and put them in a new line in the BoardConfig.mk. e.g.

    ramdisk_offset=0x02000000
    tags_offset=0x00000100
    #becomes
    BOARD_MKBOOTIMG_ARGS :=  --ramdisk_offset 0x02000000 --tags_offset 0x00000100 --dt device/nubia/NX529J/dt.img

You should be able to recompile and test the img now.

    make clean
    lunch
    time mka recoveryimage
    adb reboot bootloader
    sudo fastboot boot out/target/product/NX529J/recovery.img

## fstab file

The `mkvendor.sh` creates a `recovery.fstab` file which describes the partitions on your device. However, TWRP now requires it to be called `twrp.fstab' so we need to rename it.

    mv recovery.fstab twrp.fstab

We also need to make sure that it is copied by changing `PRODUCT_COPY_FILES` in omni_NX529J.mk. The format of the line is `<location in project><:><recovery/root/etc/twrp.fstab>`

    PRODUCT_COPY_FILES += \
        $(LOCAL_KERNEL):kernel \
        device/nubia/NX529J/twrp.fstab:recovery/root/etc/twrp.fstab

We should also add it to the BoardConfig.mk.

    # Recovery
    TARGET_RECOVERY_FSTAB := device/nubia/NX529J/twrp.fstab

The fstab created by `mkvendor.sh` is likely to be wrong. Take the contents of `ramdisk/etc/recover.fstab` from the extracted recovery image and reorder the columns (skip the last 2).
e.g.

    /dev/block/bootdevice/by-name/system       /system         ext4    ro,barrier=1                                                    wait
    /dev/block/bootdevice/by-name/cache        /cache          ext4    noatime,nosuid,nodev,barrier=1,data=ordered                     wait,check

becomes

    /system           ext4        /dev/block/bootdevice/by-name/system
    /cache            ext4        /dev/block/bootdevice/by-name/cache

If a file system type says `fat` you can change it to `vfat`.

# Alternatives

Other ways of doing stuff,

## Create Device Tree By Hand (incomplete)

There is a template (based on 6.0 but never mind) that we can use.

    cd ~/android/twrp/device
    wget https://github.com/lj50036/android_device_vendor_codename/archive/android-6.0.zip
    unzip android-6.0.zip
    rm android-6.0.zip
    mkdir nubia
    mv android_device_vendor_codename-android-6.0/ nubia/NX529J

Now we need to edit the template to our needs.

    cd nubia/NX529J/
    mv omni_codename.mk omni_NX529J.mk
    sed -i 's/omni_codname-eng/omni_NX529J-userdebug/' vendorsetup.sh
    sed -i 's/codename/NX529J/' Android.mk
    sed -i 's/codename/NX529J/' AndroidProducts.mk
    rm system.prop

## Kernel (incomplete)

We can also compile the kernel if we want to.

We need to get the kernel source from somewhere.
ZTE are nice enough to have a website http://opensource.ztedevice.com/.
The Nubia Z11 Mini (NX529J) is not listed so we need to email tech.sp@zte.com.cn

I think this might be the official kernel source on GitHub but I don't have confirmation.
https://github.com/ztemt/NX529J_L_kernel


# Notes

There is a Chinese version and an international version of this phone. The Chinese version will have CN in the build number.
e.g. `NX529J_CNCommon_V1.26`
Also the international version doesn't come in 64 GB apparently.Recovery
Remember, just because your phone is in English doesn't mean that it is the international version!

## Lunch Combo

The name of the lunch combo can have different suffixes:
* eng
* user
* userdebug

> **eng** - Engineering build comes with default root access.

> **user** - User build is the one flashed on production phones. Has no root access.

> **userdebug** - User debug build does not come with default root access but can be rooted. It also contains extra logging.

One thing to note here is although an eng build might suggest extra logging it is not so. The user-debug will contain maximum logging and should be used during development

## Specs and Similar Phones

**Moto G4 Plus** has the same specs:

> Qualcomm Snapdragon 617 MSM8952 SoC

> CPU - 4x1.5 GHz ARM Cortex-A53+ 4x1.2 GHz ARM Cortex-A53

> GPU - Qualcomm Adreno 405

## Tag manifest

To get a repeatable build you can generate a manifest specific to your build.

    repo manifest -o - -r

## Useful Links

How to compile TWRP touch recovery
https://forum.xda-developers.com/showthread.php?t=1943625

How to compile TWRP from source step by step
https://forum.xda-developers.com/android/software/guide-how-to-compile-twrp-source-step-t3404024

Understanding the Android Source Code
https://forum.xda-developers.com/showthread.php?t=2620389

How To Port CyanogenMod Android To Your Own Device
https://web.archive.org/web/20160421185935/https://wiki.cyanogenmod.org/w/Doc:_porting_intro

Carliv Image Kitchen
https://forum.xda-developers.com/android/development/tool-cika-carliv-image-kitchen-android-t3013658

Firmware download
(chinese) http://ui.nubia.cn/rom/detail/27
(Translated) https://translate.google.ca/translate?hl=en&sl=zh-CN&u=http://ui.nubia.cn/rom/detail/27

German site for international ROMs
http://www.nubia.com/de/support.php?a=phone&pid=9

Chinese Custom ROM
https://translate.google.ca/translate?hl=en&sl=zh-CN&tl=en&u=http%3A%2F%2Fbbs.ydss.cn%2Fthread-749441-1-1.html

Russian Notes
http://4pda.ru/forum/index.php?showtopic=776504&st=1400
https://translate.google.ca/translate?hl=en&sl=ru&u=http://4pda.ru/forum/index.php%3Fshowtopic%3D776504%26st%3D1400&prev=search

Russian TWRP
https://translate.googleusercontent.com/translate_c?depth=1&hl=en&prev=search&rurl=translate.google.ca&sl=ru&sp=nmt4&u=http://4pda.ru/forum/index.php%3Fshowtopic%3D776504%26st%3D60&usg=ALkJrhgTlQdySy28vJz_DQpwuaAQFEQe5A#entry53997718

Chinese TWRP on Russian website (download link now broken)
https://translate.googleusercontent.com/translate_c?depth=1&hl=en&prev=search&rurl=translate.google.ca&sl=ru&sp=nmt4&u=http://4pda.ru/forum/index.php%3Fshowtopic%3D776504%26st%3D0&usg=ALkJrhhF-p5MPxdPT9AQyqIGIBdcNA0V2Q#entry49974248

Chinese TWRP on Chinese website (download link now broken)
http://bbs.nubia.cn/forum.php?mod=viewthread&tid=647594&extra=page=1&filter=typeid&typeid=443
http://www.microsofttranslator.com/bv.aspx?from=&to=en&a=http%3A%2F%2Fbbs.nubia.cn%2Fforum.php%3Fmod%3Dviewthread%26tid%3D647594%26extra%3Dpage%253D1%2526filter%253Dtypeid%2526typeid%253D443

Chinese TWRP mirrored on a Polish website
http://telchina.pl/nubia-z11-mini-nx529j-t28452-24.html
links to https://drive.google.com/file/d/0B0hidJ2sijwIb0ctdk9kaWFGSmc/view

Translate language in ROM
https://translate.googleusercontent.com/translate_c?depth=1&hl=en&prev=search&rurl=translate.google.ca&sl=ru&sp=nmt4&u=http://4pda.ru/forum/index.php%3Fshowtopic%3D776504%26st%3D1840&usg=ALkJrhiJhNaN9Fquo2ndU9DH_Us-q_ldMw#entry57148464

Blobs lists
https://raw.githubusercontent.com/NFound/android_device_nubia_NX529J-1/cm-13.0/proprietary-files.txt
https://raw.githubusercontent.com/NFound/android_device_nubia_NX529J/master/proprietary-files.txt


# Pre-Built TWRP

As mentioned above, TWRP has already been built by a Russian (NFound) and a Chinese person (Stalence).
So how do you just use these?

Assuming Ubuntu. Install `adb` and `fastboot`.

At the time of writing the version in the repos is 1.0.32 but the latest version is 1.0.36.
You can download the latest version here
https://developer.android.com/studio/releases/platform-tools.html

Then you unzip and put it on your PATH.
e.g.

    sudo unzip platform-tools-latest-linux.zip -d /opt
    echo 'export PATH=/opt/platform-tools:$PATH' >> ~/.bashrc
    source ~/.bashrc

If you install it from the repos like this, this is what will happen.

    sudo apt install android-tools-adb
    sudo apt install android-tools-fastboot

Normally you would do the following to temporarily boot from an image.

    adb devices
    adb reboot bootloader
    fastboot devices
    fastboot boot twrp.img

However, although `adb devices` returns a value `fastboot devices` does not.
This was confusing until I downloaded the Chinese TWRP from the Polish website.
Inside was a bat file that had an extra parameter command

    sudo fastboot -i 0x19d2 boot NX529J_TWRP_3.0.2-0-521.img

According to the fastboot docs you can force fastboot to work with a device even if the vendor ID is unknown by the fastboot binary by using the -i parameter
http://elinux.org/Android_Fastboot#Commands
>-i <vendor id>                           specify a custom USB vendor id

You can find the vendor ID using

    lsusb |grep ZTE|cut -d: -f2
     ID 19d2

TWRP starts in Chinese but there is an English button that says Change Language and you can change to English.
He has also packaged SuperSu in the Advanced menu under "Stalence Tools" in order to root your phone.

 ## Rockchip devices

 If you have a rockchip device the partition layout will likely be different. You can try this.  
    adb shell
    su
    mkdir /mnt/external_sd/dumps
    dmesg >/mnt/external_sd/dumps/dmesg.txt
    logcat -d -v time >/mnt/external_sd/dumps/logcat.txt
    cat /proc/cmdline >/mnt/external_sd/dumps/cmdline.txt
    dd if=/dev/block/rknand_uboot of=/mnt/external_sd/dumps/uboot.img bs=4096
    dd if=/dev/block/rknand_resource of=/mnt/external_sd/dumps/resource.img bs=4096
    dd if=/dev/block/rknand_kernel of=/mnt/external_sd/dumps/kernel.img bs=4096
    dd if=/dev/block/rknand_boot of=/mnt/external_sd/dumps/boot.img bs=4096
    dd if=/dev/block/rknand_recovery of=/mnt/external_sd/dumps/recovery.img bs=4096
    dd if=/dev/block/rknand_system of=/mnt/external_sd/dumps/system.img bs=4096
    exit
    exit

 Then copy them to your PC.

     adb pull /mnt/external_sd/dumps
     adb pull /system/build.prop


