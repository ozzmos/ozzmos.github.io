---
layout: post
title: Building Android AOSP for Nexus devices
authors: Ozzmos
tags: android
---


The idea of building Android from sources came to me after trying to flash my Nexus 7 (2013) with Ubuntu Touch.
I spent two days on this with no luck. For a device which is considered to be a reference device, it's too bad.

However I didn't give up and decided to give a try for my own AOSP build still in the idea of taking my data back from
web giants(Google, Apple…).
<!--more-->

This is a tutorial for Nexus devices on a fresh Ubuntu 16.04.


Install essential packages

    sudo apt-get install git ccache automake lzop bison gperf \
    build-essential zip curl zlib1g-dev zlib1g-dev:i386 g++-multilib \
    python-networkx libxml2-utils bzip2 libbz2-dev libbz2-1.0 \
    libghc-bzlib-dev squashfs-tools pngcrush schedtool dpkg-dev \
    liblz4-tool make optipng maven
    
Install the java version you need.

Some ROMs (Android Lollipop / CM 12.1 and below) require OpenJDK 7.
 Marshmallow / CM 13 and above seems to require OpenJDK 8. 
 Some older devices like Nexus 7 (2013) still require openjdk-7-jdk.
 
 If, like me, you need to install openjdk-7 on Ubuntu 16.04, you will need to install it via a PPA.
 
Remove existing java versions
  
     sudo apt-get remove openjdk-* icedtea-* icedtea6-*
   
For openjdk 8 and 9

    sudo apt-get install openjdk-8-jdk

or 

    sudo apt-get install openjdk-9-jdk
    
For openjdk 7
   
     sudo add-apt-repository ppa:openjdk-r/ppa
     sudo apt-get update && sudo apt-get install openjdk-7-jdk

Get the «repo» script which fetch Android code in a «bin» directory at the root of your home folder

    mkdir ~/bin && curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo && chmod a+x ~/bin/repo
    
Add repo to the PATH: open .bashrc with nano and add some lines

    nano ~/.bashrc

    export PATH=~/bin:$PATH
    # CC_CACHE is used to speed up future compilations
    export USE_CCACHE=1 
    
Reinitialize bash

    source ~/.bashrc
    
Create a directory for Android source code

    mkdir ~/android
    cd ~/android

Specify a git identity

    git config --global user.email "john.doe@example.com"
    git config --global user.name "John Doe"

Get the source code (takes a lot of time, many gigabytes). You can change branch with -b option

Check the branch list :
https://source.android.com/source/build-numbers.html#source-code-tags-and-builds

For my Nexus 7, I will use the branch android-6.0.1_r59 (6.0.1 with Augustus security patches)

    repo init -u https://android.googlesource.com/platform/manifest -b android-6.0.1_r59
    repo sync
    
Download proprietary binaries for your hardware to be recognized.

Nexus users can download at this address :
https://developers.google.com/android/nexus/drivers

Execute each script, then copy the «vendor» directory at the root of the ~/android folder

### Build 

Set up environment

    source build/envsetup.sh
    
Choose a target, here aosp_flo-userdebug - flo is codename for Nexus7 (2013)

    lunch aosp_flo-userdebug

Set paths for products

    setpaths
    
Check paths

    env | grep ANDROID
    
Build the code (for 4 cores)

    make -j4
    
If you see in the terminal «Build finished» congrats ! You made it !
You will be able to test it on your Nexus device but first you need to get access to your device.

### Getting access to your device

Link your device with usb then scan devices with adb command

    adb devices
    
If you see «no permissions», you need to create a permission file.

Check usb id, for Nexus 7

    lsusb
    Bus 003 Device 006: ID 18d1:4ee7 Google Inc.
    
Create a permission file

    sudo nano /etc/udev/rules.d/99-android.rules
 
with content below. Don't forget to change idProduct and idVendor values according to your device
and OWNER and GROUP values according to your account's name

    # Nexus7
    SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", ATTR{idProduct}=="4ee7",  MODE="0666", OWNER="ozzmos"
    # Google default
    SUBSYSTEM=="usb", ATTR{idVendor}=="18d1", MODE="0666", GROUP="ozzmos"
    
You need to restart your pc to validate theses modifications.

    
### Flash the device

    adb reboot bootloader

    sudo -s
    source build/envsetup.sh
    lunch aosp_flo-userdebug
    setpaths
    fastboot flashall -w
 
If all works well, you can reboot your device and you should see the Android logo.
It takes some time to initialize apps, so be patient !

### Bug fix

Fix «unsupported reloc 43»

    nano ~/android/art/build/Android.common_build.mk
 
Change these lines

    ART_HOST_CLANG := false
    ifneq ($(WITHOUT_HOST_CLANG),true)
    
with

    ART_HOST_CLANG := false
    ifneq ($(WITHOUT_HOST_CLANG),false)


Fix «unsupported reloc 42»

    ln -sf /usr/bin/ld.gold /home/ozzmos/android/prebuilts/gcc/linux-x86/host/(glibc version)/x86_64-linux/bin/ld
    
To know which gcc version you use

    ldd --version