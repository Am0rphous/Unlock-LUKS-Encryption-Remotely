# Unlock-LUKS-Encryption-Remotely

Useful
- [Chkboot](https://github.com/grazzolini/chkboot) - chkboot is a tool to help detect changes to an unencrypted /boot partition.

How to unlock a LUKS encrypted Linux server remotely with [Dropbear SSH](https://github.com/mkj/dropbear). [Homepage](https://matt.ucc.asn.au/dropbear/dropbear.html)

You need
1. An encrypted physical/virtual Linux server with Linux, e.g. Ubuntu 20.04.
2. A SSH client on your machine with a generated ssh-keygen. We can create a 4096 bits key by running:
````powershell
ssh-keygen -b 4096
````

## 1. Encrypted server setup
Install the required packets on the server:
````powershell
sudo apt update && sudo apt dist-upgrade
sudo apt install dropbear initramfs-tools busybox
````
We might encounter problems with the install. Fix it with
````powershell
sudo apt -f install
````
We get a warning about `dropbear: WARNING: Invalid authorized_keys file, remote unlocking of cryptroot via SSH won't work!`. This is expected and we ignore this.

## 2. On your machine
Copy the output from your public SSH key
````powershell
cat ~/.ssh/id_rsa.pub 
````

## 3. Paste the content to your encrypted server 
Open the `autorized_keys` file and paste it into it.
````powershell
sudo nano /etc/dropbear-initramfs/authorized_keys
````

## 4. Create an unlock script
````powershell
sudo nano /etc/initramfs-tools/hooks/crypt_unlock.sh
````
and paste following content in it:
````powershell
#!/bin/sh

PREREQ="dropbear"

prereqs() {
  echo "$PREREQ"
}

case "$1" in
  prereqs)
    prereqs
    exit 0
  ;;
esac

. "${CONFDIR}/initramfs.conf"
. /usr/share/initramfs-tools/hook-functions

if [ "${DROPBEAR}" != "n" ] && [ -r "/etc/crypttab" ] ; then
cat > "${DESTDIR}/bin/unlock" << EOF
#!/bin/sh
if PATH=/lib/unlock:/bin:/sbin /scripts/local-top/cryptroot; then
kill \`ps | grep cryptroot | grep -v "grep" | awk '{print \$1}'\`
# following line kill the remote shell right after the passphrase has
# been entered.
kill -9 \`ps | grep "\-sh" | grep -v "grep" | awk '{print \$1}'\`
exit 0
fi
exit 1
EOF

  chmod 755 "${DESTDIR}/bin/unlock"

  mkdir -p "${DESTDIR}/lib/unlock"
cat > "${DESTDIR}/lib/unlock/plymouth" << EOF
#!/bin/sh
[ "\$1" == "--ping" ] && exit 1
/bin/plymouth "\$@"
EOF

  chmod 755 "${DESTDIR}/lib/unlock/plymouth"
  echo To unlock root-partition run "unlock" >> ${DESTDIR}/etc/motd

fi
````

and make it executable by running
````powershell
chmod +x /etc/initramfs-tools/hooks/crypt_unlock.sh
````

## 5. Create a static IP and change the default SSH port
If you skip this step DHCP is used.
Open the file
````powershell
sudo nano /etc/initramfs-tools/initramfs.conf 
````
and add a line which matches your network and network card
````powershell
#format [host ip]::[gateway ip]:[netmask]:[hostname]:[device]:[autoconf]
#([hostname] can be omitted)

IP=192.168.1.10::192.168.1.1:255.255.255.0::eth0:off
````
OPTIONAL: To change the default SSH port, open the file `/etc/dropbear-initramfs/config` and add e.g.
````powershell
DROPBEAR_OPTIONS="-s -I 30 -p 54000"
````
Explanation
````bash
-s: disable password logins
-I 30: idle_timeout, disconnect after 30 sec if no traffic is transmitted
-p 54000: listen on port 54000
````

## 6. Update initramfs
Command
````powershell
sudo update-initramfs -u
````
## 7. Disable DropBear service after successful decryption
This makes OpenSSH being used after.
````powershell
sudo update-rc.d dropbear disable
````

## 8. Reboot the encrypted server
````powershell
reboot
````

## 9. SSH into it as root
````powershell
ssh root@192.168.1.10
ssh root@192.168.1.10 [-i ~/.ssh/id_rsa]
````
  ### 9.1 Decrypt the drive
  ````powershell
  unlock                # might hang
  cryptroot-unlock      # or this
  ````
  Expected output
  ````powershell
  # cryptroot-unlock
  Please unlock disk sda5_crypt:
  cryptsetup: sda5_crypt set up successfully
  # 
  ````
  

### Sources
- [How do I get dropbear to actually work with initramfs?](https://askubuntu.com/questions/640815/how-do-i-get-dropbear-to-actually-work-with-initramfs)
- [https://github.com/mkj/dropbear](https://github.com/mkj/dropbear)
