# Proxmox Encrypted ZFS

This script demonstrates how to create a new Proxmox installation which has an encrypted ZFS root/`rpool`, and optionally a secondary zpool which is chain decrypted from the OS.

Dropbear is used to remotely unlock the server on boot. For security reasons, a separate SSH host keypair is used to prevent the primary keypair from being left decrypted on disk. So, the server is run on port 2222 to not cause conflicting SSH fingerprints with the host.

## Initial install

First, boot into the Proxmox installer and select one of the debug mode options. Eventually, it will get you to a shell. Type `exit` and wait for some more initialization and finally a second shell.

With your preferred editor, open `/usr/share/perl5/Proxmox/Install.pm`, and make a change to the line containing `zpool create`, adding the following flags:

```sh
-O encryption=on -O keyformat=raw -O keylocation=file:///root/zfs.key
```

Then create the temporary key and proceed to the installer:

```sh
dd if=/dev/random of=/root/zfs-temp.key bs=32 count=1 iflag=fullblock
exit
```

Follow the install prompts as normal, being sure to select ZFS, of course. When the installer completes, you'll be dropped into yet another shell. Run the following commands:

```sh
zpool import -a
zfs change-key -l -o keyformat=passphrase -o keylocation=prompt rpool
rm /root/zfs-temp.key
exit
```

## Chainloading additional pools

Replace `[zpool-name]` with your zpool name, of course.

```sh
dd if=/dev/random of=/etc/zfs/[zpool-name].key bs=32 count=1 iflag=fullblock
chmod 600 /etc/zfs/[zpool-name].key
ls -l /dev/disk/by-id
zpool create \
    -O encryption=on \
    -O keyformat=raw \
    -O keylocation=file:///etc/zfs/[zpool-name].key \
    raidz \
    ata-HGST_HUS728T8TALE6L4_VGJSJ39K \
    ata-HGST_HUS728T8TALE6L4_VGJSLT4K \
    ata-HGST_HUS728T8TALE6L4_VGJSW7HK \
    [zpool-name]
```

Create a file at `/etc/systemd/system/zfs-load-key.service` with the contents:

```
[Unit]
Description=Load encryption keys
DefaultDependencies=no
After=zfs-import.target
Before=zfs-mount.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/zfs load-key -a
StandardInput=tty-force

[Install]
WantedBy=zfs-mount.service
```

And finally:

```sh
systemctl enable zfs-load-key.service
```

## Adding Dropbear remote SSH unlock

First, install Dropbear for non-LUKS-based, `initramfs` systems, and delete the insecure host key types:

```sh
apt install dropbear dropbear-initramfs --no-install-recommends
cd /etc/dropbear/
rm dropbear_ecdsa_host_key dropbear_rsa_host_key
cd initramfs
rm dropbear_ecdsa_host_key dropbear_ecdsa_host_key.pub dropbear_rsa_host_key dropbear_rsa_host_key.pub
```

Add the following options to `dropbear.conf` and uncomment the line:

```
-I 360 -j -k -p 2222 -s -c zfsunlock
```

Finally, add your SSH client pubkey to `/etc/dropbear/initramfs/authorized_keys` since there is no password authentication.

When you're ready:

```sh
update-initramfs -u -k all
proxmox-boot-tool refresh
```
