# TWRP Compile Guide

A technical guide on how to compile TWRP for a new device.

The [official guide][1] currently links to an [XDA thread][2], however, there is a lot of assumed detail that is not covered, such as where/how to clone the source etc etc.
    
## Prerequisites

This guide is assuming an Ubuntu based system for building.
If you are using a different O/S or distro you will have to modify some of the commands.

### Packages Required For Compiling

Extra packages required for compiling the android source code.

    sudo apt install bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf lib32ncurses5-dev lib32readline-dev lib32z1-dev liblz4-tool libncurses5-dev libsdl1.2-dev libssl-dev libwxgtk3.0-dev libxml2 libxml2-utils lzop pngcrush schedtool squashfs-tools xsltproc zlib1g-dev

**Note:** This package list is taken from the [LineageOS wiki][3] with a few packages taken out that are already included in Ubuntu 18.04. Different versions of Ubuntu might require other packages.

e.g. older versions of Ubuntu didn't ship with `unzip`

### Install The repo Command

Android uses as a wrapper called `repo` for dealing with multiple git repos at the same time. 

This isn't packaged so we create a directory called `bin` in the home folder and download the repo tool into it then make  it executable (runnable).
We then set it as executable and add the bin dir to the `PATH` in the `.bashrc` file so the `repo` command can be called from anywhere.

    mkdir ~/bin
    curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
    chmod +x ~/bin/repo
    echo -e '\nexport PATH=~/bin:$PATH\n' >> ~/.bashrc
    source ~/.bashrc

## Get The TWRP Source

Since we are only compiling TWRP and not a full ROM we will get a minimal set of files to save time and space.

The TWRP source is in OmniROM, however, the `repo` command will download/clone from multiple sources as android source is split over multiple projects.

To simplify this when you are only building TWRP, there is a [minimal manifest project][4] to restrict sources to only downloaded the required projects.
 
Choose the branch that matches the version of android your phone has. In this case we are using `twrp-9.0`.

    mkdir -p ~/android/twrp
    cd ~/android/twrp
    repo init -u git://github.com/minimal-manifest-twrp/platform_manifest_twrp_omni.git --depth=1 -b twrp-9.0
    repo --time sync --no-clone-bundle -j8

You can change the number of threads (simultaneous downloads) by changing `j8` (twice number of CPU cores is a good start).

- `--time` will output the time taken at the end.
- `--depth=1` means only get the last commit and not the entire history (which should reduce the amount downloaded).
- `--no-clone-bundle` will reduce the download further by disabling git bundles that aren't affected by the `-c` option.

This can take a long time. Even with a decent connection it took 1h40m. The `.repo` dir was about 5.2 GB in size and the full twrp dir (including the 5.2 GB .repo) was about 22 GB.

## Extracting Recovery/System img Files

Not all manufacturers provide kernel source (even though they are supposed to under GPL) and even when they do it can be a tricky exercise in itself to get it compile properly.

Therefore, a simpler way to compile TWRP is to extract the system images from your existing working phone.

### Accessing Your Phone Over USB

The first step is to enable Developer options on your phone. This is typically done by going into the `Settings - System - About` menu and tapping the Build Number 7 times.

Once you have done this you will have an extra menu `Settings - System - Developer options`.

Within that menu tick `USB debugging` now when you connect your USB cable the phone should say USB debugging.

You should now be able to see your phone from the command line by typing:

    adb devices

Also tick `OEM Unlocking` so you can unlock the bootloader and flash new images.

Now to unlock the bootloader. 

**THIS WILL FACTORY RESET YOUR PHONE AND WIPE ANY DATA**

    adb reboot bootloader
    fastboot flashing unlock

There will be a warning on screen and instructions to select using the volume buttons.

After you reboot you will go through the basic phone setup options again.

You will have to enable Developer Options in the settings again but this time you should see that `OEM Unlocking` is greyed out as your bootloader is already unlocked.

### MTK Devices

MediaTek devices use a scatter file and work slightly differently.

Download SP Flash Tool from [here](https://spflashtool.com/)

It needs libqtwebkit4

    sudo apt install libqtwebkit4
    
It relies on an old version of `libpng` though and you are likely to get the error:

    ./flash_tool: error while loading shared libraries: libpng12.so.0: cannot open shared object file: No such file or directory

A newer version (16 instead of 12) is now included in Ubuntu but you can download the old version from the Ubuntu repos

    wget -q -O /tmp/libpng12.deb http://mirrors.kernel.org/ubuntu/pool/main/libp/libpng/libpng12-0_1.2.54-1ubuntu1_amd64.deb 
    dpkg -x libpng12.deb /tmp/libpng12
    cp /tmp/libpng12/lib/x86_64-linux-gnu/libpng12.so.0* .
    ln -s libpng12.so.0.54.0 libpng12.so.0
    
    sudo dpkg -i /tmp/libpng12.deb

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

  [1]: https://twrp.me/faq/howtocompiletwrp.html
  [2]: https://forum.xda-developers.com/showthread.php?t=1943625
  [3]: https://github.com/LineageOS/lineage_wiki/blob/master/_includes/templates/device_build.md
  [4]: https://github.com/minimal-manifest-twrp/platform_manifest_twrp_omni
