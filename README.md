on-imagebuilder [![Build Status](https://travis-ci.org/RackHD/on-imagebuilder.svg?branch=master)](https://travis-ci.org/RackHD/on-imagebuilder)
===============

Copyright 2015, EMC, Inc.

This repository contains a set of scripts that execute ansible playbooks and roles used for building
debian-based linux microkernel images and overlay filesystems, primarily for use
with the [on-taskgraph workflow engine](https://github.com/rackhd/on-taskgraph).
There are three ansible playbooks included, which build mountable
[squashfs](https://en.wikipedia.org/wiki/SquashFS) images,
[overlay](https://en.wikipedia.org/wiki/OverlayFS) filesystems, and initrd images.

<br>

Requirements
---------------

- Any Debian/Ubuntu based system (support for other distributions coming soon, theoretically though, just install debootstrap and it should work)
- Ansible (> 2.0) (**NOTE**: Version 2.1.0.0 is not workable) is installed. To install newest Ansible version ([Ansible Installation Documents](http://docs.ansible.com/ansible/intro_installation.html)):

**WARNING**: We have seen on some system that an existing installation of Ansible can have adverse effects, this can be resolved by insuring they are removed prior to 
the next steps, below is an example of removing an existing dpkg installation but this could vary depending on the package manager.  
    ```
    sudo dpkg -r ansible
    ```

- Install ansible:

    ```
    sudo apt-get -y update
    sudo apt-get -y install python-software-properties python-pip python-dev libffi-dev
    sudo pip install ansible==2.0.2.0
    ```

- Internet access OR network access to an apt cache/proxy server from the build machine

<br>

Overview
---------------

### Terms

**minbase image**

This is a lightweight debian filesystem packaged up as a mountable squashfs image.
Essentially, it's just a [debootstrap](https://wiki.debian.org/Debootstrap) minbase
filesystem with some added configurations and packages. It is ~50mb squashed
and occupies ~120mb of space when mounted. The base image[s] are used as a shared
image that different overlays can be built and mounted with, and take 3-5 minutes to build.

**base image**

This is a standard debian file filesystem packaged up as a mountable squashfs image.
Essentially, it's just a [debootstrap](https://wiki.debian.org/Debootstrap)
filesystem with some added configurations and packages. It is ~64mb squashed
and occupies ~220mb of space when mounted. This larger base image can be used for
applications that require more than minbase provides, and should be fully compatible
with any overlays built against minbase. It takes 3-5 minutes to build. 

**overlay filesystem**

The overlay filesystem is a gzipped cpio archive of copy-on-write changes made to
a mounted base image. Usually an overlay filesystem just contains a few packages
and/or shell scripts, and is often under 10mb in size and takes under a minute to build.

**provisioner**

An ansible role used to specify changes that should be made to the filesystem
of an initrd, base image or overlay (e.g. extra packages, scripts, files). An
example provisioner is the oem role in this repository, which is an example of
the ansible tasks and config values needed to make a microkernel with vendor
specific tooling for use with workflows.

### Bootstrap process

The images produced by these playbooks are intended to be netbooted and run in RAM.
The typical flow for how these images are used/booted is this:

- Netboot the kernel and initrd via PXE/iPXE
- The custom-built initrd runs a startup script (roles/initrd/provision_initrd/files/local)
  that requests a base squashfs image and an overlay filesystem from the boot server. It then
  mounts both images together (union mount) into a tmpfs and boots into that as the root.

The basefs and initrd images aren't intended to be changed very often. It's more likely
that one will add new provisioner roles to build custom overlays that can be mounted
with the base image built by the existing ansible roles in this repository.

<br>

Getting started
---------------

### Building images

To build images, define an imagebuilding script (see example.sh for an example) or
use the default one. For example, to build the default images:

```
$ cd on-imagebuilder/
$ sudo ./build_all.sh
```

**All builds runs must be done on the host machine that is building the images.** This is because we use
the ansible_chroot connection type, which is not supported over ssh connections.

The build artifacts will be located in these directories below (which are defined in the `./hosts` file):
- **/tmp/on-imagebuilder/builds/**    :  it includes the artifacts:
  * base.*.squashfs.img
  * discovery.overlay.cpio.gz
  * initrd.img-*
  * vmlinuz-*

- **/tmp/on-imagebuilder/ipxe/**      :  it includes the artifacts:
  * monorail-efi32-snponly.efi
  * monorail-efi64-snponly.efi  
  * monorail.ipxe  
  * monorail-undionly.kpxe

- **syslinux**/undionly.kkpxe 


### Adding provisioner roles and configuration files

The provisioner role is what specifies how the filesystem of an initrd, base
image or overlay should be customized. To add a new provisioner, do the following.
If you have experience with Ansible, some of these steps will be familiar:

- Make a new directory in `roles/<initrd|basefs|overlay>/tasks`, depending on the image type
- Create and edit a main.yml file in the above directory to do the tasks you
  want (see [ansible modules](http://docs.ansible.com/ansible/modules_intro.html)
  if new to Ansible)
- Add a new config yaml file into the vars directory. This will be included in the
  Ansible run as a set of top-level variables (via the -e argument) to be included/used
  by tasks in the role.
- Configure a new script (see example.sh) to run the appropriate wrapper
  playbook with the config file and the provisioner role specified as vars, for example:

    ```
    sudo ansible-playbook \
    -i hosts common/overlay_wrapper.yml \
    -e "config_file=vars/overlay.yml \
    provisioner=roles/overlay/provision_discovery_overlay"
    ```
  The wrapper playbooks handle all the setup and cleanup required to run a
  provisioner, such as filesystem mounting and creation, and build file creation.
- **NOTE**: 
  * The customized overlay depends on initrd and basefs build result, that said, before running the overlay playbook, below playbook should run firstly:

    ```
    sudo ansible-playbook -i hosts common/initrd_wrapper.yml \
    -e "config_file=vars/initrd.yml provisioner=roles/initrd/provision_initrd"

    sudo ansible-playbook -i hosts common/basefs_wrapper.yml \
    -e "config_file=vars/basefs.yml provisioner=roles/basefs/provision_rootfs"
    ```
    So it should run `build_all.sh` which include inirtd and basefs playbook before `build_oem.sh`.
  * OEM roles provision_raid_overlay and provision_secure_erase_overlay require storcli_1.17.08_all.deb being copied into common/files.
    User can download it from http://docs.avagotech.com/docs/1.17.08_StorCLI.zip.
    Tool name ***storcli_1.17.08_all.deb*** is hard coded.
    If a package with different name is to be used, user should edit configure file in var/oem.
    For example if user wants to use storcli.15.03_all.deb instead of storcli_1.17.08_all.deb for secure erase, user should edit vars/oem/secure_erase_overlay.yml and replace below line:

    ```
    secure_erase_overlay_storcli_package: "storcli_1.17.08_all.deb"
    ```

    with:

    ```
    secure_erase_overlay_storcli_package: "storcli_1.15.03_all.deb"
    ```
    There is no requirement on tool name format. It is that secure_erase_overlay_storcli_package value should be exactly the same with the package file name stored in common/files.

  * OEM roles provision_dell_raid_overlay and provision_secure_erase_overlay require perccli_1.11.03-1_all.deb being copied into common/files.
    There is no .deb version perccli tool. User can download .rpm perccli from:
    https://downloads.dell.com/FOLDER02444760M/1/perccli-1.11.03-1_Linux_A00.tar.gz
    unzip the package and then use **alien** to get a .deb version perccli tool as below:

    ```
    sudo apt-get install alien
    sudo alien -k perccli-1.11.03-1.noarch.rpm
    ```
    Again, user can use a different perccli package via aligning its name in configure files of vars/oem and common/files.

### Changing the global configuration

All playbooks and roles depend on the variables defined in hosts and
group_vars/on_imagebuild. These variables specify where the build roots are
located, and which apt server/package repositories are used.

**Changing the build root**

Update the paths in hosts to the desired build root. The build root paths are
duplicated between the host sections (e.g. [overlay_build_chroot]) and the
[on_imagebuild:vars] section, so they must be changed in both places.

**Changing the repository URLs**

It is highly recommended that an apt-cacher-ng server be used rather than the
upstream archive.ubuntu.com server specified by default. Depending on the network
connection, this can cut build times for the basefs and initrd images in half. To
set this up, run:

```
apt-get install apt-cacher-ng
```

Then edit the apt_server variable in group_vars/on_imagebuild to equal the address
of your apt cache server, e.g.

```
apt_server: 192.168.100.5:3142
```

The first build will still be slow as no packages are cached, but subsequent builds will be much faster.


FYI
---------------

While Ansible is leveraged because of its great modular design and variable
configuration support, these playbooks can only be run LOCALLY, not against remote
hosts as is usual with Ansible. This is because the chroot ansible_connection
type is used for most builders and provisioners, which is not supported over ssh
and other remote ansible_connection types.


But, why not containers?
---------------

The goal here is to optimize for size on disk and modularity. By creating many
different overlays that all share a base image, we avoid data
duplication on the boot server (50MB base image + 10 * 5MB overlay archives
vs. 10 * 55MB container images).
Additionally, it gives us flexibility to update the base image
and any system dependencies/scripts/etc. on it without having to rebuild
any overlays. For example, we use a custom rc.local script in the base image
that is used to receive commands from
[workflows](https://github.com/rackhd/on-tasks) on startup. Making
changes to this script should only have to be done in one place.

That said, please send us a note if you think this is incorrect! If we can leverage
existing container technology instead of a homebrewed imagebuilding process and
satisfy our design constrains, then we're all for it.


Licensing
---------------

Licensed under the Apache License, Version 2.0 (the “License”); you may not use this file except in compliance with the License. You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an “AS IS” BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

RackHD is a Trademark of Dell EMC
