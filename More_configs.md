## Encryption

Load the encryption modules to be safe.

```
$ modprobe dm-crypt
$ modprobe dm-mod
```

Setting up encryption on our luks lvm partiton

```
$ cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme0n1p3
```

Enter in your password and **Keep it safe**. There is no "forgot password" here.


If you have a home partition, then initialize this as well

```
$ cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme1n1p1
```

Mount the drives:

```
$ cryptsetup open /dev/nvme0n1p3 luks_lvm
```

If you have a home parition:

```
$ cryptsetup open /dev/nvme1n1p1 arch-home
```

## Volume setup

Create the volume and volume group

```
$ pvcreate /dev/mapper/luks_lvm

$ vgcreate arch /dev/mapper/luks_lvm
```

Create a volume for your swap space. A good size for this is your RAM size + 2GB.
In my case, 64GB of RAM + 2GB = 66G.

```
$ lvcreate -n swap -L 66G arch
```

Next you have a few options depending on your setup

### Single Disk
If you have a single disk, you can either have a single volume for your root 
and home, or two separate volumes.

#### Single volume 

Single volume is the most straightforward. To do this, just use the entire
disk space for your root volume

```
$ lvcreate -n root -l +100%FREE arch
```

#### Two volumes

For two volumes, you'll need to estimate the max size you want for either your
root or home volumes. With a root volume of 200G, this looks like:

```
$ lvcreate -n root -L 200G arch
```

Then use remaining disk space for home

```
$ lvcreate -n home -l +100%FREE arch
```

### Dual Disk

If you have two disks, then create a single volume on your LVM disk.

```
$ lvcreate -n root -l +100%FREE arch
```

