## `V` System Configuration
The purpose of this stage is to configure the system to be boot-able, and installing the Linux Kernel with desired Bootloader (and/or EFI Stub).

### `1` - Creating the `/etc/resolv.conf` File
> The system will need some means of obtaining Domain Name Service (DNS) name resolution to resolve Internet domain names to IP addresses, and vice versa. This is best achieved by placing the IP address of the DNS server, available from the ISP or network administrator, into "/etc/resolv.conf".

> Use Cloudflare DNS as primary, Google as secondary.
```bash
cat > /etc/resolv.conf << "EOF"
# Begin /etc/resolv.conf

nameserver 1.1.1.1
nameserver 8.8.8.8

# End /etc/resolv.conf
EOF
```

### `2` - Customizing the `/etc/hosts` File
> When booting, OpenRC will set the system hostname from "/etc/conf.d/hostname".
```bash
# Define preferred default hostname. Below will use the host's name.
HOSTNAME="${HOSTNAME}"

# "/etc/hosts" file.
cat > /etc/hosts << EOF
# Begin /etc/hosts

127.0.0.1 ${HOSTNAME} localhost
127.0.1.1 ${HOSTNAME}
::1       ${HOSTNAME} localhost ip6-localhost ip6-loopback

# End /etc/hosts
EOF
```

### `3` - Customizing the `/etc/nanorc` File
> Setup the Nano text-editor default configuration.
```bash
cat > /etc/nanorc << "EOF"
# Begin /etc/nanorc

# Functions
# ---
set autoindent
set tabsize 4
set tabstospaces
set linenumbers
set softwrap

# Color Schemes
# ---
set titlecolor black,red
set statuscolor magenta
set keycolor blue
set functioncolor white
set numbercolor white

# End /etc/nanorc
EOF
```

### `4` - Configure Locales and Shell Startup Files
```bash
# Determine locale supported by `musl-locales`. Should display "UTF-8".
LC_ALL=en_US.utf8 locale charmap
```
```bash
# Once the proper locale settings have been determined, create the "/etc/profile" file.
cat > /etc/profile << "EOF"
# Begin /etc/profile

export LANG="en_US.utf8"
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
export EDITOR="${EDITOR:-nano}"
export PAGER="${PAGER:-more}"

# 077 would be more secure, but 022 is generally quite realistic
umask 022

# process *.sh files in /etc/profile.d
for sh in /etc/profile.d/*.sh ; do
    [ -r "$sh" ] && . "$sh"
done; unset sh

if [ -n "${BASH_VERSION-}" ]; then
    # Newer bash ebuilds include /etc/bash/bashrc which will setup PS1
    # including color.  We leave out color here because not all
    # terminals support it.
    if [ -f /etc/bash/bashrc ]; then
        # Bash login shells run only /etc/profile
        # Bash non-login shells run only /etc/bash/bashrc
        # Since we want to run /etc/bash/bashrc regardless, we source it 
        # from here.  It is unfortunate that there is no way to do 
        # this *after* the user's .bash_profile runs (without putting 
        # it in the user's dot-files), but it shouldn't make any 
        # difference.
        . /etc/bash/bashrc
    else
        PS1='^ \u@\h [ \w ]\n\$ '
    fi
else
    # Setup a bland default prompt.  Since this prompt should be useable
    # on color and non-color terminals, as well as shells that don't
    # understand sequences such as \h, don't put anything special in it.
    PS1="^ ${USER:-$(whoami 2>/dev/null)}@$(uname -n 2>/dev/null)\n\$ "
fi

# End /etc/profile
EOF
```
```bash
# Install Heiwa/Linux bashrc configuration.
install -vDm644 /sources/extra/bash/files/bashrc /etc/bash/
```
```bash
# Reload the environment.
. /etc/profile
```

### `5` - Creating the `/etc/inputrc` File (GNU Readline)
> The inputrc file is the configuration file for the readline library, which provides editing capabilities while the user is entering a line from the terminal. It works by translating keyboard inputs into specific actions. Readline is used by bash and most other shells as well as many other applications.

> Most people do not need user-specific functionality so the command below creates a global "/etc/inputrc" used by everyone who logs in. If you later decide you need to override the defaults on a per user basis, you can create a ".inputrc" file in the user's home directory with the modified mappings.
```bash
cat > /etc/inputrc << "EOF"
# Begin /etc/inputrc

# Allow the command prompt to wrap to the next line.
set horizontal-scroll-mode Off

# Enable 8bit input.
set meta-flag On
set input-meta On

# Turns off 8th bit stripping.
set convert-meta Off

# Keep the 8th bit for display.
set output-meta On

# Do not bell on tab-completion.
set bell-style none

# All of the following map the escape sequence of the value contained in the 1st argument to the readline specific functions.
"\eOd": backward-word
"\eOc": forward-word

# Linux console and RH/Debian xterm.
# Allow the use of the Home/End keys.
"\e[1~": beginning-of-line
"\e[4~": end-of-line
# Map "page up" and "page down" to search history based on current cmdline.
"\e[5~": history-search-backward
"\e[6~": history-search-forward
# Allow the use of the Delete/Insert keys.
"\e[3~": delete-char
"\e[2~": quoted-insert

# GNOME / others (escape + arrow key).
"\e[5C": forward-word
"\e[5D": backward-word

# konsole / xterm / rxvt (escape + arrow key).
"\e\e[C": forward-word
"\e\e[D": backward-word

# GNOME / konsole / others (control + arrow key).
"\e[1;5C": forward-word
"\e[1;5D": backward-word

# aterm / eterm (control + arrow key).
"\eOc": forward-word
"\eOd": backward-word

# konsole (alt + arrow key).
"\e[1;3C": forward-word
"\e[1;3D": backward-word

# Chromebooks, remap (alt + backspace) so provide alternative (alt + k).
"\ek": backward-kill-word

$if term=rxvt
"\e[8~": end-of-line
$endif

# For non RH/Debian xterm, can't hurt for RH/Debian xterm.
"\eOH": beginning-of-line
"\eOF": end-of-line

# Freebsd console.
"\e[H": beginning-of-line
"\e[F": end-of-line

# End /etc/inputrc
EOF
```

### `6` - Creating the `/etc/shells` File
> The shells file contains a list of login shells on the system. Applications use this file to determine whether a shell is valid. For each shell a single line should be present, consisting of the shell's path relative to the root of the directory structure (/).

> For example, this file is consulted by `chsh` to determine whether an unprivileged user may change the login shell for her own account. If the command name is not listed, the user will be denied the ability to change shells.

> It is a requirement for applications such as GDM which does not populate the face browser if it can't find "/etc/shells", or FTP daemons which traditionally disallow access to users with shells not included in this file.
```bash
cat > /etc/shells << "EOF"
# Begin /etc/shells

/bin/sh
/bin/bash

# End /etc/shells
EOF
```

### `7` - Linux
> #### `5.13.x` (CacULE) or newer
> The Linux package contains the Linux kernel.

> **Required!** If not, will you boot with the OpenBSD kernel?
```bash
# Make sure to disable Clang/LLVM environment variables to build kernel correctly.
mv -fv ~/.bash_profile{,_disabled}

# Reload the environment.
exec env -i HOME=/root TERM=xterm bash --login
```
```bash
# Apply patch to fix "swab.h" under musl libc while building Linux kernel.
patch -Np1 -i \
/sources/extra/linux-headers/patches/include-uapi-linux-swab-Fix-potentially-missing-__always_inline.patch

# Make sure there are no stale files embedded in the package.
time { make LLVM=1 LLVM_IAS=1 mrproper; }

# Copy Heiwa/Linux kernel configuration.
cp -rfv /sources/extra/linux-xanmod-cacule/files/{.config,localversion,drivers} .

# Configure source.
time { make LLVM=1 LLVM_IAS=1 menuconfig; }

# Build.
time { nice -n -1 make LLVM=1 LLVM_IAS=1 -j$(nproc); }

# Install modules.
time { make LLVM=1 LLVM_IAS=1 modules_install; }

# If the host system has a separate "/boot" partition, the files copied below should go there.
# The easiest way to do that is to bind "/boot" on the host (outside chroot) to "${HEIWA}/boot" before proceeding.
# As the root user in the host system.
# mount -R /boot /media/Heiwa/boot

# Export target kernel version to use on installation steps.
export KVER="5.13.1-heiwa-x86_64"

# Install.
# The path to the kernel image may vary depending on the platform being used.
# The filename below can be changed to suit your taste, but the stem of the filename should be vmlinuz to be compatible with the automatic setup of the boot process described in the next section.
# The following command assumes an x86 architecture.
cp -iv arch/$(uname -m)/boot/bzImage /boot/vmlinuz-${KVER}

# System.map is a symbol file for the kernel.
# It maps the function entry points of every function in the kernel API, as well as the addresses of the kernel data structures for the running kernel.
# It is used as a resource when investigating kernel problems.
# Issue the following command to install the map file.
cp -iv System.map /boot/System.map-${KVER}

# The kernel configuration file .config produced by the make menuconfig step above contains all the configuration selections for the kernel that was just compiled.
# It is a good idea to keep this file for future reference.
cp -iv .config /boot/config-${KVER}

# Unset exported KVER variable.
unset KVER

# Create the modules-load.d and modprobe directories in the "/etc".
mkdir -pv /etc/mod{ules-load,probe}.d
```

### `8` - Creating the `/etc/fstab` File
> The "/etc/fstab" file is used by some programs to determine where file systems are to be mounted by default, in which order, and which must be checked (for integrity errors) prior to mounting.
```sh
# If using GPT, preferred to use "PARTUUID=" instead of "/dev/" and/or "UUID=" that requires initramfs.
cat > /etc/fstab << "EOF"
# Begin /etc/fstab

# file system  mount-point  type     options             dump  fsck
#                                                              order

/dev/<xxx>     /boot        vfat     noatime             0     0
/dev/<yyy>     /            ext4     noatime,discard     0     0
/dev/<zzz>     none         swap     sw                  0     0

# End /etc/fstab
EOF
```

### `9` - Tweaking sysctl
```bash
# Apply some sysctl tweaks.
cat > /etc/sysctl.conf << "EOF"
# Reduce Swappiness
vm.swappiness = 10
vm.vfs_cache_pressure = 50

# Network Optimization
net.core.netdev_max_backlog = 100000
net.core.netdev_budget = 50000
net.core.netdev_budget_usecs = 5000
net.core.somaxconn = 1024
net.ipv4.tcp_fastopen = 3
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.ping_group_range = 0 2147483647
EOF
```

### `10` - Setup bootloader or direct EFI
> #### ^ EFI Stub
```bash
# Use "blkid" to determine PARTUUID.
blkid

# Export partuuid variable. Below is example.
export PU="8e110801-8f49-2b41-a458-631e2ce4df34"

# Without initramfs.
efibootmgr --create --part 1 --disk /dev/sda --label "HEIWA_heiwa-x86_64" --loader "\vmlinuz-5.12.10-heiwa-x86_64" \
-u "root=PARTUUID=${PU} rootfstype=ext4 rootflags=discard rw,noatime loglevel=4"

# Unset exported PARTUUID variable.
unset PU
```

### `11` - Configure OpenRC Services
> Configure openrc services, so can normally boot up.
```sh
# Remove login "No mail" and last login prompt.
touch ~/.hushlogin

# Copy default agetty configuration files for tty1-tty6.
for X in {1..6};do
    cp -v /etc/conf.d/agetty{,.tty${X}}
done

# Configure to not clean up init output after agetty.tty1 spawned.
sed -i 's|#agetty_options=".*"|agetty_options="--noclear"|' \
/etc/conf.d/agetty.tty1

# Fix sysctl init script failed to start since using Toybox instead of Procps-ng.
sed -i 's|--system|-p|' /etc/init.d/sysctl

# Configure for better openrc settings.

# Parallel execution.
sed -i 's|#rc_parallel=.*|rc_parallel="YES"|' \
/etc/rc.conf

# Since using Linux.
sed -i 's|#rc_shell=.*|rc_shell=/sbin/sulogin|' \
/etc/rc.conf

# Unicode support.
sed -i 's|#unicode=.*|unicode="YES"|' \
/etc/rc.conf

# Some tweaks.
sed -i 's|#rc_send_sighup=.*|rc_send_sighup="YES"|' \
/etc/rc.conf

sed -i 's|#rc_timeout_stopsec=.*|rc_timeout_stopsec="9"|' \
/etc/rc.conf

sed -i 's|#rc_send_sigkill=.*|rc_send_sigkill="YES"|' \
/etc/rc.conf

# Adds opentmpfiles to runlevel.
rc-update add opentmpfiles-dev sysinit
rc-update add opentmpfiles-setup boot

# Adds udev-gentoo-scripts to runlevel.
rc-update add udev sysinit
rc-update add udev-trigger sysinit

# Remove useless service from default runlevel.
rc-update delete netmount default

# Remove net on netmount dependencies.
sed -i 's|rc_need="net"|#rc_need="net"|' \
/etc/conf.d/netmount
```

### `12` - The End
> Release notes and logout.
```sh
install -vm644 -t /etc/ /sources/extra/branding-heiwa/etc/{lsb,os}-release

# Exits chroot environment.
logout
```
> #### * End of as root in a chroot env!
