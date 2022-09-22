# samba-with-ad

I have not found any complete guides on how to configure Samba to do access control on shares using Active Directory users and groups. I finally worked it out on an Ubunutu 22 VM and documented the steps.

Note: this guide reflects my understanding of the subject and I might be wrong... :)

## Install SSSD

Install required components

```
sudo apt install sssd-ad sssd-tools realmd adcli samba winbind
```

## Install Kerberos tools

This installs `klist` and `kinit`.

```
sudo apt install krb5-user
```

## Join realm

```
sudo realm join domain.tld
```

Note: replace `domain.tld` with your AD DNS name.

## Enable home directory creation

```
sudo pam-auth-update --enable mkhomedir
```

## Edit smb.conf

Edit `/etc/samba/smb.conf` with the following:

```
[global]
workgroup = GAAL
realm = GAAL.EU
security = ADS
kerberos method = secrets and keytab
```

Note: the last line tells samba to write computer account password to not only the samba secrets file but to keytab too.

Also add:

```
[global]
idmap config * : backend = tdb
idmap config * : range = 10000-19999
idmap config DOMAIN : backend = nss
idmap config DOMAIN : range = 200000-2000200000
```

Note: replace `DOMAIN` with your AD domain's name (not DNS name).

## Edit sssd.conf

Add the following line to `/etc/sssd/sssd.conf`:
```
ad_update_samba_machine_account_password = true
```
Note: this line tells sssd to write computer account password not only to keytab but to the samba secrets file too.

## Restart services
```
systemctl restart sssd
systemctl restart smbd
systemctl enable winbind
systemctl start winbind
```

## Join AD domain through net

```
sudo net ads join -U Administrator
```

Note: replace Administrator with the user that has permissions to join computers to the domain.

## Create a share

Create the share directory:

```
mkdir /srv/share1
```

Add a share to `/etc/samba/smb.conf`:
```
[share1]
path = /srv/share1
read only = no
```

Set permissions on the directory:

```
sudo chgrp "DOMAIN\\groupname" /srv/share1
```

Restart samba
```
systemctl restart smbd
```

## Test

Now you should be able to access the share using Kerberos auth from a domain joined Windows computer.
