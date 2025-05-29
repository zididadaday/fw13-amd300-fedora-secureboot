# Framework 13 (AMD Ryzen AI 300 Series) Fedora

## Content

- Enabling automatic decryption of your encrypted luks drive over TPM
- Secure Boot on Fedora
- Compiling kernel to allow for hibernation with secureboot
- Enabling hibernation/testing hibernation
- Sources

## Enabling automatic unlock of encrypted disk with TPM 2.0

With the commands below we are using the PCR settings 1+3+5+7+11+12+14.

`1 platform-config - Core system firmware data/host platform configuration; typically contains serial and model numbers, changes on basic hardware/CPU/RAM replacements`

`3 external-config - Extended or pluggable firmware data; includes information about pluggable hardware`

`5 boot-loader-config - GPT/Partition table; changes when the partitions are added, modified, or removed`

`7 secure-boot-policy - Secure Boot state; changes when UEFI SecureBoot mode is enabled/disabled, or firmware certificates (PK, KEK, db, dbx, ...) changes.`

`11 kernel-boot - measures the ELF kernel image, embedded initrd and other payload of the PE image it is placed in into this PCR.`

`12 kernel-config - measures the kernel command line into this PCR.`

`14 shim-policy - The shim project measures its "MOK" certificates and hashes into this PCR.`

I tested different PCR settings on my laptop, and found that I had to remove 8 and 9 because changes were constantly detected so the automatic unlock of my encrypted disk did not work.


```
# Add decryption key to tpm. 
systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=1+3+5+7+11+12+14 /dev/nvme0n1p3

# Wipe old keys and enroll new key. You have to execute this command again after a kernel upgrade.

sudo systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "1+3+5+7+11+12+14" /dev/nvme0n1p3

# verify TPM added

sudo systemd-cryptenroll /dev/nvme0n1p3


# Add tpm2 configuration option to /etc/crypttab
luks-$UUID UUID=disk-$UUID none tpm2-device=auto,luks,discard,tpm2-pcrs=1+3+5+7+11+12+14

# Update initramfs (to get necessary tpm2 libraries and parameters for decryption into initramfs)
dracut -fv --regenerate-all
```

`https://community.frame.work/t/guide-setup-tpm2-autodecrypt/39005`

`https://man.archlinux.org/man/systemd-cryptenroll.1`

`https://gist.github.com/jdoss/777e8b52c8d88eb87467935769c98a95`



### Changes that stop TPM 2.0 from unlocking encrypted disk

`new kernel`


## Secure Boot

This repository should include the nescessary steps for you to run Fedora with Secure Boot and hibernation enabled, and allow you to unlock your encrypted drives with TPM 2.0, so you don't have to enter a passphrase every time you boot your computer.

If you don't have secureboot enabled, you should start by enabling that, otherwise this guide will not help you.

You can check to see if secureboot is enabled on Fedora with the command: `mokutil --sb-state`
If it is enabled it will return `SecureBoot enabled`.

### Before compiling a kernel.

You need to prepare your pc so that it can sign your compiled kernel with a certificate that will work with secure boot.

The steps are listed below
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

# After rebooting create PKCS #12 key file and import it into the nss database
# nss database - https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/developer_guide/che-nsslib
# nss database is 'Network Security Services (NSS) database'

openssl pkcs12 -export -out key.p12 -inkey key.pem -in cert.der
certutil -A -i cert.der -n "$name" -d /etc/pki/pesign/ -t "Pu,Pu,Pu"
pk12util -i key.p12 -d /etc/pki/pesign
```


### Compiling kernel to enable secureboot

`sed -i "s/%define buildid \.$name/%define buildid .$name\n%define pe_signing_cert $name/g" kernel.spec`


### Installation of Fedora

No special installation steps required.
However, if you wish to use Timeshift you need to make sure your homedir are setup correctly.
You can follow this guide:

### Compiling a new kernel version (upgrade)

`installing the new kernel source files`
`checking that the kernel.spec contains the new kernel details`



### Enabling hibernation

You can test hibernation with the command `systemctl hibernate`
If it works your laptop will begin to hibernate, if it doesn't work the commandline will throw an error.



## Sources for this guide

`https://community.frame.work/t/guide-fedora-36-hibernation-with-enabled-secure-boot-and-full-disk-encryption-fde-decrypting-over-tpm2/25474`

`https://docs.fedoraproject.org/en-US/quick-docs/kernel-build-custom/`

`https://gist.github.com/kelvie/917d456cb572325aae8e3bd94a9c1350`
