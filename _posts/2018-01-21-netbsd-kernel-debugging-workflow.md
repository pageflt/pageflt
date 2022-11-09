---
layout: post
title: "Working with the NetBSD kernel"
permalink: "/working-with-the-netbsd-kernel/"
---

## Overview

When working on complex systems, such as OS kernels, your attention span and cognitive energy are too valuable to be wasted on inefficiencies pertaining to ancillary tasks. After experimenting with different environmental setups for kernel debugging, some of which were awkward and distracting from my main objectives, I have arrived to my current workflow, which is described here. This approach is mainly oriented towards security research and the study of kernel internals.

Before delving into the details, this is the general outline of my environment:

- My host system runs Linux. My target system is a QEMU guest.
- I'm tracing and debugging on my host system by attaching GDB (with NetBSD x86-64 ABI support) to QEMU's built-in GDB server.
- I work with NetBSD-current. All sources are built on my host system with the cross-compilation toolchain produced by `build.sh`.
- I use NFS to share the source tree and the build artifacts between the target and the host.
- I find IDEs awkward, so for codebase navigation I mainly rely on `vim`, `tmux` and `ctags`.
- For non-intrusive instrumentation, such as figuring out control flow, I'm using `dtrace`.


## Preparing the host system

For starters, there are few things that must be configured on the host system.

### QEMU

Make sure your host system has a recent version of QEMU with support for the target architecture of your choice. The rest of this post will be dealing with x86-64 NetBSD guests. Also, make sure your system is configured for [bridged-mode networking](/basic-networking-with-qemu/){:target="_blank"}, it will make debugging and working with QEMU guests much more convenient.

### GDB
You will need a version of GDB with support for NetBSD x86_64 ABI. Most likely you won't find it in your platform's package repository, so you'll have to compile it yourself. For example:

```
$ wget http://ftp.gnu.org/gnu/gdb/gdb-8.0.tar.xz
$ tar -xf gdb-8.0.tar.xz && cd gdb-8.0
$ sudo mkdir -p /opt && sudo chmod 755 /opt
$ ./configure --prefix=/opt --target=x86_64-netbsd
$ make -j4 && sudo make install
```

Assuming you had all the required dependencies for the build, you should now have a GDB version with NetBSD x86-64 ABI support at `/opt/bin/x86_64-netbsd-gdb`. You might want to add this to your `$PATH` or symlink it to something more memorable.

### NFS exports

In order to avoid manually copying files around, it's a good idea to set up a directory structure on the host system for the source tree, the toolchain and the build artifacts, and share it with the guest over NFS.

Here's the directory structure that I use:

```
~/Code/netbsd-current			# The build root
~/Code/netbsd-current/destdir		# Userland binaries' location after compilation
~/Code/netbsd-current/objdir		# Build artifacts go here
~/Code/netbsd-current/releasedir	# Release files (created from `destdir`)
~/Code/netbsd-current/src		# Full source tree
~/Code/netbsd-current/tooldir		# Cross-compilation toolchain
```

Once the structure of the build root is created, export it, and restart your NFS server. In my case, I export `~/Code/netbsd-current`. This approach will keep everything neat and separate, without polluting the entire source tree with object files.


## Building NetBSD-current

### A word of warning

Now is a great time to familiarize yourself with the [build.sh tool and its options](https://www.netbsd.org/docs/guide/en/chap-build.html#chap-build-environment-options){:target="_blank"}. Be especially carefull with the following options:

```
    -r          Remove contents of TOOLDIR and DESTDIR before building.
    -u          Set MKUPDATE=yes; do not run "make clean" first.
		Without this, everything is rebuilt, including the tools.
```

Chance are, you **do not** want to use these options once you've successfully built the cross-compilation toolchain and your entire userland, because building those takes time and there aren't many good reasons to recompile them from scratch. Here's what to expect:

- On my desktop, running a quad-core Intel i5-3470 at 3.20GHz with 24GB of RAM and underlying directory structure residing on a SSD drive, the entire process took about 55 minutes. I was running `make` with  `-j12`, so the machine was quite busy.
- On an old Dell D630 laptop, running Intel Core 2 Duo T7500 at 2.20GHz with 4GB of RAM and a slow hard drive (5400RPM), the process took approximatelly 2.5 hours. I was running `make` with `-j4`. Based on the temperature alerts and CPU clock throttling messages, it was quite a struggle.

### Acquiring the sources

Install `cvs` if your haven't done so already, configure `CVS_RSH` and `CVSROOT`, start the checkout process and grab a cup of coffee. At the moment of this writing, the entire source tree is around 2.6GB, so you'll have to wait. I'm based in the Netherlands, so I use the closest CVS mirror, which is in France:

```
$ export CVSROOT="anoncvs@anoncvs.fr.NetBSD.org:/pub/NetBSD-CVS"; export CVS_RSH="ssh"
$ cd ~/Code/netbsd-current
$ cvs checkout -PA src
```

### Compiling the sources

Once the checkout process is complete, you can start the lengthy process of building the toolchain and the system. I do not customize the kernel at this point, I build everything with the vanilla configuration and generate an ISO image from which I will provision my guest later on:

```
$ cd ~/Code/netbsd-current/src
$ # Take a nap, grab a coffee, go on a holiday. This is going to take some time.
$ ./build.sh -m amd64 -T ../tooldir -D ../destdir \
	     -R ../releasedir -O ../objdir \
             -U -j6 release iso-image
```

Once the build has completed successfully, this is what you get:
- Bootable ISO image under `releasedir/images/`
- Installation sets under `releasedir/amd64/`
- Cross-compilation toolchain under `toolchain/`, to be used by `build.sh` for builds performed on your host system. As long as you have the compiler set installed, you don't need it for builds from within your guest.

Depending on your system's specs and whether you value disk space or computational time, you can also backup your `tooldir` somewhere outside the build root.


## Preparing the guest system

### Provisioning your guest

You can use the ISO image generated earlier to build your target system:

```
$ mkdir ~/vhd
$ qemu-img create ~/vhd/netbsd-current.img 10G
$ qemu-system-x86_64 -drive file=~/vhd/netbsd-current.img,format=raw \
                     -cdrom ~/Code/netbsd-current/releasedir/images/NetBSD-8.99.12-amd64.iso \
                     -device e1000,netdev=net0 -netdev bridge,id=net0,br=br0 -boot d \
                     -m 1024 -enable-kvm
```

I tend to install my guests roughly as follows:

- Use CD/DVD media as the installation medium
- Install all the sets except "Games", "X11 sets", "Source and debug sets"
- Configure network to use DHCP
- Set the timezone
- Set the root password
- Enable `sshd`. Disable `cgd`, `raidframe`
- Add a regular user, who is also part of the `wheel` group
- Drop to shell and power-down the VM instance

Once the installation of the guest system is complete, all subsequent QEMU invocations should be in the following format:

```
$ qemu-system-x86_64 -drive file=~/vhd/netbsd-current.img,format=raw \
                     -device e1000,netdev=net0 -netdev bridge,id=net0,br=br0 \
                     -m 1024 -enable-kvm -nographic -s
```

You might want an alias for that, or shell script wrapper.

***Note:** If you don't see your guest's console output in QEMU's standard output, you'll have to manually swtch the console device to `com0` in your `/boot.cfg` by adding `consdev com0;` to your first boot entry. Refer to `boot(8)` for more details.*{:.warning}

### Pkgin and NFS shares

The base system is quite... basic, so eventually you'll want some packages. There's the archaic `pkg_add`, but all the mentally stable cool kids nowadays use `pkgin`.

```
$ su -
# export PKG_URL="http://cdn.netbsd.org/pub/pkgsrc/packages/NetBSD/amd64/8.0_current/All"
# pkg_add "$PKG_URL/pkgin-0.9.4nb6.tgz"
# echo $PKG_URL > /usr/pkg/etc/pkgin/repositories.conf
# pkgin update
```

You should also mount your host system's NFS shares. First, ensure they are properly exported and visible from your guest:

```
$ showmount -e [HOST IP ADDRESS]
```

Next, add the appropriate exports to `/etc/fstab` with `rw` permissions, run `mount -a`, and now you should be able to see the source tree and the build artifacts under the designated mount point on your guest. I tend to mount the build root within my guests at the same path as it is on my host system: `~/Code/netbsd-current/`.


### Tailoring the kernel for debugging

On your host system, make a copy of the GENERIC configuration:

```
$ cd ~/Code/netbsd-current/src/sys/arch/amd64/conf
$ cp GENERIC QEMU
```

For source-level debugging and DTrace support you  should at the very least have the following options enabled:

```
makeoptions     DEBUG="-g"      # compile full symbol table for CTF
options         KDTRACE_HOOKS   # kernel DTrace hooks
```

Neither the in-kernel debugger nor the in-kernel KGDB stub are of any use in this scenario, so make sure they're disabled:

```
#options        DDB                     # in-kernel debugger
#options        DDB_COMMANDONENTER="bt" # execute command when ddb is entered
#options        DDB_ONPANIC=1           # see also sysctl(7): `ddb.onpanic'
#options        DDB_HISTORY_SIZE=512    # enable history editing in DDB
#options        KGDB                    # remote debugger
#options        KGDB_DEVNAME="\"com\"",KGDB_DEVADDR=0x3f8,KGDB_DEVRATE=9600
```

Once you're happy with your kernel configuration, it needs to be built. I'm building everything on my host system, so naturally I'm using `build.sh`:

```
$ cd ~/Code/netbsd-current/src
$ ./build.sh -m amd64 -T ../tooldir -D ../destdir \
             -R ../releasedir -O ../objdir -U -u -j6 kernel=QEMU
```

Depending on your system's specs and your kernel's configuration, that shouldn't take more than few minutes.

### Installing the new kernel

Once the build is complete, you can find the new kernel under `objdir/sys/arch/amd64/compile/QEMU/netbsd` in the shared directory tree. Jump into your guest system, backup the old kernel and install the new one:

```
# cp /netbsd /netbsd.old
# cp objdir/sys/arch/amd64/compile/QEMU/netbsd /netbsd
```

### Configuring DTrace

Add the following modules to `/etc/modules.conf` so you can use DTrace within your guest:
```
solaris
dtrace
dtrace_sdt
dtrace_fbt
dtrace_lockstat
dtrace_profile
dtrace_syscall
```

Finally, reboot your guest. Assuming you didn't mess up the kernel configuration, your guest's kernel should be ready for debugging.


## Debugging the guest's kernel

Now that the groundwork has been laid, you can simply load the kernel binary with the debugging symbols into your host's `gdb`, and attach it to the live kernel on the QEMU instance:

```
$ cd ~/Code/netbsd-current/objdir/sys/arch/amd64/compile/QEMU
$ /opt/bin/x86_64-netbsd-gdb ./netbsd.gdb
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0xffffffff8021d16e in x86_stihlt ()
(gdb) continue
```

Here are few useful GDB tips, if you haven't used it much before:

- You can interrupt the kernel execution at any time by invoking `Ctrl+C` in your debugger. You can continue the execution flow through `continue`/`c`.
- You can add breakpoints at most memory addresses, function names or line numbers within source files:
```
# Adding breakpoints on function names:
(gdb) break sys_open
Breakpoint 1 at 0xffffffff804b267f:
  file /home/dimitris/Code/netbsd-current/src/sys/kern/vfs_syscalls.c, line 1672.
(gdb) c
Continuing.
# Adding breakpoints on specific code lines within source files:
(gdb) break /home/dimitris/Code/netbsd-current/src/sys/kern/vfs_syscalls.c:1672
Breakpoint 2 at 0xffffffff804b267f:
  file /home/dimitris/Code/netbsd-current/src/sys/kern/vfs_syscalls.c, line 1672.
(gdb)
```
- You can single-step the execution using the `s` command
- You can look-up struct definitions:
```
(gdb) ptype struct mbuf
type = struct mbuf {
    struct m_hdr m_hdr;
    union {
        struct {...} MH;
        char M_databuf[456];
    } M_dat;
}
```
- GDB can pretty-print internal kernel data structures
```
(gdb) p uap
$4 = (const struct sys_open_args *) 0xffff800032da7f00
(gdb) p/x *uap
$5 = {path = {pad = 0x7f7fff9b46b0, le = {datum = 0x7f7fff9b46b0},
      be = {pad = 0xffff800032da7f00, datum = 0x7f7fff9b46b0}},
      flags = {pad = 0x0, le = {datum = 0x0},
      be = {pad = {0x0, 0x0, 0x0, 0x0}, datum = 0x0}}, 
      mode = {pad = 0x1b6, le = {datum = 0x1b6},
      be = {pad = {0xb6, 0x1, 0x0, 0x0}, datum = 0x0}}}
(gdb) p uap->path->le->datum
$6 = 0x7f7fff9b46b0 "/etc/pam.d/cron"
```

Describing GDB is beyond the scope of this article. Refer to the [official GDB documentation](https://www.gnu.org/software/gdb/documentation/){:target="_blank"} to find out the many ways that GDB can make your life easier.


## Additional workflow tips

- Use `vim` and `ctags` instead of `grep` for symbol lookup. It's much more efficient. Also learn how to use markers, buffers and windows within `vim`.
- To avoid namespace pollution, run `ctags -R .` in the root of your kernel (that is, in `src/sys/`) instead of the root of the entire source tree (`src/`).
- Add the following parameters to your `.vimrc`, so it can use `ctags` no matter how deep in the source tree you are:
```
set autochdir
set tags=tags;
```
- Use `tmux` to avoid getting lost in all those open terminals.
- Learn some basic DTrace scripting with `fbt` and `syscall` providers. It will save you time and effort down the road.
- If you can't find information on a specific aspect of the NetBSD kernel, check the documentation, maling lists and books written about the other BSD-derived systems. They all have "BSD" in their name for a reason.


## Further Reading:
- [NetBSD Documentation: Chapter 30. Obtaining the sources](https://www.netbsd.org/docs/guide/en/chap-fetch.html){:target="_blank"}
- [NetBSD Documentation: Chapter 31. Crosscompiling NetBSD with build.sh](https://www.netbsd.org/docs/guide/en/chap-build.html){:target="_blank"}
- [NetBSD Documentation: Kernel](http://www.netbsd.org/docs/kernel/){:target="_blank"}
- [NetBSD Tutorials: How to enable and run DTrace](https://wiki.netbsd.org/tutorials/how_to_enable_and_run_dtrace/){:target="_blank"}
- [FreeBSD Wiki: The DTrace One-Liner Tutorial](https://wiki.freebsd.org/DTrace/Tutorial){:target="_blank"}
- [Debugging with GDB](https://sourceware.org/gdb/current/onlinedocs/gdb/){:target="_blank"}
