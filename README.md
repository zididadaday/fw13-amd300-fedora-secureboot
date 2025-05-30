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
Fedora installation was 42, with Secure Boot enabled. 1TB NVME drive. 64GB DDR5 5600 MHZ CL40.

`https://fedoraproject.org/workstation/download/`

## Tested with Fedora kernel versions

- Fedora 6.14.6 fc42 - working
- Fedora 6.14.8 fc 42 - not working (can it be fixed?)
- Fedora 6.14.9 fc 42 - not working (can it be fixed?)

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

my main goal is to have enough security without having to enter the encryption passphrase every time I start up my laptop

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

1. `new kernel`
2. `hardware changes`

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

Then setup the resume from hibernation scripts:


Create file `/usr/lib/systemd/system-sleep/suspend-then-hibernate-post-suspend.sh` with content:
```
#!/bin/bash

if [ "$1" = "post" ] && [ "$2" = "suspend-then-hibernate" ] && [ "$SYSTEMD_SLEEP_ACTION" = "suspend" ]
then
    echo "suspend-then-hibernate (post suspend): enabling swapfile and disabling zram"
    /usr/sbin/swapon /swap/swapfile && /usr/sbin/swapoff /dev/zram0
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
ExecStart=/usr/sbin/swapoff /swap/swapfile

[Install]
WantedBy=suspend-then-hibernate.target
```

Then enable it with `systemctl enable suspend-then-hibernate-resume.service`


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

You should see an entry with 'Pu,Pu,Pu'

You can proceed with the next step.


### Compiling kernel to enable hibernation with secureboot

In the guide on `https://community.frame.work/t/guide-fedora-36-hibernation-with-enabled-secure-boot-and-full-disk-encryption-fde-decrypting-over-tpm2/25474` the forum poster sets some variables. We will skip that, replace the code as needed.

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
koji download-build --arch=src kernel-6.14.8-300.fc42
rpm -Uvh kernel-6.14.8-300.fc42.src.rpm
```
Now your kernel sources are installed, you need to change directory and apply a patch to allow hibernation with secureboot.

```
cd ~/rpmbuild/SPECS

### Apply patches and customize kernel configuration
# Get patch to enable hibernate in lockdown mode (secure boot)
wget https://gist.githubusercontent.com/zididadaday/abc96cf47f95f3d36b2955363533932d/raw/50749cb9e7726d855918bd6dca0394802793ab80/0001-Add-a-lockdown_hibernate-parameter.patch -O ~/rpmbuild/SOURCES/0001-Add-a-lockdown_hibernate-parameter.patch

# Define patch in kernel.spec for building the rpms
# Patch2: 0001-Add-a-lockdown_hibernate-parameter.patch
sed -i '/^Patch999999/i Patch2: 0001-Add-a-lockdown_hibernate-parameter.patch' kernel.spec

# Add patch as ApplyOptionalPatch
sed -i '/^ApplyOptionalPatch linux-kernel-test.patch/i ApplyOptionalPatch 0001-Add-a-lockdown_hibernate-parameter.patch' kernel.spec

# Add custom kernel name
sed -i "s/# define buildid .local/%define buildid .fedora/g" kernel.spec

# Add machine owner key
sed -i "s/%define buildid \.fedora/%define buildid .fedora\n%define pe_signing_cert fedora/g" kernel.spec

# Install necessary dependencies for compiling hte kernel
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
grubby --args="lockdown_hibernate=1" --update-kernel=ALL
```
Check that the command has added the argument to your kernels entries

```
sudo grubby --info=ALL
```

**-- IMPORTANT FOR KERNEL 6.14.8 and 6.14.9 --**

```
It seems  you need to add a resume and resume_offset entry to the kernel for hibernation to work properly
```

Run this command `sudo findmnt -no UUID -T /var/swap/swapfile` to find the UUID of the disk where you swap file is located.

Then run `sudo btrfs inspect-internal map-swapfile -r /var/swap/swapfile` to find the resume_offset value.

Run `sudo grubby --info=ALL` and note down the kernel path for the entry you want to update.

Then  `sudo grubby --args="resume=UUID=<uuid from above> resume_offset=<offset value from above>" --update-kernel="<kernel boot path>"`

Example: `sudo grubby --args="resume=UUID=591cccae-daa2-4bef-b993-33773a1f36b9 resume_offset=3202344" --update-kernel="/boot/vmlinuz-6.14.9-300.fedora.fc42.x86_64"`

**-- END --**


Update initramfs
`sudo dracut -fv --regenerate-all`


You can reboot, you will find your laptop is prompting you to enter the passphrase to unlock your encrypted disk. A change to your system was detected (kernel change), so you cannot automatically decrypt the disk. You need to follow the steps below to wipe and re-enroll a new key.

`sudo systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+3+5+7+11+12+14" /dev/nvme0n1p3`

`sudo dracut -fv --regenerate-all`

Reboot to verify it worked.

You can test that hibernation is enabled by running the below command.

`systemctl status hibernate.target`

Make sure you have performed the hibernate prerequsities before testing hibernation with your patched kernel.

### Cleaning up your rpmbuild directory

If you have installed your patched kernel and all is working, clean-up your rpmbuild directory so you are ready for the next time you need to compile a patched kernel.

```
cd ~/

mv rpmbuild rpmbuild_`date '+%F_%H%M'
```

### Compiling a new kernel version (upgrade)

If your Fedora installation installs a new kernel version, your modified kernel will disappear, disabling hibernation once more and you will have to compile a new patched kernel (which you would anyhow want to do).

`installing the new kernel source files`
`checking that the kernel.spec contains the new kernel details`


## Hibernation

You have to make some changes to get hibernation working on a secure boot setup.

More explained on this link.

`https://gist.github.com/eloylp/b0d64d3c947dbfb23d13864e0c051c67?permalink_comment_id=3936683#gistcomment-3936683`


### Testing hibernation

You can test hibernation with the command `systemctl hibernate`
If it works your laptop will begin to hibernate, if it doesn't work the commandline will throw an error.

#### Hibernation add-ins - menu items

I wanted to add hibernate to my gnome power menu, where you can restart, sleep, shut down, log out.
I installed the extension below after customizing it to only see the menu options I wanted to see.

## Installation of Fedora

No special installation steps required.
However, if you wish to use Timeshift you need to make sure your homedir are setup correctly.
You can follow this guide:


## Sources for this guide

`https://community.frame.work/t/guide-fedora-36-hibernation-with-enabled-secure-boot-and-full-disk-encryption-fde-decrypting-over-tpm2/25474`

`https://docs.fedoraproject.org/en-US/quick-docs/kernel-build-custom/`

`https://gist.github.com/kelvie/917d456cb572325aae8e3bd94a9c1350`

`https://gist.github.com/eloylp/b0d64d3c947dbfb23d13864e0c051c67?permalink_comment_id=3936683#gistcomment-3936683`


`https://community.frame.work/t/guide-setup-tpm2-autodecrypt/39005`

`https://community.frame.work/t/guide-framework-16-hibernate-w-swapfile-setup-on-fedora-40/53080`

## Other useful commands

Show Grub menu options on boot.
`sudo grub2-editenv - unset menu_auto_hide`
Hide Grub menu options on boot.
`sudo grub2-editenv - set menu_auto_hide=1`
