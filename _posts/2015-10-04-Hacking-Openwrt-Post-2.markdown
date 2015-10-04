---
layout: post
title:  "Openwrt on Virtualbox"
date:   2015-10-04 13:33:35
author: Gaurav Suryagandh
categories: Openwrt
---

In this blog, we will see how to compile and boot openwrt inside virtualbox. Before going into the steps, let me explain the problem i was trying to solve. I wanted to try out few modification in openwrt code and wanted to try it before flashing my TP-Link router. I also wanted to compile lighttpd with openwrt image. This was the motivation behind compiling/running openwrt inside virtualbox. I tried it with Virtual machine manager also and it works the same way. I am assuming that we have a copy of openwrt source code. 

step#1 Make Menuconfig. Choose below settings in menuconfiuration.
  
We can choose any target system but its a good idea to choose x86 as it has better driver support with underlying virtualization technology.
{% highlight ruby %}
Target System : x86 
Subtarget     : x86_64
{% endhighlight%}

As we are compiling it for virtual machine, target image needs to be vmdk or vdi. 

{% highlight ruby %}
Target Image: Vmdk
Target Image: Vdi
{% endhighlight%}

If we choose default size for kernel and rootfs, we will not be able to edit / partitions on openwrt router, as firmware will eat all the space. To increase the size of above mentioned partitions, we will need to set these values in target image section

{% highlight ruby %}
Kernel partition size in MB : 16
RootFS partition size in MB : 64 or 128
{% endhighlight%}

step#2 Execute make. In case any errors, check build.log. If you have multi core machine, you can distribute make job across available cores by specifying -j [no_of_cores].

{% highlight ruby %}
sudo make V=s 2>&1 | tee build.log | grep -i error
{% endhighlight %}

step#3 After successful completion of above step, new firmware will be available under `bin` directory. 

step#4 Open Virtualbox. Create a virtual machine with existing virtual hard drive.

This should give us a openwrt virtual machine. In the next blog, we will see how we can turn this virtual machine into router/bridge. 
