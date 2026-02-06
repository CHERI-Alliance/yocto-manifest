# CHERI repo Manifests for the Yocto Project Build System

This repository provides repo manifests to setup the Yocto build
system for the CHERI QEMU platform.

The [Yocto Project](https://www.yoctoproject.org/) allows the creation of custom Linux distributions
for embedded systems.  It is a collection of git repositories known as
*layers* each of which provides *recipes* to build software packages
as well as configuration information.

[Repo](https://gerrit.googlesource.com/git-repo) is a tool that enables the management of many git repositories
given a single *manifest* file.  Tell repo to fetch a manifest from
this repository and it will fetch the git repositories specified in
the manifest and, by doing so, setup a Yocto Project build environment
for you!

## Getting Started


### 1. Build Host preparation

The [Build Host](https://docs.yoctoproject.org/ref-manual/terms.html#term-Build-Host) needs to meet a number of requirements to be able to build Yocto. These are listed in the [System Requirements](https://docs.yoctoproject.org/current/ref-manual/system-requirements.html#system-requirements) of the Yocto [Reference Manual](https://docs.yoctoproject.org/current/ref-manual/index.html). In particular the commands for installing the necessary development packages are given for some Linux distributions in the [Required Packages for the Build Host](https://docs.yoctoproject.org/current/ref-manual/system-requirements.html#required-packages-for-the-build-host).


### 2. Install Repo

Many distributions include repo, so if you don't have it installed already you might be able to install it using the distribution's package manager:

```shell
# Debian/Ubuntu.
% sudo apt-get install repo

# Gentoo.
% sudo emerge dev-vcs/repo

# Redhat/Rocky.
% sudo dnf install repo
```

Alternatively, you can install it manually:

1. Download the Repo script:

    ``` shell
    % curl http://storage.googleapis.com/git-repo-downloads/repo > repo
    ```

2. Make it executable:

    ``` shell
    % chmod a+x repo
    ```

3. Move it on to your system path:

    ```shell
    % sudo mv repo /usr/local/bin/
    ```

If it is correctly installed, you should see a Usage message when invoked
with the help flag.

``` shell
% repo --help
```

### 3. Initialize Working Directory

Create an empty directory to hold your working files:

``` shell
% mkdir yocto
% cd yocto
```

Tell Repo to download the manifest:

``` shell
% repo init \
    --manifest-url=https://github.com/CHERI-Alliance/yocto-manifest.git \
    --manifest-name=codasip-scarthgap.xml
```

A successful initialization will end with a message stating that Repo is
initialized in your working directory. Your directory should now
contain a .repo directory where repo control files such as the manifest are
stored but you should not need to touch this directory.

> [!NOTE]
> The **codasip-scarthgap** manifest describes a set of git repositories suitable for building the Scarthgap release (5.0) of the Yocto Project's poky distribution, with the changes to support building this with CHERI support.

To learn more about repo, look at [Repo Command Reference](https://source.android.com/source/using-repo "Using repo")

### 4. Fetch all the repositories

``` shell
% repo sync
```

### 5. Initialize the Yocto Project Build Environment

``` shell
% source ./meta-cheri/setup-qemu.sh
```

This creates the [Build directory](https://docs.yoctoproject.org/ref-manual/terms.html#term-Build-Directory)  structure and sets up some environment variables.

Inside the build directory is the [Configuration directory](https://docs.yoctoproject.org/ref-manual/terms.html#term-Configuration-File) called `conf`. Files in this directory can be used to customise the build, so you may wish to edit these configuration files for your specific requirements. This configuration directory is not under revision control.

### 6. Build an image

This process downloads over 10 gigabytes of source code and then
proceeds to do an awful lot of compilation so make sure you have
plenty of space, and expect several hours of build time
depending on your network connection and host speed.  Don't worry, it
is just the first build that takes a while.

A number of images can be built, however the smallest (and the quickest to build) is `core-image-minimal`, so:

``` shell
% bitbake core-image-minimal
```

If everything goes well, you should end up with everything you need to boot a Linux system on QEMU in the `tmp/deploy/images/qemuriscv64cheri` directory.  If you run into problems, the most likely
candidate is missing software packages on the host, hopefully the error message will give some indication of what is missing.

### 7. Running the image on QEMU

QEMU provides the [virt](https://www.qemu.org/docs/master/system/riscv/virt.html) Virtual Platform which is ideal for running simple images like this. So to run the images which has just been built:

``` shell
% ./tmp/work/x86_64-linux/qemu-system-native/7.0.0+git/build/qemu-system-riscv64cheristd \
  -machine virt \
  -cpu codasip-x730,cheri_pte=on,cheri_levels=2 \
  -m 512 \
  -nographic \
  -no-reboot \
  -bios tmp/deploy/images/qemuriscv64cheri/fw_dynamic.bin \
  -kernel tmp/deploy/images/qemuriscv64cheri/Image \
  -drive file=tmp/deploy/images/qemuriscv64cheri/core-image-minimal-qemuriscv64cheri.rootfs.wic,if=none,format=raw,id=virtdisk \
  -device virtio-blk-device,drive=virtdisk,bus=virtio-mmio-bus.0 \
  -append "root=/dev/vda1 earlycon console=ttyS0,115200"
```

## Other images to try

Many of the other recipes in the poky distribution have been checked for compatibility with CHERI, and updates applied where necessary. The minimal image uses (Busybox)[https://busybox.net/] for most programs, but if you want to use the full version of programs,  then:

``` shell
% bitbake core-image-full-cmdline
```

will build am image with a lot more programs.

Poky also has support for a simple graphical environment called Sato: a mobile environment and visual style that works well with mobile devices. The image supports X11 with a Sato theme and applications such as a terminal, editor, file manager, media player, and so forth.

To build a Sato image:

``` shell
% bitbake core-image-sato
```

Running an image like this on QEMU requires more devices to be added, so to simplify things Yocto provides wrapper which added the extra options. So to run the last image built:

``` shell
% runqemu serialstdio
```

## Staying Up to Date

To pick up the latest changes for all source repositories, run:

``` shell
% repo sync
```

Enter the Yocto Project build environment:

``` shell
% source poky/oe-init-build-env build
```

If you forget to setup these environment variables prior to running
`bitbake`, the OS will complain that it can't find bitbake on the path.  Don't try to
install bitbake using a package manager, just run the above command.

You can then rebuild as before:

``` shell
% bitbake core-image-minimal
```

## Starting Fresh

So something broke... what do you do now?

There are several degrees of *starting fresh*: individual packages can be
rebuilt or the whole system can be reconstructed.

1. clean a package: `bitbake <package-name> -c cleansstate`
2. re-download package: `bitbake <package-name> -c cleanall`
3. destroy everything but downloads: `rm -rf build/sstate-cache build/tmp` (or wherever your sstate and work directories are)
4. destroy it all (not recommended): `rm -rf build`

To understand better how bitbake processes recipes, look at the
[Yocto Project Reference Manual](http://www.yoctoproject.org/docs/current/poky-ref-manual/poky-ref-manual.html).

To make sense of the differences between these cleaning methods, it is useful to
understand that Yocto caches both the downloaded source files for all the
packages it tries to build (the `DL_DIR` configuration parameter) and the packages
once built (the `SSTATE_DIR` configuration parameter). Typically, deleting the
downloaded source is a bad idea - this just means re-fetching gigabytes of code
which wastes network bandwidth. Cleaning the sstate cache for a particular
package ensures that it actually gets rebuilt from source rather than simply
restored from the cache.
