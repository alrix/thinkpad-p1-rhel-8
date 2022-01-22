# RHEL 8 on a Thinkpad P1 (Gen1)

This repo contains instructions for installing RHEL8 on a Lenovo Thinkpad P1 (Gen1). These instructions may work on other generations of P1 and also the X1 Extreme. 

## Why RHEL 8?

When I first purchased the laptop in 2019, I tried running CentOS 8 and RHEL 8 on the machine, with little success. The biggest issue was suspend and resume but also lock ups and no prime support for the GPU. I ended up switching to Pop! OS for a while, then back to Windows 10 + WSL2, then OpenSUSE Tumbleweed, another stint with Pop! OS and finally Fedora 35 Silverblue. 

Recently, I used RHEL 8.5 on a desktop and was impressed with its polish and extra app support through flatpak. I thought it was time to try again on the Thinkpad and I have to say the results this time are amazing - a really stable and secure system with a great interface that I know will be supported for the next few years. I can comfortably say I will be sticking with it for the forseeable.

This README covers the steps I had to take to make RHEL 8 fully functional on the laptop and serve as a reminder to me for if I rebuild or when the inevitable upgrade to RHEL 9 comes along.

For most things, I use RedHat's own documentation or if I need to dig further, the ArchLinux wiki. I find these are better sources of information than following random blogs or troubleshooting wiki's.

## Steps

### 1 RHEL Installation
Install RHEL 8.5 from dvd - I chose Workstation installation. Use the defaults (with encryption) for hard-disk setup.

### 2 NVIDIA Drivers and setup

I followed the instuctions here:

https://developer.nvidia.com/blog/streamlining-nvidia-driver-deployment-on-rhel-8-with-modularity-streams/

Basically:

```
$ subscription-manager repos --enable=codeready-builder-for-rhel-8-x86_64-rpms
$ sudo dnf config-manager --add-repo=https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
$ sudo dnf module install nvidia-driver:latest
```

I didn't use the DKMS drivers. The downside of using the precompiled drivers is if RedHat release an updated kernel and Nvidia doesn't follow suit immediately, you will get errors with package mismatch when doing a dnf update. It usually only takes a few hours though for the issue to go away.

Reboot the system - it will come up with the nvidia dedicated card for Xorg. To use PRIME, update /etc/X11/xorg.conf.d/10-nvidia.conf commenting out the following line:
```
# Option "PrimaryGPU" "yes"
```

Reboot the system. You can use the nvidia-smi command to determine whether Prime is working.

### 3 Update PulseAudio and Bluetooth Configuration (Optional)

I update pulseaudio and bluetooth configuration. This is up to you.

For auto-switching when using bluetooth headset, update /etc/pulse/default.pa:

```
load-module module-bluetooth-policy auto_switch=2
```

For bluetooth timeouts, I set the following in /etc/bluetooth/main.conf:

```
DiscoverableTimeout = 30
PairableTimeout = 30
```

### 4 Install EPEL

EPEL provides a set of useful additional tools. See https://docs.fedoraproject.org/en-US/epel/. 

Install using the following

```
subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
```

### 5 Install and setup TLP

TLP helps optimise battery life on the laptop - both runtime and the life-time of the battery itself. See https://linrunner.de/tlp/ for more information.

To install:

```
dnf install tlp tlp-rdw
```

I update the recharge thresholds to extend battery life by editing /etc/default/tlp:

```
START_CHARGE_THRESH_BAT0=50
STOP_CHARGE_THRESH_BAT0=80
```

TLP depends on a kernel module acpi_call to handle threshold adjustment and also batter recalibration. Its not available in RPM format that will work without installing lots of dependencies from different repos. This is what I do:

Install DKMS:

```
# dnf install dkms
```

You can clone the following nix community repository where the acpi_call module is being maintained - https://github.com/nix-community/acpi_call or download the following:

```
# wget https://github.com/nix-community/acpi_call/archive/refs/tags/v1.2.2.zip
```

Extract out to somewhere - e.g. opt

```
# unzip v1.2.2.zip
```

Go into the directory and use DKMS to build and install the module. On new kernel installs, it will automatically get updated:

```
# make dkms-add
# make dkms-install
```

You can check to see if the module is loaded using lsmod.

```
# lsmod | grep acpi_call
acpi_call              16384  0
```

Finally, run tlp-stat and check through the output. You should see the following line to confirm the acpi_call module is all working:

```
tpacpi-bat = active (recalibrate)
```

### 6 Enable FlatPak

Flatpak is useful if you want to install software such as Spotify. To enable, follow RedHat's instructions here:

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_the_desktop_environment_in_rhel_8/assembly_installing-applications-using-flatpak_using-the-desktop-environment-in-rhel-8

Once setup, I then enable flatpak.org's repository to gain access, following their instructions at https://flatpak.org/setup/Red%20Hat%20Enterprise%20Linux/

### 7 Scanning

Unfortunately, SANE doesn't pick up my canon pixma scanner automatically. I follow the instructions on the excellent ArchLinux wiki here to update the files under /etc/sane.d - https://wiki.archlinux.org/title/SANE

### 8 Final Tuning

I set the power-profile to powersave but there are other options:

```
# tuned-adm profile powersave
```

I enable mdns for zeroconf setup of local resources such as printers:
```
# dnf install nss-mdns
# firewall-cmd --add-service=mdns 
# firewall-dms --add-service=mdns --permanent
```

I add the icc profile in this repo to improve the screen colours. 


