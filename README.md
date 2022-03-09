# Compile-rom-from-source. 

To build a A10+ (AOSP/LOS based) on Ubuntu 20.04+ (or distros based on it), there are four main steps:
(This guide is applicable for recoveries as well (TWRP, OFRP...))
Working on Android 9, 10, 11 and 12
 
#################################################################
#                  Step 1: Setup your environment               #
#################################################################

****Setup Linux to build Android****

sudo apt update && sudo apt full-upgrade && sudo apt install build-essential ccache libncurses5 libssl-dev m4 unzip zip brotli bc bison curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev liblz4-tool libncurses5-dev libsdl1.2-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev adb fastboot python python3.8 abootimg

If you use a Linux distribution based on Arch, use this instead: https://github.com/akhilnarang/scripts/blob/master/setup/arch-manjaro.sh

****Setup repo****

mkdir ~/bin && PATH=~/bin:$PATH && curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo && chmod a+x ~/bin/repo
git config --global user.name "YourName" && git config --global user.email "you@example.com" && git config --global credential.helper "cache --timeout=7200"

The credential helper's cache saves you logging in each time you want to push your trees to GitHub, and the name and email will appear in the signature of your commits whenever you use the --signoff in your commit command (git commit -m "Example of commit" --signoff)

############################################################
#               Step Two: Download the source              #
############################################################

****Create a folder for your source****

mkdir foldername && cd foldername

When you want to build a ROM, you must download its source. All ROMs will have their source code available on GitHub. To properly download the source, follow these steps:
-- Go to your ROM's Github (e.g. http://github.com/crdroidandroid)
-- Search for a manifest (usually called manifest or android_manifest).
-- Go into the repository and make sure you are in the right branch.
-- Go into the README file and search for a repo init command. 
   Recommended: adding —-depth=1 between repo init and -u in order to make a shadow clone of the source (will reduce drastically the space used).
   Paste it into a terminal under the folder you've just created and hit enter.
-- After the repo has been initialized, run this command in order to download the source properly:
   repo sync --current-branch --no-clone-bundle --no-tags --optimized-fetch --prune --force-sync -j$( nproc --all )
-- This process can take a while depending on your internet connection. Go and have a beer/coffee/nap/the whole day in the meantime! The whole ROM source can take ~65 GB of your disk (a SSD is always recommended).

###########################################################################
#                  Step Three: Download the device trees                  #
###########################################################################

In the ROM folder, there is a hidden folder called .repo.
So, to be able to download the device trees: 

cd .repo
mkdir local_manifests
nano zbuild.xml

There you will have to create your own manifest with your device's repos (usually three: device tree, kernel and vendor). Take a look at this example:

https://github.com/masemoel/local_manifests/blob/master/picasso_q.xml

NAME means the GitHub repository's name, PATH means where you want the repository to be saved, REMOTE... you can ignore this; and REVISION means which branch you want to clone.
Here you have a template:

<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project name="masemoel/android_device_brand_codename" path="device/brand/codename" remote="github" revision="11.x"/>
</manifest>

****Save your manifest at zbuild.xml****

Then repo sync --current-branch --no-clone-bundle --no-tags --optimized-fetch --prune --force-sync -j$( nproc --all ) again (it will take much less than the first time, as you already have almost all your source downloaded 😄).

The next part is to setup the device tree created for the specific ROM you want to build. I won't explain you here how to configure your device tree to a specific Android version (for example, if you have an Android 7.1.2 device tree, you cannot build Android 8 or 9 with it; you need some adaptions), so ensure you're cloning the tree/branch for the desired Android version.

To configure the device tree to build a specific ROM do the following (using the Redmi K30 5G, whose codename is picasso, as an example, and explained by aoleary):
1. cd device/xiaomi/picasso
2. nano AndroidProducts.mk
3. replace havoc_picasso.mk to what the ROM requires: <rom_name>_picasso.mk (check the ROM repo for another device to see what they need. For example, for LineageOS, it should be lineage_picasso.mk, du_picasso.mk for DirtyUnicorns,...)
4. Rename the file havoc_picasso.mk to what the ROM requires: <rom_name>_picasso.mk
5. Within <rom_name>_picasso.mk, change everywhere it says "havoc" to <rom_name>.
6. Also verify that the paths are consistent, within <rom_name>_picasso.mk, specially for the variable:
  # Inherit some common LineageOS stuff.
  $(call inherit-product, vendor/<rom_name>/config/common.mk)

==> You need to browse the ROM vendor repo and go and check the file exists with the same name and that the path exists as well. <==

##################################################
#              Step 4. Build the ROM             #
##################################################

NEW! Android 12 needs this workaround to avoid A12 building errors with ccache https://gist.github.com/masemoel/537301229c1bd4c8b20c50f535751d19

Before starting to build, we are going to turn on ccache to speed up the build process by typing in a terminal:
export USE_CCACHE=1 && export CCACHE_EXEC=/usr/bin/ccache && ccache -M xG (where x means X GB of ccache. Anywhere from 25GB-100GB will result in very noticeably increased build speeds (for instance, a typical 1hr build time can be reduced to 20min). If you’re only building for one device, 25GB-50GB is fine. If you plan to build for several devices that do not share the same kernel source, aim for 75GB-100GB.) This is not noticeable for the first build, but it's very helpful for future builds. As far as I know, this ccache is storaged at /home/user/.ccache .

Allocate Jack enough memory:
export ANDROID_JACK_VM_ARGS="-Dfile.encoding=UTF-8 -XX:+TieredCompilation -Xmx#G" (put your GB of RAM where # is).

The following command fixes some compilation issues when building on Ubuntu 18.04 or higher, so this is a must for us:
export LC_ALL=C 

The building commands are usually individual to each ROM. It's within the same README.md from step 2. For crDroid or any other LineageOS-based, they are:
. build/envsetup.sh
brunch devicecodename

For devices with less than 16GB of RAM - Android 10:

cd build/soong && git fetch https://github.com/Magma-WIP/build_soong ten-metalava && git cherry-pick bcd1bb529132905cf55e72f5a2a6ba19a99f60ac^..dc3365fbde3b2a5773e655f690bb073967100795 && cd ../..
And then:
lunch lineage_picasso-userdebug && mka api-stubs-docs && mka hiddenapi-lists-docs && mka system-api-stubs-docs && mka test-api-stubs-docs && mka bacon

For devices with less than 16GB of RAM - Android 11:
cd build/soong && git fetch https://github.com/masemoel/build_soong_legion-r 11 && git cherry-pick b45c5ae22f74f1bdbb9bfbdd06ecf7a25033c78b && git cherry-pick e020f2130224fbdbec1f83e3adfd06a9764cca87 && cd ../..
And then, run the building commands for your ROM (again, check the README at the manifest of the ROM; there you will see it. LineageOS uses brunch picasso, while for example AEX uses lunch_picasso-userdebug && mka aex).

When the build process has finished, the flashable zip will be saved at /out/target/product/devicecodename

########################################################################################
#                              Success! So... what's next?                             #
########################################################################################

You’ve done it! Welcome to the elite club of self-builders. You’ve built your operating system from scratch, from the ground up. You are the master/mistress of your domain… and hopefully you’ve learned a bit on the way and had some fun too.🥳🥳

****Useful for future builds****

If you've made some important changes ==> . build/envsetup.sh && make clobber (to recompile everything from scratch and avoid weird bugs. Sometimes, you can only wipe out/target/product/codename , but don't abuse of this!).
Keep the source up to date ==> repo sync --current-branch --no-clone-bundle --no-tags --optimized-fetch --prune --force-sync -j$( nproc --all )
For Ubuntu 18.04 users or higher ==> export LC_ALL=C does NOT persist on reboot!!!!
If you have a compilation error and you don't know how to solve it, you can ask here following the rules ==> https://t.me/AndroidBuildersHelp

Take logcat: adb logcat -d > logcat.log
Clear logcat history (it's cleared automatically on each reboot): adb logcat -b all -c

Credits to:
- LineageOS team (as I took some necessary stuff from their guides)
- aoleary (he did the h815 example for setting up the device tree and more)
- nathanchance (this is his guide template)
- masemoel (me)
