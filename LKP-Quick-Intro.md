# Linux Kernel Live Patching – Quick Introduction

Linux Kernel Live patching allows patching of Kernel at run time. Yes at run time without need to reboot the system to upgrade it with the newer Kernel or in simple terms rebootless patching. Why it’s one of the cool features, consider the most common use case of security issues, those gets discovered and requires immediate upgrade of the Kernel. These upgrades usually calls for a scheduled downtime, mostly when number of user are minimally impacted and with additional overhead of stopping/redirecting of services handled by the system. With Live patching, which is a hybrid approach taking the best of both Kpatch/Kgraft worlds to rebootless patch of the Kernel. This article is limited to how to use. The technical details behind Live patching will be published in following articles.

The system I am using is Ubuntu 15.04 (vivid)

```
$ cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=15.04
DISTRIB_CODENAME=vivid
DISTRIB_DESCRIPTION="Ubuntu 15.04"
```

Livepatch got merged into v4.0, if the current kernel installed on the system is >= 4.0.
You might want to skip this step. To find which kernel version your system is currently running using

```
$ uname -r
3.19.0-49-generic
```

I need to compile and boot the latest Kernel or any Kernel 4.0

```
1. git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/ linux (or just get the tar bar from kernel.org)
2. cp /boot/config-$(uname -r) .config
3. make localmodconfig (Make sure following config are enabled)
DYNAMIC_FTRACE_WITH_REGS=y
HAVE_LIVEPATCH=y
SAMPLES=y
SAMPLE_LIVEPATCH=m
Last two configs are need for building the sample livepatch modules.
4. make modules_install
5. make install
```

Reboot into the latest compiled kernel, In my case it 4.5.0-rc2.
### Live Patching

Linux Kernel sources hosts sample live patch Kernel module, which is best starting point to build module with complex patch, modifying code at different levels. The sample modules should available, if the Kernel was compiled with `SAMPLE_LIVEPATCH=m` in previous step. The compiled sample module, will be available at `<kernelsources>/samples/livepatch/livepatch-sample.ko`

Sample livepatch module, does a simple patching of function `cmdline_proc_show()`. This function is called when ever `cat /proc/cmdline` is read to print the boot parameters of the booted kernel, like root filesystem and others. On my Debian guest, when I read the `/proc/cmdline`:

```
root@debian-amd64:~# cat /proc/cmdline
root=/dev/sda1 ro console=ttyS0,9600
```

Livepatch sample module should be first inserted to Kernel, like any other Kernel module. I am using `insmod` command

```
# insmod ./livepatch-sample.ko
# dmesg |tail -3
[ 166.881759] livepatch_sample: module verification failed: signature and/or required key missing - tainting kernel
[ 166.921876] livepatch: tainting kernel with TAINT_LIVEPATCH
[ 166.923943] livepatch: enabling patch 'livepatch_sample'
```

`dmesg` should display above lines, once kernel was able to successfully patch the function `cmdline_proc_show()` with the newer definition in Live patch sample module. Warnings about TAINT can be safely ignored and is out of scope of this article. Re-running the cat command to read the cmdline file, should re-direct to new function provided by the sample module.

```
# cat /proc/cmdline
this has been live patched
```

This should get you started with the Live patching.
