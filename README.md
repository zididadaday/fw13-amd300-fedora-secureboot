# Framework 13 (AMD Ryzen AI 300 Series) Fedora

## Content

- Background
- Secure Boot on Fedora
- Enabling automatic decryption of your encrypted luks drive over TPM
- Hibernation prerequisites
- Compiling kernel to allow for hibernation with secureboot
- Enabling hibernation/testing hibernation
- Other notes about installing Fedora
- Sources

## Background

I have made below tests on a Framework Laptop 13 with AMD Ryzen™ AI 300 (340) from May 2025.
Fedora installation was 42, with Secure Boot enabled. 1TB NVME drive. 64GB DDR5 5600 MHZ CL40. I have secureboot enabled, a luks encrypted drive, and hibernation working (including suspend-then-hibernation)

`https://fedoraproject.org/workstation/download/`

## Tested with Fedora kernel versions

- Fedora 6.14.6 fc42 - working
- Fedora 6.14.8 fc42 - working*
- Fedora 6.14.9 fc42 - working*
- Fedora 6.14.11 fc42 - working
- Fedora 6.15.2 fc42 - testing

*see additional kernel boot parameters required 

## Secure Boot

This repository should include the nescessary steps for you to run Fedora with Secure Boot and hibernation enabled, and allow you to unlock your encrypted drives with TPM 2.0, so you don't have to enter a passphrase every time you boot your computer.

If you don't have secureboot enabled, you should start by enabling that, otherwise this guide will not help you.

You can check to see if secureboot is enabled on Fedora with the command: `mokutil --sb-state`
If it is enabled it will return `SecureBoot enabled`.


## Enabling automatic unlock of encrypted disk with TPM 2.0

With the commands below we are using the PCR settings 1+3+5+7+11+12+14.

| PCR # | PCR name | Description |
|-------|----------|-------------|
|1| platform-config | Core system firmware data/host platform configuration; typically contains serial and model numbers, changes on basic hardware/CPU/RAM replacements|
|3| external-config | Extended or pluggable firmware data; includes information about pluggable hardware|
|5| boot-loader-config | GPT/Partition table; changes when the partitions are added, modified, or removed|
|7| secure-boot-policy | Secure Boot state; changes when UEFI SecureBoot mode is enabled/disabled, or firmware certificates (PK, KEK, db, dbx, ...) changes.|
|11| kernel-boot | measures the ELF kernel image, embedded initrd and other payload of the PE image it is placed in into this PCR.|
|12| kernel-config | measures the kernel command line into this PCR.|
|14| shim-policy | The shim project measures its "MOK" certificates and hashes into this PCR.|

I tested different PCR settings on my laptop, and found that I had to remove 8 and 9 because changes were constantly detected so the automatic unlock of my encrypted disk did not work. Some guides use other combinations of PCR, I suggest you test / try to understand what you want to enable.

My main goal is to have enough security without having to enter the encryption passphrase every time I start up my laptop. 2025-06-15: just got a secureboot update, PCR 7, 8, 9, 10 changed. 

```
# Add decryption key to tpm. Note that /dev/nvme0n1p3 is the name of my disk.
systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=1+3+5+7+11+12+14 /dev/nvme0n1p3

# verify TPM added (see tpm2 entry)
sudo systemd-cryptenroll /dev/nvme0n1p3

# Add tpm2 configuration option to /etc/crypttab
# discard, luks, tpm2-pcrs because it does no harm
# The entry should already exist, you just have to modify the config on the line end
luks-$UUID UUID=disk-$UUID none tpm2-device=auto,luks,discard,tpm2-pcrs=1+3+5+7+11+12+14

# Update initramfs (to get necessary tpm2 libraries and parameters for decryption into initramfs)
dracut -fv --regenerate-all

# Reboot to test that you are no longer prompted to enter your passphrase for disk decryption.
```

If you change something on your system and your system keeps prompting for a passphrase on boot, you can use the below command to remove the existing entry and add it once more.

```
# Wipe old keys and enroll new key. You have to execute this command again after a kernel upgrade.
sudo systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+3+5+7+11+12+14" /dev/nvme0n1p3
```

`https://community.frame.work/t/guide-setup-tpm2-autodecrypt/39005`

`https://man.archlinux.org/man/systemd-cryptenroll.1`

`https://gist.github.com/jdoss/777e8b52c8d88eb87467935769c98a95`

### Changes that stop TPM 2.0 from unlocking encrypted disk

To check what changes cause the automatic unlock from working.
`sudo tpm2 pcrread`

You can log a value before a reboot / update and compare the value after rebooting.

```
sudo tpm2 pcrread > ~/pcr_`date '+%F_%H:%M:%S'.txt`
```

You will see some output like this:

```
  sha1:
  sha256:
    0 : 0x6D299DAFFEED8FB698510B2C39186E483629C3E4B85EBBD0A03BE88B955937CA
    1 : 0xCFDF7448D6A43E7A93289BDC2D7FE99DB8DDD2B8FA6D1D8A87D40AD0BEB561CA
    2 : 0x3D458CFE55CC03EA1F443F1562BEEC8DF51C75E14A9FCF9A7234A13F198E7969
    3 : 0x3D458CFE55CC03EA1F443F1562BEEC8DF51C75E14A9FCF9A7234A13F198E7969
    4 : 0xDD040DD582246B6D02B301898B8898749E8C0117F47B89E8B974A745B5457B31
    5 : 0x5D7A89F7600D10396CACCC983401F7D5140445F37D609CD8A8C3C546EB10F20C
    6 : 0x3D458CFE55CC03EA1F443F1562BEEC8DF51C75E14A9FCF9A7234A13F198E7969
    7 : 0x0CA7BC73C39EE60B19E81B97660814580EEB9F5E21F014624DCC2DA1E30F58C5
    8 : 0x115AD1916B0808ABFD23BAE85222F848D9623A4FA744E350EB18A3C1B892AEF8
    9 : 0xFEF56CFA71A5BB5B54E0A7F5F86BD3C0C992EE31FF21ABDB9D53BA7E60C16D3E
    10: 0x8DC33179261D642EEF979E3C52809F966E0D127E9D138F681095996C8EA1D07C
    11: 0x0000000000000000000000000000000000000000000000000000000000000000
    12: 0x0000000000000000000000000000000000000000000000000000000000000000
    13: 0x0000000000000000000000000000000000000000000000000000000000000000
    14: 0xF591BCC0FBDC43DAFB30FC9373677556EF21D6C2D43997839AC540DAF7EC6173
    15: 0x0000000000000000000000000000000000000000000000000000000000000000
    16: 0x0000000000000000000000000000000000000000000000000000000000000000
    17: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    18: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    19: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    20: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    21: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    22: 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    23: 0x0000000000000000000000000000000000000000000000000000000000000000
  sha384:
```

Changes that I believe can change the PCR state.


1. `new kernel`
2. `hardware changes`
3. `secure boot updates`

## Hibernation prerequisites

Read on this link. It explains well what we have to do to enable hibernation on our systems.

`https://gist.github.com/eloylp/b0d64d3c947dbfb23d13864e0c051c67?permalink_comment_id=3936683#gistcomment-3936683`

If I have to summarize how I understand it: the default swap device on Fedora is a zram swop device. This swop partition is stored in the ram. When we hibernate, the laptop is powered off, so we need to store our swop partition on our nvme drive. Why would we want to hibernate and not just use suspend/sleep? Sometimes sleep uses more power than we want, hibernate can save power and allow us to resume working where we left off.

```
# first create the swapfile
SWAPSIZE=$(free | awk '/Mem/ {x=$2/1024/1024; printf "%.0fG", (x<2 ? 2*x : x<8 ? 1.5*x : x) }')
sudo btrfs subvolume create /var/swap
sudo chattr +C /var/swap
sudo restorecon /var/swap
sudo mkswap --file -L SWAPFILE --size $SWAPSIZE /var/swap/swapfile
sudo bash -c 'echo /var/swap/swapfile none swap defaults 0 0 >>/etc/fstab'
sudo swapon -av
sudo bash -c 'echo add_dracutmodules+=\" resume \" > /etc/dracut.conf.d/resume.conf'
sudo dracut -f
```

Then setup the hibernation and suspend-then-hibernation scripts:

```
/usr/lib/systemd/system-sleep/suspend-then-hibernate-post-suspend.sh
/etc/systemd/system/suspend-then-hibernate-resume.service
/etc/systemd/system/hibernate-resume.service
/etc/systemd/system/hibernate-preparation.service
```
Note the path to the `swapfile` in all the scripts, `/var/swap/swapfile`, change it if it is located elsewhere for you.

Create file `/usr/lib/systemd/system-sleep/suspend-then-hibernate-post-suspend.sh` with content:
```
#!/bin/bash

if [ "$1" = "post" ] && [ "$2" = "suspend-then-hibernate" ] && [ "$SYSTEMD_SLEEP_ACTION" = "suspend" ]
then
    echo "suspend-then-hibernate (post suspend): enabling swapfile and disabling zram"
    /usr/sbin/swapon /var/swap/swapfile && /usr/sbin/swapoff /dev/zram0
fi
```
Make sure it is executable `sudo chmod +x /usr/lib/systemd/system-sleep/suspend-then-hibernate-post-suspend.sh`

Create service file `/etc/systemd/system/suspend-then-hibernate-resume.service`
with content
```
[Unit]
Description=Disable swapfile after resuming from hibernation during suspend-then-hibernate
After=suspend-then-hibernate.target

[Service]
User=root
Type=oneshot
ExecStart=/usr/sbin/swapoff /var/swap/swapfile

[Install]
WantedBy=suspend-then-hibernate.target
```

Then enable it with `systemctl enable suspend-then-hibernate-resume.service`


Create the file `/etc/systemd/system/hibernate-resume.service`

```
[Unit]
Description=Disable swap after resuming from hibernation
After=hibernate.target

[Service]
User=root
Type=oneshot
ExecStart=/usr/sbin/swapoff /var/swap/swapfile

[Install]
WantedBy=hibernate.target
```
Then enable it with `systemctl enable hibernate-resume.service`

Create the file `/etc/systemd/system/hibernate-preparation.service`

```
[Unit]
Description=Enable swap file and disable zram before hibernate
Before=systemd-hibernate.service

[Service]
User=root
Type=oneshot
ExecStart=/bin/bash -c "/usr/sbin/swapon /var/swap/swapfile && /usr/sbin/swapoff /dev/zram0"

[Install]
WantedBy=systemd-hibernate.service
```
Then enable it with `systemctl enable hibernate-preparation.service`



```
https://fedoramagazine.org/update-on-hibernation-in-fedora-workstation/
https://gist.github.com/eloylp/b0d64d3c947dbfb23d13864e0c051c67?permalink_comment_id=3936683#gistcomment-3936683
```

Done. Now you can patch and build a kernel that will let you hibernate with secure boot enabled.

## Before compiling a kernel.

You need to prepare your pc so that it can sign your compiled kernel with a certificate that will work with secure boot. The steps are listed below.

Important - once you run the `mokutil` to import your key to your UEFI database, you need to reboot.
On reboot you will be asked to confirm you want to add the key. Follow the steps listed.

You can check if you have succesfully added the cert to your UEFI bios by running this command: `mokutil --list-enrolled`

```
### Custom Machine owner key for secure boot
# add your user to /etc/pesign/users - we use editor nano to do that below
sudo nano /etc/pesign/users
# Allow kernel signing
sudo /usr/libexec/pesign/pesign-authorize
# Create key - we set the key name with the subject as your account name
openssl req -new -x509 -newkey rsa:2048 -keyout "key.pem" -outform DER -out "cert.der" -nodes -days 36500 -subj "/CN=$USER"
# Import key to UEFI database.
mokutil --import "cert.der"
# You have to reboot the system after importing the key with "mokutil" to import the key via UEFI system
# Your computer will present some screens when you reboot asking you to confirm the import.
```
Rebooted your computer? You can proceed with the steps below.

```
# you can check that it is added with below command, it will appear with the -subj name you gave it (?)
mokutil --list-enrolled

# After rebooting create PKCS #12 key file and import it into the nss database
# nss database - https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/developer_guide/che-nsslib
# nss database is 'Network Security Services (NSS) database'

openssl pkcs12 -export -out key.p12 -inkey key.pem -in cert.der
certutil -A -i cert.der -n "$name" -d /etc/pki/pesign/ -t "Pu,Pu,Pu"
pk12util -i key.p12 -d /etc/pki/pesign
```

You can check the key is in pesign.

`certutil -L -d sql:/etc/pki/pesign/`

You should see an entry with the text `Pu,Pu,Pu`, this is the cert you added.

You can proceed with the next step.


### Compiling kernel to enable hibernation with secureboot

In the guide on `https://community.frame.work/t/guide-fedora-36-hibernation-with-enabled-secure-boot-and-full-disk-encryption-fde-decrypting-over-tpm2/25474` the forum poster sets some variables to pass as arguments in the commands he executes. We will skip that, replace the arguments as required in the commands below.

```
### Setup build system
### this will setup 
rpmdev-setuptree
```
The above command will setup a folder structure in ~/rpmbuild/ that prepares a build environment for you.
Try and run `tree rpmbuild -L 2 -d` to see the structure that has been created.

```
### check your current kernel that you want to re-compile to allow for hibernation
uname -rv
`6.14.8-300.fedora.fc42.x86_64 #1 SMP PREEMPT_DYNAMIC Thu May 29 17:29:18 CEST 2025`

### download kernel sources - don't change the arch value, we want the src files for the build
### compare below value to your uname -rv output and adjust accordingly
koji download-build --arch=src kernel-6.14.11-300.fc42
rpm -Uvh kernel-6.14.11-300.fc42.src.rpm
```
Now your kernel sources are installed, you need to change directory and apply a patch to allow hibernation with secureboot.

```
cd ~/rpmbuild/SPECS

### Apply patches and customize kernel configuration
# Get patch to enable hibernate in lockdown mode (secure boot)
wget https://gist.githubusercontent.com/zididadaday/abc96cf47f95f3d36b2955363533932d/raw/2ca6f68414577cffb4649c3bf0494e0cfda53f14/0001-Add-a-lockdown_hibernate-parameter.patch -O ~/rpmbuild/SOURCES/0001-Add-a-lockdown_hibernate-parameter.patch

# Define patch in kernel.spec for building the rpms
# Patch2: 0001-Add-a-lockdown_hibernate-parameter.patch
sed -i '/^Patch999999/i Patch2: 0001-Add-a-lockdown_hibernate-parameter.patch' kernel.spec

# Add patch as ApplyOptionalPatch
sed -i '/^ApplyOptionalPatch linux-kernel-test.patch/i ApplyOptionalPatch 0001-Add-a-lockdown_hibernate-parameter.patch' kernel.spec

# Add custom kernel name. Here we call it fedora.
sed -i "s/# define buildid .local/%define buildid .fedora/g" kernel.spec

# Add machine owner key
sed -i "s/%define buildid \.fedora/%define buildid .fedora\n%define pe_signing_cert fedora/g" kernel.spec

# Install necessary dependencies for compiling the kernel
rpmbuild -bp kernel.spec
```

Now we can compile the kernel.
```
### we prefix the command with time to see how long it takes.
### change arch to match your architecture
time rpmbuild -bb --with baseonly --without debuginfo --target=x86_64 kernel.spec | tee ~/build-kernel_`date '+%F_%H:%M:%S'.txt`.log
```

Compilation should be succesful, otherwise check what the errors say.


`https://gist.github.com/zididadaday/abc96cf47f95f3d36b2955363533932d`

### Installing the compiled kernel, update grub to run your new kernel allowing hibernation, fixing TPM2 disk unlocking

```
cd ~/rpmbuild/RPMS/x86_64/
sudo dnf install *.rpm
```

Now update your grub entry to include the boot parameter.
```
# this will add below argument to all boot entries in grub.
# not really needed for more than just our patched kernel, but does no harm.
grubby --args="lockdown_hibernate=1" --update-kernel=ALL
```
Check that the command has added the argument to your kernels entries

```
sudo grubby --info=ALL
```
If you want to remove an argument from all entries, you can run below command
```
# this will remove the lockdown_hibernate=1 argument from all grub entries
sudo grubby --remove-args="lockdown_hibernate=1" --update-kernel=ALL
```

## **-- IMPORTANT FOR KERNEL 6.14.8 and 6.14.9 --**


It seems  you need to add a resume and resume_offset entry to the kernel for hibernation to work properly. You can probably also do this for 6.14.6 if you want as well, it will not have any negative effects.

Run this command `sudo findmnt -no UUID -T /var/swap/swapfile` to find the UUID of the disk where you swap file is located.

Then run `sudo btrfs inspect-internal map-swapfile -r /var/swap/swapfile` to find the resume_offset value.

Run `sudo grubby --info=ALL` and note down the kernel path for the entry you want to update.

Then  `sudo grubby --args="resume=UUID=<uuid from above> resume_offset=<offset value from above>" --update-kernel="<kernel boot path>"`

Example: `sudo grubby --args="resume=UUID=591cccae-daa2-4bef-b993-33773a1f36b9 resume_offset=3202344" --update-kernel="/boot/vmlinuz-6.14.9-300.fedora.fc42.x86_64"`

**-- END --**


Update initramfs
```
# this will update the initramfs for the grubby entry for the kernel we just modified
sudo dracut -fv
```

You can reboot, you will find your laptop is prompting you to enter the passphrase to unlock your encrypted disk. A change to your system was detected (kernel change), so you cannot automatically decrypt the disk. You need to follow the steps below to wipe and re-enroll a new key.

`sudo systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+3+5+7+11+12+14" /dev/nvme0n1p3`

`sudo dracut -fv --regenerate-all`

Reboot to verify it worked.

You can now test that hibernation is working by running

```
systemctl hibernate
```

Make sure you have performed the hibernate prerequsities before testing hibernation with your patched kernel.

### Cleaning up your rpmbuild directory

If you have installed your patched kernel and all is working, clean-up your rpmbuild directory so you are ready for the next time you need to compile a patched kernel.

```
cd ~/

mv rpmbuild rpmbuild_`date '+%F_%H%M'
```

### Compiling a new kernel version (upgrade)

If your Fedora installation installs a new kernel version, your patched kernel will no longer load as default, disabling hibernation once more. You can set your old kernel as default or perform the kernel patching steps for the new kernel version.

`installing the new kernel source files`

`checking that the kernel.spec contains the new kernel details`


## Hibernation settings

Now that your computer can hibernate, you can customize a few things. We can set your computer to hibernate after it has been in standby for X amount of time, and we can add a menu option from the power action menu where you can trigger hibernation.

### Suspend then hibernate

`https://mitchellroe.dev/auto-hibernate.html`

Modify the file `/etc/systemd/sleep.conf.d/sleep.conf`

If it doesn't exist, you can create it.

```
[Sleep]
AllowSuspend=yes
AllowHibernation=yes
AllowSuspendThenHibernate=yes
HibernateMode=platform shutdown
# you can change this delay to another number, this is in seconds
# The amount of time the system spends in suspend mode before the system is automatically put into hibernate mode. 
HibernateDelaySec=720
HibernateOnACPower=yes
```
Modify the file `/etc/systemd/logind.conf.d/logind.conf`

If it doesn't exist, you can create it.

```
[Login]
# 3 actions, all should trigger a suspend then hibernate
HandleSuspendKey=suspend-then-hibernate
HandleLidSwitch=suspend-then-hibernate
HandleLidSwitchExternalPower=suspend-then-hibernate
```

```
https://www.freedesktop.org/software/systemd/man/latest/sleep.conf.d.html#
https://www.freedesktop.org/software/systemd/man/latest/logind.conf.d.html#
```
After restarting your changes will work.

### Adding hibernate options to the power menu

I wanted to add hibernate to my gnome power menu, where you can restart, sleep, shut down, log out.
I installed the extension below after customizing it to only see the menu options I wanted to see.

You need to install `gnome-shell-extension-hibernate-status`. I downloaded the source, configured it and compiled it, and then put it in the Extensions path. That way I could choose my menu options.

You need to install the `Extensions` add-in for Gnome.
```
sudo dnf install gnome-extensions-app
```

```
# cd
cd ~/Downloads

# download the extension
wget https://github.com/arelange/gnome-shell-extension-hibernate-status/archive/refs/heads/master.zip -O hibernate-extension.zip

# unzip the extension
unzip hibernate-extension.zip -d ~/

# cd
cd ~/gnome-shell-extension-hibernate-status-master

# to modify the menu choices, you have to modify org.gnome.shell.extensions.hibernate-status-button.gschema.xml in schemas
# change the menu items that you wish to hide's default field with false
nano schemas/org.gnome.shell.extensions.hibernate-status-button.gschema.xml

# make the extension
make

# compile the schema
glib-compile-schemas schemas

# make a folder for our extension
mkdir -p ~/.local/share/gnome-shell/extensions/hibernate-status@dromi

# copy over the extension
cp -r ~/gnome-shell-extension-hibernate-status-master/* ~/.local/share/gnome-shell/extensions/hibernate-status@dromi/

# enable the shell extension
gnome-extensions enable hibernate-status@dromi
```

Check the power menu for your new menu items. If you want to change the menu items, modify the schema and run make again, and copy the extension over once more.

I think you can actually change the settings from the Extensions GUI, where you enable and disable what menu items appear.

### Testing hibernation

You can test hibernation with the command `systemctl hibernate`
If it works your laptop will begin to hibernate, if it doesn't work the commandline will throw an error.

## Installation of Fedora

No special installation steps required.
However, if you wish to use Timeshift you need to make sure your homedir are setup correctly.
You can follow this guide: `https://www.geeksforgeeks.org/techtips/how-to-setup-timeshift-with-btrfs-in-fedora/`

### VPN - IPSec

I have a pfSense firewall running that I can connect to with VPN that I setup long ago. It has worked on iOS devices and my Windows 10 laptop. Now I need to configure it on Fedora 42.

Install ipsec v2 package (something about ipsec package changing name to strongswan?)
`sudo dnf install NetworkManager-strongswan-gnome`

If you are having issues with the VPN connecting, you can run `sudo journalctl -b -r | grep charon-nm` to see what error you are getting. It turns out that even though I had thought my internal CA was imported, it wasn't.

To import your CA pem (internal ca) (see guides below)

`https://discussion.fedoraproject.org/t/cannot-connect-to-ikev2-vpn-on-fedora-33-no-trusted-rsa-public-key-found/73547/20`


`https://ajmaradiaga.com/Adding-trusting-CA-Fedora/`

`https://www.reddit.com/r/Fedora/comments/1ar5tc9/problem_setting_up_vpn_with_ikev2_strongswan/`

`https://discussion.fedoraproject.org/t/ikev2-vpn-nm-strongswan-issues-with-routing/116967/3`

### force all traffic through the vpn

If you run the command `ip route` you should see that there is no route (no internet access). I had to run the command below once to fix this.

`sudo ip route add 0.0.0.0/1 via 172.20.10.1 dev wlp192s0`


`resolvectl dns wlp192s0 192.168.10.1`
`resolvectl domain wlp192s0 .local`

Wrote a script to run these commands on VPN connection.
`/etc/NetworkManager/dispatcher.d/01-vpn-dns.sh`

`https://askubuntu.com/questions/1111652/network-manager-script-when-interface-up`

### view heic images in Image Viewer

Not really sure which command did it.
I think the `-freeworld` one solved it.

`sudo dnf install libheif-freeworld`

`sudo dnf install heif-pixbuf-loader`




## Sources for this guide

`https://community.frame.work/t/guide-fedora-36-hibernation-with-enabled-secure-boot-and-full-disk-encryption-fde-decrypting-over-tpm2/25474`

`https://docs.fedoraproject.org/en-US/quick-docs/kernel-build-custom/`

`https://gist.github.com/kelvie/917d456cb572325aae8e3bd94a9c1350`

`https://gist.github.com/eloylp/b0d64d3c947dbfb23d13864e0c051c67?permalink_comment_id=3936683#gistcomment-3936683`


`https://community.frame.work/t/guide-setup-tpm2-autodecrypt/39005`

`https://community.frame.work/t/guide-framework-16-hibernate-w-swapfile-setup-on-fedora-40/53080`

## Other useful commands

Show Grub menu options on boot.
```
sudo grub2-editenv - unset menu_auto_hide
```

Hide Grub menu options on boot.
```
sudo grub2-editenv - set menu_auto_hide=1
```

Install package from newer Fedora

```
https://stackoverflow.com/questions/24968410/install-single-package-from-rawhide
```

Only mount network drives in /mnt on demand, disconnect when not in use
```
https://askubuntu.com/questions/1210867/remount-cifs-on-network-reconnect
```