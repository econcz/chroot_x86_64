# QEMU-based x86_64 chroot for Raspberry Pi 4B or later with (in-built) Stata® and other x86_64 software support

This script a) creates a [QEMU VM qcow2 file](https://qemu.readthedocs.io/en/latest/) from an automatically downloaded ISO image, b) rsyncs (copies) its filesystem into a chroot, c) upgrades it when necessary, d) configures the chroot to launch x86_64 software on machines with arm64 (aarch64) CPU architecture.

Tailored for [Raspberry Pi 4B (64-bit microcomputer) or later](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) and SW like [Stata®](https://www.stata.com/) (in-built support), [Conda (Anaconda, Miniconda) 3+](https://www.anaconda.com/), Adobe Acrobat DC (Legacy) etc.

**NB** [CrossOver 20+](https://www.codeweavers.com/crossover/) doesn’t yet seem to be working under chroot, please use Wine instead!

### Requirements:
- Debian/Ubuntu-based system (preferably Raspberry Pi OS)
- qemu (with x86_64 support) - has to be built (see below)!
- qemu-user-static
- samba
- libguestfs-tools
- udisks2
- zsh (I switched to Zsh some years ago, sorry)

```
# qemu and qemu-system (the build takes ca. 1-2 hours)
# courtesy of https://www.raspberrypi.org/forums/viewtopic.php?t=246886
####
sudo apt install build-essential ninja-build libepoxy-dev libdrm-dev libgbm-dev libx11-dev libvirglrenderer-dev libpulse-dev libsdl2-dev git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev libepoxy-dev libdrm-dev libgbm-dev libx11-dev libvirglrenderer-dev libpulse-dev libsdl2-dev git
git clone https://git.qemu.org/git/qemu.git
cd qemu
git submodule init
git submodule update --recursive
./configure --enable-sdl  --enable-opengl --enable-virglrenderer --enable-system --enable-modules --audio-drv-list=pa --enable-kvm
ninja -C build
sudo ninja install -C build

# qemu-user-static, samba, libguestfs-tools, udisks2, and zsh
####
sudo apt install qemu-user-static samba libguestfs-tools udisks2 zsh
```

**NB** You can backup the build for repeated use with `tar czfv ~/qemu.tar.gz ~/qemu` before deleting it with `rm -rf ~/qemu`.

### Configuration:
`user____________="pi"` Raspberry Pi (host) user ("pi")  
`user_chroot_____="pi"` Chroot user ("pi" is recommended)  
`display_________=0` Raspberry Pi display (0)  
`path_filesystem_="/tmp"` SD Card, USB drive etc.  
`path_software___="/tmp"` SD Card, USB drive etc.  
`path_backup_file="/home/pi/Documents/non_qcow2.tar.gz"` Backup tar.gz archive for `chroot_x86_64 --backup`  
`qemu_image_url__="https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-10.10.0-amd64-netinst.iso"` ISO image URL  
`qemu_image_size_="16G"` QEMU VM, image size, at least 16GB (maximum size)  
`qemu_smp________=2` QEMU VM, number of cores, 50% of host (recommended)  
`qemu_m__________="4G"` QEMU VM, RAM size, 50% of host (recommended)  
`qemu_ssh_port___=22000` QEMU VM, SSH port  
`path_local_sw___="/usr/local"` Path to Stata® and other SW  
`path_mountpoint_="/mnt"` Mountpoint location in host  
`path_chroot_____="/var/chroot_x86_64"` Chroot location in host  
`path_user_chroot="/home/pi"` Chroot user home folder (~)  
`vm_on_upgrade___=1` Run QEMU VM on `chroot_x86_64 --upgrade` (1 for yes, 0 for no)    
`rsync_ignore____="/home/pi/ ${path_local_sw___}/stata17/"` Chroot locations to ignore on `chroot_x86_64 --upgrade` and to backup with `chroot_x86_64 --backup` (no patterns allowed!)  

**NB** If `chroot_x86_64 --upgrade` is interrupted, unsynced files (defined in `$rsync_ignore____`) can be found in `~/.rsync_ignore/`!

### Usage:
`chroot_x86_64 --install` to install the chroot environment  
`chroot_x86_64 --setup` to setup the chroot environment  
`chroot_x86_64 --upgrade` to upgrade the chroot environment  
`chroot_x86_64 --backup` to backup unsynced folders/files  
`chroot_x86_64 --restore` to restore the chroot environment  
`chroot_x86_64 <cmd>` to run <cmd> under chroot

### Running Stata (change paths and version if necessary):
Requires `libncurses5` and `libgtk2.0-0` installed in QEMU VM and rsynced to chroot.  
If intended to be run over SSH, don’t forget to set DISPLAY `export DISPLAY=:0`.

Slower but with Java and Python (Azul Zulu JDK build shipped with Stata doesn’t seem to be working under qemu-user-static, probably because of dynamic library loading):  
`qemu-x86_64 /usr/local/stata17/stata` (Stata IC console)  
`qemu-x86_64 /usr/local/stata17/xstata` (Stata IC GUI)  
`qemu-x86_64 /usr/local/stata17/stata-se` (Stata SE console)  
`qemu-x86_64 /usr/local/stata17/xstata-se` (Stata SE GUI)  
```
# For Jupyter kernel (to launch from chroot):
echo "#!/bin/bash" | sudo tee /var/chroot_x86_64/usr/local/bin/stata-se
echo "chmod 0600 ~/.ssh/id_rsa; ssh -oAddressFamily=inet -oStrictHostKeyChecking=no -t -i ~/.ssh/id_rsa pi@localhost 'cd; qemu-x86_64 /usr/local/stata17/stata-se'" | sudo tee -a /var/chroot_x86_64/usr/local/bin/stata-se
```

Faster but without Java and Python:  
`/usr/local/stata17/stata` (Stata IC console)  
`/usr/local/stata17/xstata` (Stata IC GUI)  
`/usr/local/stata17/stata-se` (Stata SE console)  
`/usr/local/stata17/xstata-se` (Stata SE GUI)

**NB** Most Stata commands are C-based (especially compiled Mata .mo and .mlib files), only a small fraction are Java-based (for example, `sdmxuse`) or Python-based (for example, `pyconvertu`).  
**NB** The `ado` directory and hidden `.stata17` folder will be created in host `~/`.

### Running other software:
`sudo chroot_x86_64 <cmd>` (Any command, for example, `acroread`).  
**NB** The hidden folders will be created in chroot, for example, `/var/chroot_x86_64/home/pi/`.

Happy usage!