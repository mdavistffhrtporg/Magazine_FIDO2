As previously discussed, there are options for unlocking LUKS partitions by using systemd-cryptenroll in coordination with a TPM2 chip and FIDO2 U2F security keys. By utilizing this, TPM2 chips and FIO2 U2F security keys offer an alternative to entering a password every time the system is booted. On other non-atomic editions, this is pretty straight forward. But, there is no consistent documentation on how to do this on Fedora Atomic Desktops.

**Note:** systemd-cryptenroll does not work with LUKS1. LUKS1 volumes must be upgraded to LUKS2 to work with systemd-cryptenroll. Fedora Linux has used LUKS2 by default since release 30, but users who are using LUKS volumes that were created on older Fedora Linux releases will need to upgrade their LUKS volumes. References: documentation, discussion

What are TPM2 and FIDO2?

A TPM2 chip is a type of storage with secure APSs that allow users to store secrets, such as decryption keys, that are protected by UEFI Secure Boot. Secure Boot ensures that code loaded upon boot are not modified and provides tamper protection. The TPM2 can store LUKS decryption keys and allows the sense of protection, with the user knowing that their system will not be accessed unless there is (ideally) no tampering. On the other hand, the Secure Boot and TPM2 keys are known to have weaknesses such that they need to be invalidated every time a system upgrade is done

While TPM2 chips may have weaknesses, FIDO2 U2F keys are phishing-resistent hardware tokens that are not vulnerable because they are independent from, and not tied to, the hardware. FIDO2 keys that are compliant with their standards are external storage devices with secure APIs that can store and retrieve secrets. They can be used both as multi- or sole- factoor authentication, and are known to reduce credential threat risk by 99.9%. Furthermore, FIDO2 secrets never leave the device, meaning that the client does all of the verification. These types of tokens mitigate the inherint risk of using only a password for authentication, as they require the user to be physically present for authentication. And, FIDO2 keys can block malicious actors by preventing brute-force attacks and locking out that person after a pre-determined number of wrong PIN attempts.

The first step in tying your LUKS encrypted disk to a TPM2 or FIDO2 key is to find out your filesystem path. lsblk can be used for this.


You will need to find out the parition number hosting your LUKS partition(s). This is necessary in later sections.

Choose whether you want TPM2 or FIDO2 decryption

How to use a TPM2 chip systemd-cryptenroll

First, add the tpm2-tss module to your dracut configuration

$ echo "add_dracutmodules+=\" tpm2-tss \"" | sudo tee /etc/dracut.conf.d/tpm2.conf
add_dracutmodules+=" tpm2-tss "

Second, enroll your TPM2 chip as an alternate decryption factor for you LUKS partitions. The --wipe-slot tpm2 option ensures that, after enrollment, all previous bindings are removed. This command is necessary every time you update the binding.

sudo systemd-cryptenroll --wipe-slot tpm2 --tpm2-device auto --tpm2-pcrs "0+1+2+3+4+5+7+9" /dev/nvme0n1p3

Third, update your /etc/crypttab by appending tpm2-device=auto,tpm2-pcrs=0+1+2+3+4+5+7+9 appropriately (depending on what PCRs you use).

Finally, rebuild your initramfs by using the following command:

rpm-ostree initramfs --enable --arg=--force-add --arg=tpm2-tss

How to use a FIDO2 U2F key with systemd-cryptenroll

First, add the fido2 module to your dracut configuration

$ echo "add_dracutmodules+=\" fido2 \"" | sudo tee /etc/dracut.conf.d/fido2.conf
add_dracutmodules+=" fido2 "

Second, enroll your FIDO2 key as an alternate decryption factor for your LUKS partitiojns.See systemd-cryptenroll(1) to find out how to change options such as touch or pin prompts. The default is to require both touch and pin.

$ sudo systemctld-cryptenroll --fido2-device auto /dev/nvme0n1p3

Third, update your /etc/crypttab by appending fido2-device=auto

Finally, rebuild your initramfs by using the following command:

rpm-ostree initramfs --enable --arg=--force-add --arg=fido2-device