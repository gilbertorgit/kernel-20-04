# Ubuntu 20-04 with UKSM 

## Authors

* **Gilberto Rampini**

## Optional Config - UKSM 

* Although it's totally optional but as you can se below, it likely to improve your server performance as well as enable some additional kernel features. 

```
#memory use before
root@ubuntu:/home/lab/apstra_script# free -g
              total        used        free      shared  buff/cache   available
Mem:            125          77          31           0          17          47
Swap:             0           0           0


#memory use after USKM 
root@ubuntu:/home/lab/apstra_script# free -g
              total        used        free      shared  buff/cache   available
Mem:            125          36          70           0          18          86
Swap:             0           0           0
```

```
apt-get -y install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc gnupg2 libelf-dev flex  bison dwarves

apt-get -y --no-install-recommends install kernel-package 

```


```

mkdir kernel-5-17-8-source
cd kernel-5-17-8-source/

wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.17.8.tar.sign

wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.17.8.tar.xz


unxz linux-5.17.8.tar.xz

```

```
root@jerry:~/kernel-5-17-8-source# gpg2 --verify linux-5.17.8.tar.sign
gpg: directory '/root/.gnupg' created
gpg: keybox '/root/.gnupg/pubring.kbx' created
gpg: assuming signed data in 'linux-5.17.8.tar'
gpg: Signature made Sun 15 May 2022 06:21:55 PM UTC
gpg:                using RSA key 647F28654894E3BD457199BE38DBBDC86092693E
gpg: Can't check signature: No public key

```

```
root@jerry:~/kernel-5-17-8-source# gpg --keyserver keyserver.ubuntu.com --recv-keys 647F28654894E3BD457199BE38DBBDC86092693E
gpg: key 38DBBDC86092693E: 1 duplicate signature removed
gpg: /root/.gnupg/trustdb.gpg: trustdb created
gpg: key 38DBBDC86092693E: public key "Greg Kroah-Hartman <gregkh@linuxfoundation.org>" imported
gpg: Total number processed: 1
gpg:               imported: 1

root@jerry:~/kernel-5-17-8-source# gpg2 --verify linux-5.17.8.tar.sign
gpg: assuming signed data in 'linux-5.17.8.tar'
gpg: Signature made Sun 15 May 2022 06:21:55 PM UTC
gpg:                using RSA key 647F28654894E3BD457199BE38DBBDC86092693E
gpg: Good signature from "Greg Kroah-Hartman <gregkh@linuxfoundation.org>" [unknown]
gpg:                 aka "Greg Kroah-Hartman <gregkh@kernel.org>" [unknown]
gpg:                 aka "Greg Kroah-Hartman (Linux kernel stable release signing key) <greg@kroah.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 647F 2865 4894 E3BD 4571  99BE 38DB BDC8 6092 693E

```

```
tar -xvf linux-5.17.8.tar 

cd linux-5.17.8

cp /boot/config-5.4.0-122-generic .config
```

```
- edit br_private.h and change from
vi net/bridge/br_private.h
#define BR_GROUPFWD_RESTRICTED (BR_GROUPFWD_STP | BR_GROUPFWD_MACPAUSE | \
                                BR_GROUPFWD_LACP)

- to
vi net/bridge/br_private.h
#define BR_GROUPFWD_RESTRICTED  0x0000u
```
```
wget https://raw.githubusercontent.com/dolohow/uksm/master/v4.x/uksm-4.20.patch

wget https://raw.githubusercontent.com/dolohow/uksm/master/v5.x/uksm-5.17.patch

patch -p1 < uksm-5.17.patch
```


```
change .config 

```
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
```

with the above we don't need it: I left it here for awareness only :
----------------------------------------------------------
vi .config

from 
CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem" 
to
CONFIG_SYSTEM_TRUSTED_KEYS=""
```
----------------------------------------------------------


This command will generate the kernel .config script 
```
make oldconfig
```
_You will see a few prompots before the UKSM option (around 18 steps to get the UKSM question). In my case was the option 1. 
Make sure you select the right option to enable UKSM and select the default option for everything else_

Compile your new kernel - 14 is the number of my processor core (It will take a while) 
```
Enable KSM for page merging (KSM) [Y/n/?] y
  Choose UKSM/KSM strategy
  > 1. Ultra-KSM for page merging (UKSM) (NEW)
    2. Legacy KSM implementation (KSM_LEGACY) (NEW)
```

```
make -j14 deb-pkg LOCALVERSION=-uksm

# You should see something like that 

root@ubuntu:~/kernel-4-9-40-source/linux-4.9.40# ls -la ../*.deb
-rw-r--r-- 1 root root    957376 Mar 20 11:07 ../linux-firmware-image-4.9.40-uksm_4.9.40-uksm-1_amd64.deb
-rw-r--r-- 1 root root  10546258 Mar 20 11:08 ../linux-headers-4.9.40-uksm_4.9.40-uksm-1_amd64.deb
-rw-r--r-- 1 root root  46015482 Mar 20 11:09 ../linux-image-4.9.40-uksm_4.9.40-uksm-1_amd64.deb
-rw-r--r-- 1 root root 479960848 Mar 20 11:29 ../linux-image-4.9.40-uksm-dbg_4.9.40-uksm-1_amd64.deb
-rw-r--r-- 1 root root    870920 Mar 20 11:08 ../linux-libc-dev_4.9.40-uksm-1_amd64.deb
```

Install the new kernel, update your grub and reboot your system
```
dpkg -i ../linux-headers-4.20.17-uksm_4.20.17-uksm-1_amd64.deb
dpkg -i ../linux-image-4.20.17-uksm_4.20.17-uksm-1_amd64.deb 
dpkg -i ../linux-libc-dev_4.20.17-uksm-1_amd64.deb 

sudo update-grub

root@lab:/home/lab/kernel-4-20-17-source/linux-4.20.17# uname -a
Linux lab 4.15.0-147-generic #151-Ubuntu SMP Fri Jun 18 19:21:19 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

reboot
```

```
lab@lab:~$ uname -a
Linux lab 4.20.17-uksm #1 SMP Thu Jul 1 20:12:07 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```
