# YubiKey Guide

This guide will walk you through the process of setting up your YubiKey to function as a powerful security tool. By following these steps, you'll be able to use your YubiKey for:

* Secure encryption, signature, and authentication operations (**OpenPGP**).
* Seamlessly access SSH accounts and sign git commits (**FIDO2**).

For functionalities like Personal Identity Verification (PIV), consult the official Yubico documentation.

**Disclaimer:** This guide is heavily inspired by drduh's exceptional [guide](https://github.com/drduh/YubiKey-Guide).
## Prerequisites 

To embark on your YubiKey journey, you'll need the following:

* At least two YubiKeys are recommended (except for FIDO-only Security Key Series and Bio Series YubiKeys). This provides a backup in case you lose your primary key;
* Two or more portable storage devices (USB flash drives, SD cards, etc.) will be used during the setup process.
## Setup

The security of your cryptographic keys is paramount. **Avoid generating keys on public or shared computers, or even your everyday personal device.** Instead, opt for a dedicated, secure system or a live bootable environment without persistent storage.

By taking these precautions, you can significantly enhance the protection of your sensitive cryptographic materials.

I personally chose [Arch Linux](https://wiki.archlinux.org/title/Installation_guide) for this tutorial, but you can use whatever you prefer.

**Make sure to perform the next steps in the hardened environment.**
### Install Software

Open a terminal window and install the required software packages based on your operating system:
#### Arch
```bash
sudo pacman -Syu gnupg pcsclite ccid yubikey-personalization
```
#### Debian/Ubuntu
```bash
sudo apt update
sudo apt -y upgrade
sudo apt -y install wget gnupg2 gnupg-agent dirmngr cryptsetup scdaemon pcscd yubikey-personalization yubikey-manager
```
#### macOS
```bash
brew install gnupg yubikey-personalization ykman pinentry-mac wget
```
### Prepare Environment

Follow the steps below to setup your local environment.

1. Create a temporary directory and set it as the GnuPG directory:
```bash
export GNUPGHOME=$(mktemp -d -t gnupg-$(date +%Y-%m-%d)-XXXXXXXXXX)
```
This creates a temporary directory for storing GnuPG data. It will be automatically deleted later.

2. Import or create a hardened configuration:
```bash
cd $GNUPGHOME

cat <<EOF > gpg.conf
personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
no-comments
no-emit-version
no-greeting
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
require-cross-certification
no-symkey-cache
armor
use-agent
throw-keyids
EOF
```

3. Create your identity:
```bash
export IDENTITY="YubiKey User <yubikey@example.org>"
```
Other formats are accepted but may be incompatible with certain use cases.

4. Select the desired algorithm and key size:
```bash
export KEY_TYPE=rsa4096
```

5. Set the expiration date of your Subkeys:
```bash
export EXPIRATION=2y
```
Longer periods can be set, it's up to you.

6. Generate a passphrase for the Certify key. It will be used infrequently to manage Subkeys and should be very strong:
```bash
export CERTIFY_PASS=
```

7. Confirm all environment variables before proceeding:
```bash
printf "IDENTITY: %s\n" "$IDENTITY"
printf "KEY_TYPE: %s\n" "$KEY_TYPE"
printf "EXPIRATION: %s\n" "$EXPIRATION"
printf "CERTIFY_PASS: %s\n" "$CERTIFY_PASS"
```
### Create Certify Key

The primary key to generate is the Certify key, which is responsible for issuing Subkeys for encryption, signature and authentication operations.

The Certify key should be kept offline at all times and only accessed from a dedicated and secure environment to issue or revoke Subkeys.

**Do not set an expiration date on the Certify key.**

1. Create the Certify key:
```bash
gpg --batch --passphrase "$CERTIFY_PASS" --quick-generate-key "$IDENTITY" "$KEY_TYPE" cert never
```
2. Set and view the Certify key identifier and fingerprint for use later:
```bash
export KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')

export KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')

printf "\nKey ID: %40s\nKey FP: %40s\n\n" "$KEYID" "$KEYFP"
```
### Create Subkeys

Use the following command to generate Signature, Encryption and Authentication Subkeys using the previously configured key type, passphrase and expiration:

```bash
for SUBKEY in sign encrypt auth ; do \
  gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
      --quick-add-key "$KEYFP" "$KEY_TYPE" "$SUBKEY" "$EXPIRATION"
done
```
### Verify Keys

List available secret keys:

```bash
gpg -K
```

The output will display **C**ertify, **S**ignature, **E**ncryption and **A**uthentication keys:

```bash
sec   rsa4096/0xF0F2CFEB04341FB5 2024-01-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example.org>
ssb   rsa4096/0xB3CD10E502E19637 2024-01-01 [S] [expires: 2026-05-01]
ssb   rsa4096/0x30CBE8C4B085B9F7 2024-01-01 [E] [expires: 2026-05-01]
ssb   rsa4096/0xAD9E24E1B8CB9600 2024-01-01 [A] [expires: 2026-05-01]
```
### Backup

First, save a copy of the Certify key, Subkeys and public key:

```bash
gpg --output $GNUPGHOME/$KEYID-Certify.key \
    --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
    --armor --export-secret-keys $KEYID

gpg --output $GNUPGHOME/$KEYID-Subkeys.key \
    --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
    --armor --export-secret-subkeys $KEYID

gpg --output $GNUPGHOME/$KEYID-$(date +%F).asc \
    --armor --export $KEYID
```

**Note:** If the commands above fail to execute, remove `--pinentry-mode=loopback --passphrase "$ENV_VAR"` arguments and set them manually via the interactive menu.

Create an encrypted backup on portable storage to be kept offline in a secure and durable location.

The following process is recommended to be repeated several times on multiple portable storage devices, as they are likely to fail over time.
#### Linux

1. Attach a portable storage device and check its label, in this case `/dev/sdb`:

```bash
sudo dmesg | tail
usb-storage 3-2:1.0: USB Mass Storage device detected
sd 2:0:0:0: [sdc] Attached SCSI removable disk
```

2. Zero the header to prepare for encryption:

```bash
sudo dd if=/dev/zero of=/dev/sdb bs=4M count=1
```

3. Remove and re-connect the storage device.
4. Erase and create a new partition table:
```bash
sudo fdisk /dev/sdb <<EOF
g
w
EOF
```

5. Create a small (at least 20 MB is recommended to account for the LUKS header size) partition for storing secret materials:
```bash
sudo fdisk /dev/sdb <<EOF
n


+20M
w
EOF
```

6. Generate/choose another unique password (ideally different from the one used for the Certify key) to protect the encrypted volume:
```bash
export LUKS_PASS=
```

This passphrase will also be used infrequently to access the Certify key and should be very strong.

7. Format the partition:
```bash
echo $LUKS_PASS | sudo cryptsetup -q luksFormat /dev/sdb1
```

8. Mount the partition:
```bash
echo $LUKS_PASS | sudo cryptsetup -q luksOpen /dev/sdb1 gnupg-secrets
```

9. Create an ext2 filesystem:
```bash
sudo mkfs.ext2 /dev/mapper/gnupg-secrets -L gnupg-$(date +%F)
```

10. Mount the filesystem and copy the temporary GnuPG working directory with key materials:
```bash
sudo mkdir /mnt/encrypted-storage
sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage
sudo cp -av $GNUPGHOME /mnt/encrypted-storage/
```

11. Unmount and close the encrypted volume:
```bash
sudo umount /mnt/encrypted-storage
sudo cryptsetup luksClose gnupg-secrets
```

Repeat the process for any additional storage devices (at least two are recommended).
### Export Public Key

Without the public key, it will not be possible to use GnuPG to decrypt nor sign messages. However, YubiKey can still be used for SSH authentication.

You can export the keys in a couple of different ways, the easiest being sending your key to a remote server, like:

```bash
gpg --send-key $KEYID
```

Alternatively, connect another portable storage device or create a new partition on the existing one.
#### Linux

1. Using the same `/dev/sdb` device as in the previous step, create a small (at least 20 Mb is recommended) partition for storing materials:
```bash
sudo fdisk /dev/sdb <<EOF
n


+20M
w
EOF
```

2. Create a filesystem and export the public key:
```bash
sudo mkfs.ext2 /dev/sdb2
sudo mkdir /mnt/public
sudo mount /dev/sdb2 /mnt/public
gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc
sudo chmod 0444 /mnt/public/*.asc
```

3. Unmount and remove the storage device:
```bash
sudo umount /mnt/public
```
## Configure YubiKey

YubiKey's PGP interface has its own PINs separate from other modules such as PIV:

| Name          | Default value | Capability                                                   |
| ------------- | ------------- | ------------------------------------------------------------ |
| **User** PIN  | `123456`      | cryptographic operations (decrypt, sign, authenticate)       |
| **Admin** PIN | `12345678`    | reset PIN, change Reset Code, add keys and owner information |

Determine the desired PIN values. They can be shorter than the Certify key passphrase due to limited brute-forcing opportunities; the User PIN should be convenient enough to remember for every-day use.

The User PIN must be at least 6 characters and the Admin PIN must be at least 8 characters. A maximum of 127 ASCII characters are allowed.

Set PINs manually or generate them, for example a 6 digit User PIN and 8 digit Admin PIN:

```bash
export ADMIN_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | fold -w8 | head -1)

export USER_PIN=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | fold -w6 | head -1)

printf "\nAdmin PIN: %12s\nUser PIN: %13s\n\n" "$ADMIN_PIN" "$USER_PIN"
```

1. Change the Admin PIN:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
3
12345678
$ADMIN_PIN
$ADMIN_PIN
q
EOF
```

2. Change the User PIN:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --change-pin <<EOF
1
123456
$USER_PIN
$USER_PIN
q
EOF
```

3. Remove and re-insert YubiKey.

>Three incorrect User PIN entries will cause it to become blocked and must be unblocked with either the Admin PIN or Reset Code. Three incorrect Admin PIN or Reset Code entries will destroy data on YubiKey.
### Enable KDF (Optional)

Key Derived Function (KDF) enables YubiKey to store the hash of PIN, preventing the PIN from being passed as plain text.

This setting must be done now, or ignored, as it cannot be setup later.

Enable KDF using the default Admin PIN of `12345678`:

```bash
gpg --command-fd=0 --pinentry-mode=loopback --card-edit <<EOF
admin
kdf-setup
12345678
EOF
```

### Set Attributes

Set the default attributes for now, we can update them later:

```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-card <<EOF
admin
login
$IDENTITY
$ADMIN_PIN
quit
EOF
```

Run `gpg --card-status` to verify results (*Login data field*).
### Transfer Subkeys

The Certify key passphrase and Admin PIN are required to transfer keys.

In this guide we'll proceed with two keys, but you may use as many as you'd like.

1. Transfer the signature key:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 1
keytocard
1
$CERTIFY_PASS
$ADMIN_PIN
EOF
```

2. Transfer encryption key:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 2
keytocard
2
$CERTIFY_PASS
$ADMIN_PIN
EOF
```

3. Transfer authentication key:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 3
keytocard
3
$CERTIFY_PASS
$ADMIN_PIN
EOF
```

4. Insert another Yubikey.
5. Repeat the commands 1, 2 and 3 adding `save` before `EOF` to commit changes. Example:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
key 1
keytocard
1
$CERTIFY_PASS
$ADMIN_PIN
save
EOF
```

>Transferring keys to YubiKey is a one-way operation which converts the on-disk key into a stub making it no longer usable to transfer to subsequent YubiKeys. 

**Note:** If the command doesn't work, try removing `--command-fd=0 --pinentry-mode=loopback` arguments and pass in the values manually.
#### Verify Transfer

Verify Subkeys have been moved to YubiKey with `gpg -K` and look for `ssb>`, for example:

```bash
sec   rsa4096/0xF0F2CFEB04341FB5 2024-01-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example.org>
ssb>  rsa4096/0xB3CD10E502E19637 2024-01-01 [S] [expires: 2026-05-01]
ssb>  rsa4096/0x30CBE8C4B085B9F7 2024-01-01 [E] [expires: 2026-05-01]
ssb>  rsa4096/0xAD9E24E1B8CB9600 2024-01-01 [A] [expires: 2026-05-01]
```

The `>` after a tag indicates the key is stored on a smart card.

Reboot to clear the ephemeral environment and complete setup.
### More YubiKey Configuration

1. Initialize GnuPG:

```bash
gpg -k
```

2. Import or create a hardened configuration:

```bash
cd ~/.gnupg

cat <<EOF > gpg.conf
personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
no-comments
no-emit-version
no-greeting
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
require-cross-certification
no-symkey-cache
armor
use-agent
throw-keyids
EOF
```

3. Install the required packages:
	* Debian/Ubuntu:
		```bash
		sudo apt update
		sudo apt install -y gnupg gnupg-agent scdaemon pcscd
		``` 

4. Download or mount the public key:
```bash
sudo mkdir /mnt/public
sudo mount /dev/sdc2 /mnt/public
gpg --import /mnt/public/*.asc
```

```bash
gpg --recv $KEYID
```

5. Determine the key ID:
```bash
gpg -k
export KEYID=0x
```

6. Assign ultimate trust by typing trust and selecting option 5 then quit:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
trust
5
y
save
EOF
```

7. Remove and re-insert YubiKey.
8. Verify the status with `gpg --card-status` which will list the available Subkeys.

Your key should now be ready to use, however there are more things to setup (optional).
#### Update Card Information

You must do the following on each of your keys.

1. Enter the edit context:
```bash
gpg --edit-card
```

2. Update your name:
```bash
gpg/card> name
```

3. Update your preferred language:
```bash
gpg/card> lang
```

4. Set the URL for your public key:
```bash
gpg/card> url
```

5. Quit:
```bash
gpg/card> quit
```

6. Verify the status with `gpg --card-status`.
#### Configure Touch

By default, YubiKey will perform cryptographic operations without requiring any action from the user after the key is unlocked once with the PIN.

To require a touch for each key operation, use YubiKey Manager and the Admin PIN to set key policy.

```bash
ykman openpgp keys set-touch enc on
ykman openpgp keys set-touch sig on
ykman openpgp keys set-touch aut on
```

> If you set the option to `fixed`, then it won't be possible to disable the touch requirement.
## Using YubiKey

Encrypt a message to yourself (useful for storing credentials or protecting backups):

```bash
echo "\ntest message string" | \
  gpg --encrypt --armor \
      --recipient $KEYID --output encrypted.txt
```

Decrypt the message:

```bash
gpg --decrypt --armor encrypted.txt
```

Now you can use it however you want.

