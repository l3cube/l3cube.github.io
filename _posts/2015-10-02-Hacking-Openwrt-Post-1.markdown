---
layout: post
title:  "Compiling Openwrt from source code"
date:   2015-10-02 07:33:35
author: Gaurav Suryagandh
categories: Openwrt
---


This is the first post in the 'Hacking OpenWrt' series. I am playing with openwrt for last couple of days. Essentially I am trying to compile customized packages for openwrt and link the functionality with openwrt source code.This includes compiling third party tools and changing their configuration as well.

Though there is lot of documentation available for openwrt. Most of it is related to precompiled firmware images. In this series,I will cover openwrt compilation from source code, creating vdi image, booting openwrt inside virtualbox, linking lighttpd with openwrt and turning your vm into a openwrt router. 

Let's start with OpenWrt source code compilation and customization. Lets take an example of compiling opensourcode for "Qualcom Atheros" board. 

step#1 Download openwrt source code from Openwrt Github page
 
{% highlight ruby %}
git clone git://git.openwrt.org/openwrt.git
{% endhighlight %}

step#2 Run make menuconfig inside source directory

{% highlight ruby %}
cd openwrt

sudo make menuconfig
{% endhighlight %}

step#3 Select configurations for target system. Openwrt lets you select few predefined configurations for well known systems.In this case we are selecting "Qualcomm Atheros" as target system. Save and exit menuconfig wizhard. 

step#4 Execute make. In case any errors, check build.log. If you have multicore machine, you can distribute make job across available cores by specifying -j [no_of_cores].

{% highlight ruby %}
sudo make V=s 2>&1 | tee build.log | grep -i error
{% endhighlight %}

step#5 After successful completion of above step, new firmware will be available under `bin` directory. 
