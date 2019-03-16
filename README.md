# Unprivileged container builds using stacker

One of the primary goals of
[user
namespaces](https://s3hh.wordpress.com/2012/05/10/user-namespaces-available-to-play/)
was to provide the ability for unprivileged users to have their own range of
uids over which they would have privilege, with minimal need for setuid
programs and no risk (barring bugs in the OS) of their privilege having effect
on uids which are not "their own".

We've had user namespaces for awhile now.  While there are some actions which
cannot be done in a user namespace, such as mounting a loopback filesystem,
there are many things, such as setting up a build environment with custom
package installs, which used to be a challenge without privilege but are now
simple.

## Stacker

My friend 
[Tycho](http://www.tycho.ws/) wrote stacker [stacker](https://github.com/anuvu/stacker)
as a tool for building OCI images.  A few of its features include:

* Creates OCI images.
* Can also be used for general software building.
* Supports layer re-use between build stages to minimize redundant I/O and time.
* Supports unprivileged use!

Stacker uses btrfs subvolumes for layer manipulation.  Therefore, before you
can use it unprivileged you do need to - as root - setup a filesystem where
you can play.  This is just any btrfs mounted with user_subvol_rm_allowed option,
and with a directory owned by you.

If you're starting with an Ubuntu bionic image (for instance, you've just done a

```bash
ubuntu@stacker:~/butrfs/lxc$ uvt-kvm create --memory 4096 --disk 30 --unsafe-caching lxc1 release=bionic arch=amd64
```

and logged into it), begin by installing stacker:

```bash
ubuntu@stacker:~/butrfs/lxc$ sudo snap install --classic go
ubuntu@stacker:~/butrfs/lxc$ sudo apt install ubuntu-dev-tools libgpme-dev libcap-dev acl-dev liblxc-dev btrfs-tools
ubuntu@stacker:~/butrfs/lxc$ git clone git://github.com/anuvu/stacker
ubuntu@stacker:~/butrfs/lxc$ cd stacker
ubuntu@stacker:~/butrfs/lxc$ make
ubuntu@stacker:~/butrfs/lxc$ mkdir ~/bin
ubuntu@stacker:~/butrfs/lxc$ cp stacker ~/bin
ubuntu@stacker:~/butrfs/lxc$ export PATH=~/bin:$PATH
```

If you do not already have a suitable btrfs (for instance as your rootfs), you
can create one:

```bash
ubuntu@stacker:~/butrfs/lxc$ truncate -s 200G btrfs.disk
ubuntu@stacker:~/butrfs/lxc$ mkfs.btrfs btrfs.disk
ubuntu@stacker:~/butrfs/lxc$ mkdir btrfs
ubuntu@stacker:~/butrfs/lxc$ sudo mount -o loop,user_subvol_rm_allowed btrfs.disk btrfs
ubuntu@stacker:~/butrfs/lxc$ sudo chown -R ubuntu: btrfs/
```

Let's try a little experiment.  Lets' try doing a build of the lxc git
tree, using the latest Ubuntu packaging.  This will yield us some
debs which we can install on a test system.  First, clone my demo
tree and look at the ```stacker.yaml```, the file which drives the build.

```bash
ubuntu@stacker:~/butrfs/lxc$ cd btrfs
ubuntu@stacker:~/butrfs/lxc$ git clone git://github.com/hallyn/stacker-demo
ubuntu@stacker:~/butrfs/lxc$ cd stacker-demo/lxc
ubuntu@stacker:~/butrfs/lxc$ cat stacker.yaml
```

We'll need a copy of the lxc git repo.  We could have stacker download it,
but let's just download it once on the host and have stacker import it:

```bash
ubuntu@stacker:~/butrfs/lxc$ git clone git://github.com/lxc/lxc
```

And then let's build

```bash
ubuntu@stacker:~/btrfs/lxc$ stacker build
building image lxc...
importing files...
Getting image source signatures
Skipping blob 898c46f3b1a1 (already present): 30.96 MiB / 30.96 MiB [=======] 0s
Skipping blob 63366dfa0a50 (already present): 851 B / 851 B [===============] 0s
Skipping blob 041d4cd74a92 (already present): 545 B / 545 B [===============] 0s
Skipping blob 6e1bee0f8701 (already present): 162 B / 162 B [===============] 0s
Skipping blob 898c46f3b1a1 (already present): 30.96 MiB / 30.96 MiB [=======] 0s
Skipping blob 63366dfa0a50 (already present): 851 B / 851 B [===============] 0s
Skipping blob 041d4cd74a92 (already present): 545 B / 545 B [===============] 0s
Skipping blob 6e1bee0f8701 (already present): 162 B / 162 B [===============] 0s
Copying config 885f8bbcf9de: 2.42 KiB / 2.42 KiB [==========================] 0s
Writing manifest to image destination
Storing signatures
unpacking to /home/ubuntu/btrfs/lxc/roots/.working
2019/03/16 04:13:48  warn rootless{dev/full} creating empty file in place of device 1:7
2019/03/16 04:13:48  warn rootless{dev/null} creating empty file in place of device 1:3
2019/03/16 04:13:48  warn rootless{dev/ptmx} creating empty file in place of device 5:2
2019/03/16 04:13:48  warn rootless{dev/random} creating empty file in place of device 1:8
2019/03/16 04:13:48  warn rootless{dev/tty} creating empty file in place of device 5:0
2019/03/16 04:13:48  warn rootless{dev/urandom} creating empty file in place of device 1:9
2019/03/16 04:13:48  warn rootless{dev/zero} creating empty file in place of device 1:5
running commands...
running commands for lxc
+ cd /root
+ git clone /stacker/lxc
/stacker/.stacker-run.sh: line 3: git: command not found
error: run commands failed: execute failed: exit status 127
```

That failed, because git was not found.

Let's try it by hand!

```
ubuntu@stacker:~/butrfs/lxc$ stacker chroot
#
```

This provides a shell in a running container in the build environment.  Actually, while
we're doing this, let's confirm that this is unprivleged.  Run ```sleep 200```
in the stacker chroot, and log into the host/VM over another terminal.  There,
do

```bash
# ps -ef | grep sleep
ubuntu   11526 11522  0 19:45 pts/2    00:00:00 sleep 200
ubuntu   11528 11481  0 19:45 pts/1    00:00:00 grep --color=auto sleep
```

Notice how it is in fact running as our regular 'ubuntu' user.  You can also

```
# cat /proc/self/uid_map
```

inside the stacker chroot for more information.  Anyway, in
the stacker chroot window, kill the sleep, and do

```
# which autoreconf
```

No autoreconf.  We need to add some packages before doing the build.  But we
may need to do the build repeatedly while we fix up errors in the code.  So
let's set up a build container with the needed dependencies, which can remain
unchanged across several builds, and use that as the basis for another
container in which we will do the build.  Take a look at stacker-2.yaml.  I'll
refrain from walking through it here - it should be quite self-explanatory.  It
builds a container called 'build' first, with the required software
dependencies installed.  Then on top of that it builds an 'lxc' container in
which to do our package build.  If we have to redo the package build a few
times, this will save us the time of reintalling the dependencies

```
ubuntu@stacker:~/butrfs/lxc$ stacker build -f stacker-2.yaml
```

This time it worked.  We'd like to get the packages it built, of course.  We can
look at them in a stacker chroot:

```
ubuntu@stacker:~/butrfs/lxc$ stacker chroot -f stacker-2.yaml
# ls /root/*.deb
/root/liblxc-common_3.0.3-0ubuntu1_amd64.deb  /root/liblxc1_3.0.3-0ubuntu1_amd64.deb
/root/lxc-dev_3.0.3-0ubuntu1_all.deb      /root/lxc1_3.0.3-0ubuntu1_all.deb
/root/liblxc-dev_3.0.3-0ubuntu1_amd64.deb     /root/libpam-cgfs_3.0.3-0ubuntu1_amd64.deb
/root/lxc-utils_3.0.3-0ubuntu1_amd64.deb  /root/lxc_3.0.3-0ubuntu1_all.deb
# dpkg --contents /root/liblxc-common_3.0.3-0ubuntu1_amd64.deb
...
```

We can also easily exctract the packages from the built container and
onto the host:

```
ubuntu@stacker:~/butrfs/lxc$ stacker grab lxc:/root/lxc-dev_3.0.3-0ubuntu1_all.deb
```

Or, if it were useful, we could save the image to an image store, by doing
something like:

```
ubuntu@stacker:~/butrfs/lxc$ skopeo copy oci:oci:lxc docker://my-registry/lxc-1
```
