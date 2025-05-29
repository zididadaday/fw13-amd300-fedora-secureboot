# Framework 13 (AMD Ryzen AI 300 Series) Fedora

## Content

- Secure Boot
- Installation
- Compiling kernel

### Secure Boot

This repository should include the nescessary steps for you to run Fedora with Secure Boot and hibernation enabled, and allow you to unlock your encrypted drives with TPM 2.0, so you don't have to enter a passphrase every time you boot your computer.

### Sources

`https://community.frame.work/t/guide-fedora-36-hibernation-with-enabled-secure-boot-and-full-disk-encryption-fde-decrypting-over-tpm2/25474`

### Preparing for compiling a kernel.

You need to prepare your pc so that it can sign your compiled kernel with a certificate that will work with secure boot.

The steps are listed below (source here)
Important - once you run the `mokutil` to import your key to your UEFI database, you need to reboot.
On reboot you will be asked to confirm you want to add the key. Follow the steps listed.

You can check if you have succesfully added the cert to your UEFI bios by running this command: `<lookup the command>`


```
### Custom Machine owner key for secure boot
# Allow kernel signing
sudo /usr/libexec/pesign/pesign-authorize
# Create key
openssl req -new -x509 -newkey rsa:2048 -keyout "key.pem" -outform DER -out "cert.der" -nodes -days 36500 -subj "$name"
# Import key to UEFI database.
mokutil --import "cert.der"
# You have to reboot the system after importing the key with "mokutil" to import the key via UEFI system
# After rebooting create PKCS #12 key file and import it into the nss database
openssl pkcs12 -export -out key.p12 -inkey key.pem -in cert.der
certutil -A -i cert.der -n "$name" -d /etc/pki/pesign/ -t "Pu,Pu,Pu"
pk12util -i key.p12 -d /etc/pki/pesign
```


### Compiling kernel to enable secureboot

### Installation

No special installation steps required.
However, if you wish to use Timeshift you need to make sure your homedir are setup correctly.
You can follow this guide:

### Compiling a new kernel version (upgrade)

`installing the new kernel source files`
`checking that the kernel.spec contains the new kernel details`

### Enabling unlock of encrypted disk with TPM 2.0

### Changes that stop TPM 2.0 from unlocking encrypted disk

`new kernel`