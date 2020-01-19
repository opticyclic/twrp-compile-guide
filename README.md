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

  [1]: https://twrp.me/faq/howtocompiletwrp.html
  [2]: https://forum.xda-developers.com/showthread.php?t=1943625
  [3]: https://github.com/LineageOS/lineage_wiki/blob/master/_includes/templates/device_build.md
  [4]: https://github.com/minimal-manifest-twrp/platform_manifest_twrp_omni
