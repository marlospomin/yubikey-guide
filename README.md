# YubiKey Guide

This guide will walk you through the process of setting up your YubiKey to function as a powerful security tool. By following these steps, you'll be able to use your YubiKey for:

* Secure encryption, signature, and authentication operations (**OpenPGP**).
* Seamlessly access SSH accounts and sign git commits (**FIDO2**).

For functionalities like Personal Identity Verification ([PIV](https://developers.yubico.com/yubico-piv-tool/YubiKey_PIV_introduction.html)), consult the official Yubico documentation.

**Disclaimer:** This guide is heavily inspired by drduh's exceptional [guide](https://github.com/drduh/YubiKey-Guide).

- [Prerequisites](#prerequisites)
- [Setup](#setup)
	- [Install Software](#install-software)
	- [Prepare Environment](#prepare-environment)
	- [Create Certify Key](#create-certify-key)
	- [Create Subkeys](#create-subkeys)
	- [Backup](#backup)
	- [Export Public Key](#export-public-key)
- [Configure YubiKey](#configure-yubikey)
	- [Enable KDF (Optional)](#enable-kdf-optional)
	- [Set Attributes](#set-attributes)
	- [Transfer Subkeys](#transfer-subkeys)
	- [Local Configuration](#local-configuration)
		- [Configure Touch](#configure-touch)
		- [Setup SSH (FIDO2)](#setup-ssh-fido2)
			- [Create Key Pair](#create-key-pair)
			- [SSH Agent Setup](#ssh-agent-setup)
- [Using YubiKey](#using-yubikey)
	- [GitHub](#github)
		- [Git Signing](#git-signing)
	- [SSH](#ssh)
	- [WSL](#wsl)
- [Updating Keys](#updating-keys)
	- [Renew Subkeys](#renew-subkeys)
	- [Rotate Subkeys](#rotate-subkeys)
- [Reset YubiKey](#reset-yubikey)
- [Additional Resources](#additional-resources)
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

To ensure the operation's security, start by configuring a reliable GPG environment.

1. Create a temporary directory and set it as the GnuPG directory:
```bash
export GNUPGHOME=$(mktemp -d -t gnupg-$(date +%Y-%m-%d)-XXXXXXXXXX)
```

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

6. Generate a passphrase for the Certify key:
```bash
export CERTIFY_PASS=#your_secure_and_long_password_here
```

7. Confirm all environment variables before proceeding:
```bash
printf "IDENTITY: %s\n" "$IDENTITY"
printf "KEY_TYPE: %s\n" "$KEY_TYPE"
printf "EXPIRATION: %s\n" "$EXPIRATION"
printf "CERTIFY_PASS: %s\n" "$CERTIFY_PASS"
```
### Create Certify Key

The primary key, Certify, is used to generate Subkeys for encryption, signature, and authentication. It should be stored offline in a dedicated, secure environment to minimize the risk of compromise. Certify should only be accessed to issue or revoke Subkeys.

>**Do not set an expiration date on the Certify key.**

1. Create the Certify key:
```bash
gpg --batch --passphrase "$CERTIFY_PASS" --quick-generate-key "$IDENTITY" "$KEY_TYPE" cert never
```

2. Set the Certify key identifier and fingerprint for use later:
```bash
export KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')
export KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')
```
### Create Subkeys

To create Signature, Encryption, and Authentication Subkeys, execute the following command using the previously specified key type, passphrase, and expiration:

```bash
for SUBKEY in sign encrypt auth ; do \
  gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
      --quick-add-key "$KEYFP" "$KEY_TYPE" "$SUBKEY" "$EXPIRATION"
done
```

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

To protect your keys, create a backup of Certify, Subkeys, and the public key:

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

Create an encrypted backup of your keys on a portable storage device and store it offline in a secure, durable location. 

To ensure data integrity, repeat this process multiple times using different storage devices, as they may fail over time.

1. Attach a portable storage device and check its label, in this case `/dev/sdb`:
```bash
sudo dmesg | tail
usb-storage 3-2:1.0: USB Mass Storage device detected
sd 2:0:0:0: [sdc] Attached SCSI removable disk
```

2. Clear the header to prepare for encryption:
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
export LUKS_PASS=#your_long_and_secure_password_here
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

9. Create an `ext2` filesystem:
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

Alternatively, connect another portable storage device or create a new partition on the existing one and follow the steps below.

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

| Name          | Default value | Capability                                                    |
| ------------- | ------------- | ------------------------------------------------------------- |
| **User** PIN  | `123456`      | Cryptographic operations (decrypt, sign, authenticate).       |
| **Admin** PIN | `12345678`    | Reset PIN, change Reset Code, add keys and owner information. |

Select PIN values for User and Admin access. These can be shorter than the Certify key passphrase due to limited brute-force possibilities. The User PIN should be easy to remember for daily use.

Ensure the User PIN is at least 6 characters and the Admin PIN is at least 8 characters. Both PINs should be composed of no more than 127 ASCII characters.

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

>Entering the wrong User PIN three times will lock it. To unlock it, you'll need to use the Admin PIN or Reset Code. If you enter the wrong Admin PIN or Reset Code three times, the data on your YubiKey will be permanently deleted.
### Enable KDF (Optional)

Key Derived Function ([KDF](https://developers.yubico.com/PGP/YubiKey_5.2.3_Enhancements_to_OpenPGP_3.4.html)) enables YubiKey to store the hash of PIN, preventing the PIN from being passed as plain text.

**This setting must be configured now or will be unavailable in the future.**

Enable KDF using the default Admin PIN of `12345678` (your actual PIN may differ):

```bash
gpg --command-fd=0 --pinentry-mode=loopback --card-edit <<EOF
admin
kdf-setup
12345678
EOF
```
### Set Attributes

Set the smart card details, configure the card with additional information like name, language, username, and public key URL.

1. Update the `login` data:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-card <<EOF
admin
login
$IDENTITY
$ADMIN_PIN
quit
EOF
```

2. Enter the edit context:
```bash
gpg --edit-card
```

3. Update your `name`:
```bash
gpg/card> name
```

4. Update your preferred `language`:
```bash
gpg/card> lang
```

5. Set the `URL` for your public key:
```bash
gpg/card> url
```

6. Quit:
```bash
gpg/card> quit
```

Run `gpg --card-status` to verify results.
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

4. Unplug and Insert another Yubikey.
5. Repeat the commands 1, 2 and 3 adding `save` before `EOF` to commit changes, like so:
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

Now, verify Subkeys have been moved to YubiKey with `gpg -K` and look for `ssb>`, for example:

```bash
sec   rsa4096/0xF0F2CFEB04341FB5 2024-01-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] YubiKey User <yubikey@example.org>
ssb>  rsa4096/0xB3CD10E502E19637 2024-01-01 [S] [expires: 2026-05-01]
ssb>  rsa4096/0x30CBE8C4B085B9F7 2024-01-01 [E] [expires: 2026-05-01]
ssb>  rsa4096/0xAD9E24E1B8CB9600 2024-01-01 [A] [expires: 2026-05-01]
```

The `>` after a tag indicates the key is stored on a smart card.

Reboot your machine to clear the ephemeral environment and complete setup.
### Local Configuration

1. Initialize GnuPG:

```bash
gpg -k
```

2. Import or create a hardened configuration in `~/.gnupg`.
3. Install the required packages (commands may vary):
```bash
sudo apt update
sudo apt install -y gnupg gnupg-agent scdaemon pcscd
``` 

4. Download or mount the public key:
```bash
sudo mkdir /mnt/public
sudo mount /dev/sdb2 /mnt/public
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

6. Assign ultimate trust by typing trust and selecting option 5:
```bash
gpg --command-fd=0 --pinentry-mode=loopback --edit-key $KEYID <<EOF
trust
5
y
save
EOF
```

7. Remove and re-insert YubiKey.

Verify the status with `gpg --card-status` which will list the available Subkeys.
#### Configure Touch

By default, YubiKey will perform cryptographic operations without requiring any action from the user after the key is unlocked once with the PIN.

To require a touch for each key operation, use YubiKey Manager and the Admin PIN to set key policy.

```bash
ykman openpgp keys set-touch enc on
ykman openpgp keys set-touch sig on
ykman openpgp keys set-touch aut on
```

> If you set the option to `fixed`, then it won't be possible to disable the touch requirement.
#### Setup SSH (FIDO2)

Before creating a new SSH key for your YubiKey, decide which additional authentication factors you want to use. The following table lists available factors and their corresponding commands:

| Factors                        | Description                                                                                       | Command                                                                        |
| ------------------------------ | ------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| No PIN or touch are required   | You will not be required to enter your FIDO2 PIN or touch your YubiKey each time to authenticate. | `ssh-keygen -t ed25519-sk -O resident -O no-touch-required`                    |
| PIN but no touch required      | Entering the PIN will be required but touching the physical key will not.                         | `ssh-keygen -t ed25519-sk -O resident -O verify-required -O no-touch-required` |
| No PIN but touch is required   | You will only need to touch the YubiKey to authenticate.                                          | `ssh-keygen -t ed25519-sk -O resident`                                         |
| A PIN and a touch are required | This is the most secure option, it requires both the PIN and touching to be used.                 | `ssh-keygen -t ed25519-sk -O resident -O verify-required`                      |

**Note:** Regardless of the command you choose, you'll need to physically touch your YubiKey to authenticate with services like GitHub and GitLab. While you might not be prompted for touch during `git commit`, you'll definitely need it for pull and push operations.
##### Create Key Pair

Once you've decided which option fits best for your threat model you will need to run one of the commands above. 

>If you are using a PIN you don't need to add an additional ssh passphrase as it's redundant due to the FIDO2 PIN being used instead.

```bash
ssh-keygen -t ed25519-sk -O resident -O verify-required
```
##### SSH Agent Setup

Now that you have generated a key which you can use, you will need to add it to your current ssh-agent session. 

1. You can do that by first starting the agent like so:
```bash
eval "$(ssh-agent -s)"
```

2. Then add the key on the YubiKey:
```bash
ssh-add -K
```

3. You can verify that the key was added by listing all the keys available in the current ssh-agent session:
```bash
ssh-add -l
```

At this stage feel free to delete the created key from your local directory (`ls -la`) as it's only temporary. The key can be retrieved at any moment using `ssh-keygen -K`.
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
### GitHub

In order to authenticate with GitHub you will have to add your new public key to your GitHub [profile](https://github.com/settings/keys).

1. To retrieve your key pair at any time, run the following command:

```bash
cd ~/.ssh
ssh-keygen -K
```

2. Remove `_rk` from the key name to allow SSH to automatically find and use your key. 
3. Copy the public key directly from the newly added files to the current folder, for example, `id_ed25519_sk.pub`.
4. Add it to your profile.

Now that we've added our ssh key to GitHub we can test that the setup works correctly by running:

```bash
ssh -T git@github.com
```

If this worked correctly you should be greeted by a welcoming message.
#### Git Signing

To be able to sign tags and commits you must first configure Git.

1. Set Git's GPG format to SSH:
```bash
git config --global gpg.format ssh
```

2. Turn on commit and tag signing respectively by default:
```bash
git config --global commit.gpgsign true
git config --global tag.gpgsign true
```

3. Set the `user.signingkey` to a pub key generated from a physical security key:
```bash
git config --global user.signingkey ~/.ssh/id_ed25519_sk.pub
```

4. Go to your Git provider (GitHub in this case) and add a new signing key (https://github.com/settings/keys).

All your commits and tags will now be signed.
### SSH

If you need to use your key to access private servers (like VPS or local machines), you can either use the key you just generated.

>The section below is entirely optional.

To enforce user presence update the SSH server (your server, VPS, etc.).

1. Edit the configuration file:
```bash
sudo nano /etc/ssh/sshd_config
```

2. Add the following at the bottom of the file:
```bash
PubkeyAuthOptions verify-required
```

3. Restart the server:
```bash
sudo systemctl restart sshd
```

A robust configuration might look like this:

```bash
# Support public key cryptography (includes FIDO2)
PubkeyAuthentication yes
# Enforce User Verification
PubkeyAuthOptions verify-required
# Public keys location
AuthorizedKeysFile .ssh/authorized_keys
# Allow root only with MFA
PermitRootLogin prohibit-password
# Disable password authentication
PasswordAuthentication no
PermitEmptyPasswords no
```

Add your public key to the `authorized_keys` file.

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_sk.pub user@host
```

Now you can use your key to login normally.
### WSL

To fully utilize FIDO2 support with WSL, make sure you're running a recent kernel version.

Check the version:

```bash
uname -r
```

If you have version `5.15.150.1` or above you don't have to update anything (if you do not, make sure to update it using the `wsl --update` command).

In **WSL**:

1. Create a new `udev` config file:
```bash
sudo nano /etc/udev/rules.d/99-custom-perms.rules
```

2. Insert the following and save the file:
```bash
SUBSYSTEM=="usb", MODE="0666"
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", TAG+="uaccess", MODE="0666"
```

3. Reload `udev`:
```bash
sudo udevadm control --reload
```

In **Windows**:
1. Install https://github.com/dorssel/usbipd-win.
2. List available devices to share:
```bash
usbipd list
```

3. Find your YubiKey (`USB Input Device, Microsoft Usbccid Smartcard Reader (WUDF)`) `BUSID`.
4. Bind it:
```bash
usbipd bind --busid 1-13
```

5. Attach the key to WSL:
```bash
usbipd attach --wsl --busid=1-13
```

Back in **WSL**:

1. Confirm the device was shared:
```bash
lsusb | grep Yubikey
```

You may now use your key without any problems (`ykman fido info` should work).
## Updating Keys

While PGP keys stored on YubiKeys offer enhanced security, they're not invulnerable. Physical compromise of the key or PIN, or vulnerabilities in firmware or random number generation, could potentially expose past messages. Therefore, regular Subkey rotation is recommended.

When a Subkey expires, you can either renew or replace it. Both actions require access to the Certify key.

Renewing Subkeys by updating their expiration is more convenient, as it signals continued possession of the Certify key.

Replacing Subkeys, although less convenient, offers enhanced security. New Subkeys cannot decrypt previous messages, authenticate with SSH, etc. Contacts will need updated public keys, and encrypted secrets must be decrypted and re-encrypted using the new Subkeys. This process is similar to losing and reprovisioning a YubiKey.

The choice between renewal and replacement depends on your personal identity management philosophy and threat assessment.

To renew or rotate Subkeys, follow the same process as generating keys: boot to a secure environment, install required software and disable networking (optional).

Connect the portable storage device with the Certify key and identify the disk label.

1. Decrypt and mount the encrypted volume:
```bash
sudo cryptsetup luksOpen /dev/sdb1 gnupg-secrets
sudo mkdir /mnt/encrypted-storage
sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage
```

2. Mount the non-encrypted public partition:
```bash
sudo mkdir /mnt/public
sudo mount /dev/sdb2 /mnt/public
```

3. Copy the original private key materials to a temporary working directory:
```bash
export GNUPGHOME=$(mktemp -d -t gnupg-$(date +%Y-%m-%d)-XXXXXXXXXX)
cd $GNUPGHOME
cp -avi /mnt/encrypted-storage/gnupg-*/* $GNUPGHOME
```

4. Confirm the identity is available, set the key id and fingerprint:
```bash
gpg -K
export KEYID=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^pub:/ { print $5; exit }')
export KEYFP=$(gpg -k --with-colons "$IDENTITY" | awk -F: '/^fpr:/ { print $10; exit }')
```

5. Recall the Certify key passphrase and set it, for example:
```bash
export CERTIFY_PASS=#your_certify_password_here
```

6. See the next section(s).
### Renew Subkeys

1. Determine the updated expiration:
```bash
export EXPIRATION=2026-09-01
export EXPIRATION=2y
```

2. Renew the Subkeys:
```bash
gpg --batch --pinentry-mode=loopback \
  --passphrase "$CERTIFY_PASS" --quick-set-expire "$KEYFP" "$EXPIRATION" \
  $(gpg -K --with-colons | awk -F: '/^fpr:/ { print $10 }' | tail -n "+2" | tr "\n" " ")
```

3. Export the updated public key:
```bash
gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc
```

4. Transfer the public key to the destination host and import it or publish to a public key server and download it:
```bash
gpg --import /mnt/public/*.asc

gpg --send-key $KEYID
gpg --recv $KEYID
```

The validity of the GnuPG identity will be extended, allowing it to be used again for encryption and signature operations.
### Rotate Subkeys

In order to rotate/replace the Subkeys you need to create new ones:

```bash
for SUBKEY in sign encrypt auth ; do \
  gpg --batch --pinentry-mode=loopback --passphrase "$CERTIFY_PASS" \
      --quick-add-key "$KEYFP" "$KEY_TYPE" "$SUBKEY" "$EXPIRATION"
done
```

**Note:** Previous Subkeys can be deleted from the identity.

Finish by copying new Subkeys to YubiKey.

1. Copy the new temporary working directory to encrypted storage, which is still mounted:
```bash
sudo cp -avi $GNUPGHOME /mnt/encrypted-storage
```

2. Unmount and close the encrypted volume:
```bash
sudo umount /mnt/encrypted-storage
sudo cryptsetup luksClose gnupg-secrets
```

3. Export the updated public key:
```bash
sudo mkdir /mnt/public
sudo mount /dev/sdb2 /mnt/public
gpg --armor --export $KEYID | sudo tee /mnt/public/$KEYID-$(date +%F).asc
sudo umount /mnt/public
```

4. Remove the storage device and follow the original steps to transfer new Subkeys (`4`, `5` and `6`) to YubiKey, replacing existing ones.

5. Reboot.
## Reset YubiKey

If PIN attempts are exceeded, the YubiKey is locked and must be Reset and set up again using the encrypted backup.

```bash
ykman openpgp reset
```
## Additional Resources

* https://github.com/drduh/YubiKey-Guide
* https://alexnorell.com/post/set-up-yubikey/
* https://gist.github.com/Kranzes/be4fffba5da3799ee93134dc68a4c67b
* https://developers.yubico.com/SSH/Securing_SSH_with_FIDO2.html
* https://developers.yubico.com/SSH/Securing_git_with_SSH_and_FIDO2.html
* https://dev.to/li/correctly-telling-git-about-your-ssh-key-for-signing-commits-4c2c
* https://github.com/dorssel/usbipd-win