---
layout: post
comments: true
title: Patching VMWare on Ubuntu
---

If you recently upgrading Ubuntu to 17.10 (Kernel 4.13.0) then you might just not be able to run your VMWare machines anymore. If you get an "not enough physical memory" error although your system has enough RAM to run the VMs then you will have to patch your VMWare installation like below.

Open a terminal and go to the VMWare modules directory. Then untar `vmmon.tar` and change to the `linux` folder.
{% highlight shell %}
cd /usr/lib/vmware/modules/source
tar xvf vmmon.tar
cd vmmon-only/linux
{% endhighlight %}

Download the patch from github. Note the .diff at the end of the github url.
{% highlight shell %}
wget https://github.com/mkubecek/vmware-host-modules/commit/770c7ffe611520ac96490d235399554c64e87d9f.diff -O patch.diff
{% endhighlight %}

Patch the `hostif.c` file using the patch file `patch.diff`.
{% highlight shell %}
patch hostif.c patch.diff
{% endhighlight %}

Rebuild the patched module.
{% highlight shell %}
cd ../..
tar cf vmmon.tar vmmon-only
sudo vmware-modconfig --console --install-all
{% endhighlight %}
