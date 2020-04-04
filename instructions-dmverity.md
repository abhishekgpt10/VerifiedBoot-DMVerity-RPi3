# Introduction 

### Verified Boot Stage 2: Dm-Verity

Device-Mapper’s “verity” target provides transparent integrity checking of block devices using a cryptographic digest provided by the kernel crypto API.\
Device-mapper is infrastructure in the Linux kernel that provides a generic way to create virtual layers of block devices.\
Device-mapper verity target provides read-only transparent integrity checking of block devices using kernel crypto API.\
The following instructions will help you for setiing up dm-verity on block device on which rootfs will be mounted.

## Part 1 - Building OPTEE-OS with Dm-Verity enabled

To enable dm-verity support, enable CONFIG_DM_VERITY in Device Drivers/Multi-device support configuration option (this we will do in later steps).\
To configure we need userspace components: device mapper library and veritysetup.\
veritysetup and other tools and libraries are provided by cryptsetup package in buildroot.\
Now we need to build again OPTEE-OS basically rootfs for initramfs with below changes. Add the below lines in `br-ext/configs/optee_generic` file.\

```
# dm-verify specific
BR2_PACKAGE_CRYPTSETUP=y
BR2_TARGET_ROOTFS_INITRAMFS=y
```

Now clean and rebuild, but first comment CONFIG_OF_CONTROL=y in `rpi-optee/u-boot/configs/rpi_3_defconfig`, otherwise it will pop an error for providing external dtb file.

```
#CONFIG_OF_CONTROL=y
```

Build again

```
cd ~/work/rpi-optee/build
make clean
make
```

Then copy the `rootfs.cpio.gz` to a different location and extract the content of rootfs in a tmp folder.

```
mkdir ../../cpio-rfs
cd ../../cpio-rfs
cp ../rpi-optee/out-br/images/rootfs.cpio.gz .
mkdir tmp_mnt
gunzip -c rootfs.cpio.gz | sh -c 'cd tmp_mnt/ && cpio -i'
```

Clean the kernel build

```
cd ../rpi-optee/linux
make mrproper
```

Now enable linux with dm-verity and initramfs. Modify/Add below line in `arch/arm64/configs/bcmrpi3_defconfig`.

```
CONFIG_INITRAMFS_SOURCE="/home/abhishek/work/cpio-rfs/rootfs.cpio.gz"
CONFIG_INITRAMFS_ROOT_UID=0
CONFIG_INITRAMFS_ROOT_GID=0
CONFIG_BLK_DEV_DM_BUILTIN=y
CONFIG_BLK_DEV_DM=y
CONFIG_DM_BUFIO=y
CONFIG_DM_VERITY=y
```

Now we have to build kernel again and make fitImage. Don't use signed u-boot now, build it with unsigned rpi3 dtb.

```
cd ../build
make linux

make u-boot-clean rpi3-u-boot-bin-clean rpi3-head-bin-clean
make u-boot rpi3-u-boot-bin

cd ../../fit
rm image.fit
../rpi-optee/u-boot/tools/mkimage -f image.its image.fit
```

Mount the sdcard boot partition and replace `image.fit` and `u-boot-rpi.bin`. `u-boot-rpi.bin` will be found in u-boot folder.



## Part 2 - Create partition for hash device and preparing hash device.

Create one more partition for hash device


```
fdisk /dev/sdx   # where sdx is the name of your sd-card
   > p             # prints partition table
   > d             # repeat until all partitions are deleted
   > n             # create a new partition
   > p             # create primary
   > 3             # make it the first partition
   > <enter>       # use the default sector
   > 2G            # Create 2GB partition
   > p             # double check everything looks right
   > w             # write partition table to disk.
```


Put the sdcard in RPi3 and boot, it will boot in initramfs. Run the below command for preparing hash device.

```
veritysetup create vnroot /dev/mmcblk0p2 /dev/mmcblk0p3
```

It will print output like
```
VERITY header information for /dev/mmcblk0p3
UUID:                   dc360f3f-ec65-459e-ad23-a453103c11f0
Hash type:              1
Data blocks:            1344768
Data block size:        4096
Hash block size:        4096
Hash algorithm:         sha256
Salt:                   267bfcef886b0bc0ac2ec36afc9ab92498fda5918e8fa549d24eb841d395820b
Root hash:              81d1a3905b991c2f176d0c1a0ab80e30d13f4ca01f017b9d3e4b665e770ff6a9
```

## Part 3 - Activate verity data device and do switch_root from initramfs init script.

Create a file containing root_hash in initramfs and modify init script.

```
cd ../cpio-rfs
# take your generated root_hash
echo '81d1a3905b991c2f176d0c1a0ab80e30d13f4ca01f017b9d3e4b665e770ff6a9' > tmp_mnt/etc/root-hash
```

Replacethe `tmp_mnt/init` file with below content.

```
#!/bin/sh

# devtmpfs does not get automounted for initramfs
/bin/mount -t devtmpfs devtmpfs /dev
/bin/mkdir -p /dev/pts
/bin/mkdir -p /dev/shm
/bin/mount -n -t proc proc /proc
/bin/mount -n -t sysfs sysfs /sys
/bin/mount -n -t tmpfs tmpfs /run

# Do your stuff here.
echo "This script just mounts and boots the rootfs, nothing else!"

PATH=$PATH:/bin:/sbin:/usr/bin:/usr/sbin

#get the root hash value, keep this line
ROOT_HASH=`cat /etc/root-hash`
root=/dev/mmcblk0p2
hashtable=/dev/mmcblk0p3

echo "Creating dm-verity block device"
/usr/sbin/veritysetup create vnroot $root $hashtable $ROOT_HASH --debug
retval=$?
if [ $retval -ne 0 ]
then
	echo -e "veritysetup create vnroot FAILED  with a return value $retval\n"
fi


mkdir /newroot
echo "Mounting dm-verity block device in newroot"
/bin/mount -t ext4 -o ro /dev/mapper/vnroot /newroot/

/bin/mount --move /dev /newroot/dev
/bin/mount --move /proc /newroot/proc
/bin/mount --move /sys /newroot/sys
/bin/mount --move /run /newroot/run

cd /newroot
echo "<-------------------------------------------Hello-from-Abhishek------------------------------------------------------>"

# Boot the real thing.
exec switch_root -c /dev/console . "sbin/init"

echo "Switching failed, dropping to console"
exec 0</dev/console
exec 1>/dev/console
exec 2>/dev/console

exec /sbin/getty -L  console 0 vt100 # GENERIC_SERIAL
```

Repack the rootfs after modifications.

```
rm -rf rootfs.cpio.gz
sh -c 'cd tmp_mnt/ && find . | cpio -H newc -o' | gzip -9 > rootfs.cpio.gz
```

Rebuild the kernel with updated initramfs then make fitImage

```
cd ../rpi-optee/linux/
make mrproper
cd ../build
make linux

cd ../../fit
rm image.fit
../rpi-optee/u-boot/tools/mkimage -f image.its image.fit
```

## Part 4 - Build signed u-boot and fitImage, TEST....

Clean u-boot and add CONFIG_OF_CONTROL in `rpi-optee/u-boot/configs/rpi_3_defconfig` which we commented at starting.

```
CONFIG_OF_CONTROL=y
``` 

Delete signed dtb, copy unsigned dtb and sign dtb and image.fit

```
rm bcm2710-rpi-3-b-pubkey.dtb
cp ../rpi-optee/linux/arch/arm64/boot/dts/broadcom/bcm2710-rpi-3-b.dtb bcm2710-rpi-3-b-pubkey.dtb
../rpi-optee/u-boot/tools/mkimage -F -k keys/ -K bcm2710-rpi-3-b-pubkey.dtb -r image.fit

make u-boot-clean rpi3-u-boot-bin-clean rpi3-head-bin-clean
make EXT_DTB=../../fit/bcm2710-rpi-3-b-pubkey.dtb u-boot rpi3-u-boot-bin
```

Mount the sdcard boot partition and replace `image.fit` and `u-boot-rpi.bin`. `u-boot-rpi.bin` will be found in u-boot folder.\
Boot the system.

2-Stage Verified Boot for RPi3 is completed. Below is the boot log.

```
U-Boot 2016.03-gbc6d93cf5e-dirty (Apr 01 2020 - 18:02:11 +0530)

DRAM:  944 MiB
RPI 3 Model B (0xa22082)
boot regs: 0x00000000 0x00000000 0x00000000 0x00000000
MMC:   bcm2835_sdhci: 0
reading uboot.env
In:    serial
Out:   lcd
Err:   lcd
Net:   Net Initialization Skipped
No ethernet found.
starting USB...
USB0:   Core Release: 2.80a
scanning bus 0 for devices... 3 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found
       scanning usb for ethernet devices... 1 Ethernet Device(s) found
Hit any key to stop autoboot:  0
reading image.fit
107469272 bytes read in 9072 ms (11.3 MiB/s)
## Loading kernel from FIT Image at 1f000000 ...
   Using 'config-1' configuration
   Verifying Hash Integrity ... sha256,rsa2048:dev+ OK
   Trying 'kernel-1' kernel subimage
     Description:  default kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x1f0000c0
     Data Size:    107034624 Bytes = 102.1 MiB
     Architecture: AArch64
     OS:           Linux
     Load Address: 0x00080000
     Entry Point:  0x00080000
     Hash algo:    sha256
     Hash value:   5a89506e505b5fa492464c96dd3ef3108409daebd149c4fffb014296a58b6c32
   Verifying Hash Integrity ... sha256+ OK
## Loading fdt from FIT Image at 1f000000 ...
   Using 'config-1' configuration
   Trying 'fdt-1' fdt subimage
     Description:  device tree
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x25679f60
     Data Size:    13376 Bytes = 13.1 KiB
     Architecture: AArch64
     Hash algo:    sha256
     Hash value:   393b1cce5da3fd17b245d50481eb4f6ce20d7863b8ed85f5647f9b6e0c907613
   Verifying Hash Integrity ... sha256+ OK
   Booting using the fdt blob at 0x25679f60
## Loading loadables from FIT Image at 1f000000 ...
   Trying 'tee-1' loadables subimage
     Description:  atf
     Type:         Standalone Program
     Compression:  uncompressed
     Data Start:   0x256139ac
     Data Size:    419024 Bytes = 409.2 KiB
     Architecture: AArch64
     Load Address: 0x08400000
     Entry Point:  0x08400000
     Hash algo:    sha256
     Hash value:   caec40affe81052e4911200e88645c9d74f7d10105dade44b49744046a2f8e9a
   Verifying Hash Integrity ... sha256+ OK
   Loading loadables from 0x256139ac to 0x08400000
   Loading Kernel Image ... OK
   reserving fdt memory region: addr=0 size=1000
   reserving fdt memory region: addr=8000000 size=2000000
   Loading Device Tree to 000000003ab21000, end 000000003ab2743f ... OK

Starting kernel ...

## Transferring control to ARM TF (at address 8400000) (dtb at 3ab21000)...
NOTICE:  BL3-1: v1.1(debug):1da4e155
NOTICE:  BL3-1: Built : 17:13:57, Apr  1 2020
INFO:    BL3-1: Initializing runtime services
INFO:    BL3-1: Initializing BL3-2
D/TC:0 add_phys_mem:526 TEE_SHMEM_START type NSEC_SHM 0x08000000 size 0x00400000
D/TC:0 add_phys_mem:526 TA_RAM_START type TA_RAM 0x08800000 size 0x01000000
D/TC:0 add_phys_mem:526 VCORE_UNPG_RW_PA type TEE_RAM_RW 0x08466000 size 0x0039a000
D/TC:0 add_phys_mem:526 VCORE_UNPG_RX_PA type TEE_RAM_RX 0x08420000 size 0x00046000
D/TC:0 add_phys_mem:526 TEE_RAM_START type TEE_RAM_RO 0x08400000 size 0x00020000
D/TC:0 add_phys_mem:526 CONSOLE_UART_BASE type IO_NSEC 0x3f200000 size 0x00200000
D/TC:0 verify_special_mem_areas:464 No NSEC DDR memory area defined
D/TC:0 add_va_space:565 type RES_VASPACE size 0x00a00000
D/TC:0 add_va_space:565 type SHM_VASPACE size 0x02000000
D/TC:0 dump_mmap_table:698 type TEE_RAM_RO   va 0x08400000..0x0841ffff pa 0x08400000..0x0841ffff size 0x00020000 (smallpg)
D/TC:0 dump_mmap_table:698 type TEE_RAM_RX   va 0x08420000..0x08465fff pa 0x08420000..0x08465fff size 0x00046000 (smallpg)
D/TC:0 dump_mmap_table:698 type TEE_RAM_RW   va 0x08466000..0x087fffff pa 0x08466000..0x087fffff size 0x0039a000 (smallpg)
D/TC:0 dump_mmap_table:698 type SHM_VASPACE  va 0x08800000..0x0a7fffff pa 0x00000000..0x01ffffff size 0x02000000 (pgdir)
D/TC:0 dump_mmap_table:698 type NSEC_SHM     va 0x0a800000..0x0abfffff pa 0x08000000..0x083fffff size 0x00400000 (pgdir)
D/TC:0 dump_mmap_table:698 type IO_NSEC      va 0x0ac00000..0x0adfffff pa 0x3f200000..0x3f3fffff size 0x00200000 (pgdir)
D/TC:0 dump_mmap_table:698 type RES_VASPACE  va 0x0ae00000..0x0b7fffff pa 0x00000000..0x009fffff size 0x00a00000 (pgdir)
D/TC:0 dump_mmap_table:698 type TA_RAM       va 0x0b800000..0x0c7fffff pa 0x08800000..0x097fffff size 0x01000000 (pgdir)
D/TC:0 core_mmu_entry_to_finer_grained:653 xlat tables used 1 / 5
D/TC:0 core_mmu_entry_to_finer_grained:653 xlat tables used 2 / 5
I/TC:
D/TC:0 init_canaries:164 #Stack canaries for stack_tmp[0] with top at 0x849a938
D/TC:0 init_canaries:164 watch *0x849a93c
D/TC:0 init_canaries:164 #Stack canaries for stack_tmp[1] with top at 0x849b178
D/TC:0 init_canaries:164 watch *0x849b17c
D/TC:0 init_canaries:164 #Stack canaries for stack_tmp[2] with top at 0x849b9b8
D/TC:0 init_canaries:164 watch *0x849b9bc
D/TC:0 init_canaries:164 #Stack canaries for stack_tmp[3] with top at 0x849c1f8
D/TC:0 init_canaries:164 watch *0x849c1fc
D/TC:0 init_canaries:165 #Stack canaries for stack_abt[0] with top at 0x849ce38
D/TC:0 init_canaries:165 watch *0x849ce3c
D/TC:0 init_canaries:165 #Stack canaries for stack_abt[1] with top at 0x849da78
D/TC:0 init_canaries:165 watch *0x849da7c
D/TC:0 init_canaries:165 #Stack canaries for stack_abt[2] with top at 0x849e6b8
D/TC:0 init_canaries:165 watch *0x849e6bc
D/TC:0 init_canaries:165 #Stack canaries for stack_abt[3] with top at 0x849f2f8
D/TC:0 init_canaries:165 watch *0x849f2fc
D/TC:0 init_canaries:167 #Stack canaries for stack_thread[0] with top at 0x84a1338
D/TC:0 init_canaries:167 watch *0x84a133c
D/TC:0 init_canaries:167 #Stack canaries for stack_thread[1] with top at 0x84a3378
D/TC:0 init_canaries:167 watch *0x84a337c
D/TC:0 init_canaries:167 #Stack canaries for stack_thread[2] with top at 0x84a53b8
D/TC:0 init_canaries:167 watch *0x84a53bc
D/TC:0 init_canaries:167 #Stack canaries for stack_thread[3] with top at 0x84a73f8
D/TC:0 init_canaries:167 watch *0x84a73fc
I/TC:  OP-TEE version: 3.2.0 #1 Wed Apr  1 11:43:19 UTC 2020 aarch64
D/TC:0 tee_ta_register_ta_store:534 Registering TA store: 'REE' (priority 10)
D/TC:0 tee_ta_register_ta_store:534 Registering TA store: 'Secure Storage TA' (priority 9)
D/TC:0 mobj_mapped_shm_init:559 Shared memory address range: 8800000, a800000
I/TC:  Initialized
D/TC:0 init_primary_helper:917 Primary CPU switching to normal world boot
INFO:    BL3-1: Preparing for EL3 exit to normal world
INFO:    BL3-1: Next image address = 0x80000
INFO:    BL3-1: Next image spsr = 0x3c9
D/TC:  generic_boot_cpu_on_handler:956 cpu 1: a0 0x0
D/TC:  init_secondary_helper:941 Secondary CPU Switching to normal world boot
D/TC:  generic_boot_cpu_on_handler:956 cpu 2: a0 0x0
D/TC:  init_secondary_helper:941 Secondary CPU Switching to normal world boot
D/TC:  generic_boot_cpu_on_handler:956 cpu 3: a0 0x0
D/TC:  init_secondary_helper:941 Secondary CPU Switching to normal world boot
[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 4.6.3-17583-g7d976fc52ae3-dirty (abhishek@abhi-vm) (gcc version 6.2.1 20161016 (Linaro GCC 6.2-2016.11) ) #1 SMP Wed Apr 1 09:36:54 IST 2020
[    0.000000] Boot CPU: AArch64 Processor [410fd034]
[    0.000000] debug: ignoring loglevel setting.
[    0.000000] efi: Getting EFI parameters from FDT:
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 8 MiB at 0x000000003a000000
[    0.000000] On node 0 totalpages: 241664
[    0.000000]   DMA zone: 3776 pages used for memmap
[    0.000000]   DMA zone: 0 pages reserved
[    0.000000]   DMA zone: 241664 pages, LIFO batch:31
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.0 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: Trusted OS migration not required
[    0.000000] percpu: Embedded 21 pages/cpu @ffffffc03af87000 s45592 r8192 d32232 u86016
[    0.000000] pcpu-alloc: s45592 r8192 d32232 u86016 alloc=21*4096
[    0.000000] pcpu-alloc: [0] 0 [0] 1 [0] 2 [0] 3
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 237888
[    0.000000] Kernel command line: console=tty0 console=ttyS0,115200 root=/dev/mmcblk0p2 rw rootfs=ext4 smsc95xx.macaddr=b8:27:eb:bc:6c:63 ignore_loglevel dma.dmachans=0x7f35 rootwait 8250.nr_uarts=1 elevator=deadline fsck.repair=yes bcm2708_fb.fbwidth=1920 bcm2708_fb.fbheight=1080 vc_mem.mem_base=0x3dc00000 vc_mem.mem_size=0x3f000000
[    0.000000] log_buf_len individual max cpu contribution: 4096 bytes
[    0.000000] log_buf_len total cpu_extra contributions: 12288 bytes
[    0.000000] log_buf_len min size: 16384 bytes
[    0.000000] log_buf_len: 32768 bytes
[    0.000000] early log buf free: 14584(89%)
[    0.000000] PID hash table entries: 4096 (order: 3, 32768 bytes)
[    0.000000] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes)
[    0.000000] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes)
[    0.000000] software IO TLB [mem 0x35200000-0x39200000] (64MB) mapped at [ffffffc035200000-ffffffc0391fffff]
[    0.000000] Memory: 735832K/966656K available (6408K kernel code, 746K rwdata, 2304K rodata, 95060K init, 870K bss, 222632K reserved, 8192K cma-reserved)
[    0.000000] Virtual kernel memory layout:
[    0.000000]     modules : 0xffffff8000000000 - 0xffffff8008000000   (   128 MB)
[    0.000000]     vmalloc : 0xffffff8008000000 - 0xffffffbdbfff0000   (   246 GB)
[    0.000000]       .text : 0xffffff8008080000 - 0xffffff80086c1000   (  6404 KB)
[    0.000000]     .rodata : 0xffffff80086c1000 - 0xffffff8008904000   (  2316 KB)
[    0.000000]       .init : 0xffffff8008904000 - 0xffffff800e5d9000   ( 95060 KB)
[    0.000000]       .data : 0xffffff800e5d9000 - 0xffffff800e693800   (   746 KB)
[    0.000000]     vmemmap : 0xffffffbdc0000000 - 0xffffffbfc0000000   (     8 GB maximum)
[    0.000000]               0xffffffbdc0000000 - 0xffffffbdc0ec0000   (    14 MB actual)
[    0.000000]     fixed   : 0xffffffbffe7fd000 - 0xffffffbffec00000   (  4108 KB)
[    0.000000]     PCI I/O : 0xffffffbffee00000 - 0xffffffbfffe00000   (    16 MB)
[    0.000000]     memory  : 0xffffffc000000000 - 0xffffffc03b000000   (   944 MB)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
[    0.000000] Hierarchical RCU implementation.
[    0.000000]  Build-time adjustment of leaf fanout to 64.
[    0.000000] NR_IRQS:64 nr_irqs:64 0
[    0.000000] Architected cp15 timer(s) running at 19.20MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x46d987e47, max_idle_ns: 440795202767 ns
[    0.000006] sched_clock: 56 bits at 19MHz, resolution 52ns, wraps every 4398046511078ns
[    0.000210] Console: colour dummy device 80x25
[    0.001382] console [tty0] enabled
[    0.001420] Calibrating delay loop (skipped), value calculated using timer frequency.. 38.40 BogoMIPS (lpj=76800)
[    0.001471] pid_max: default: 32768 minimum: 301
[    0.001779] Mount-cache hash table entries: 2048 (order: 2, 16384 bytes)
[    0.001813] Mountpoint-cache hash table entries: 2048 (order: 2, 16384 bytes)
[    0.002626] Disabling cpuset control group subsystem
[    0.002779] ftrace: allocating 22525 entries in 88 pages
[    0.058109] ASID allocator initialised with 65536 entries
[    0.059134] EFI services will not be available.
[    0.071569] Detected VIPT I-cache on CPU1
[    0.071627] CPU1: Booted secondary processor [410fd034]
[    0.083882] Detected VIPT I-cache on CPU2
[    0.083919] CPU2: Booted secondary processor [410fd034]
[    0.096135] Detected VIPT I-cache on CPU3
[    0.096170] CPU3: Booted secondary processor [410fd034]
[    0.096250] Brought up 4 CPUs
[    0.096395] SMP: Total of 4 processors activated.
[    0.096426] CPU: All CPU(s) started at EL2
[    0.097478] devtmpfs: initialized
[    0.105119] DMI not present or invalid.
[    0.105430] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.106146] pinctrl core: initialized pinctrl subsystem
[    0.106771] NET: Registered protocol family 16
[    0.108137] vdso: 2 pages (1 code @ ffffff80086c7000, 1 data @ ffffff800e5e0000)
[    0.108214] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.109423] DMA: preallocated 256 KiB pool for atomic allocations
[    0.109547] Serial: AMBA PL011 UART driver
[    0.111000] bcm2835-mbox 3f00b880.mailbox: mailbox enabled
[    0.111736] uart-pl011 3f201000.uart: could not find pctldev for node /soc/gpio@7e200000/uart0_pins, deferring probe
[    0.158645] HugeTLB registered 2 MB page size, pre-allocated 0 pages
[    0.161150] bcm2835-dma 3f007000.dma: DMA legacy API manager at ffffff8008048000, dmachans=0x1
[    0.161742] SCSI subsystem initialized
[    0.161911] usbcore: registered new interface driver usbfs
[    0.162001] usbcore: registered new interface driver hub
[    0.162123] usbcore: registered new device driver usb
[    0.162399] dmi: Firmware registration failed.
[    0.164058] raspberrypi-firmware soc:firmware: Attached to firmware from 2017-02-15 17:14
[    0.169568] clocksource: Switched to clocksource arch_sys_counter
[    0.215573] VFS: Disk quotas dquot_6.6.0
[    0.215681] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.215970] FS-Cache: Loaded
[    0.216377] CacheFiles: Loaded
[    0.226845] NET: Registered protocol family 2
[    0.227644] TCP established hash table entries: 8192 (order: 4, 65536 bytes)
[    0.227782] TCP bind hash table entries: 8192 (order: 5, 131072 bytes)
[    0.228020] TCP: Hash tables configured (established 8192 bind 8192)
[    0.228205] UDP hash table entries: 512 (order: 2, 16384 bytes)
[    0.228272] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes)
[    0.228563] NET: Registered protocol family 1
[    0.228910] RPC: Registered named UNIX socket transport module.
[    0.228939] RPC: Registered udp transport module.
[    0.228963] RPC: Registered tcp transport module.
[    0.228987] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    5.400971] hw perfevents: enabled with armv8_pmuv3 PMU driver, 7 counters available
[    5.402325] futex hash table entries: 1024 (order: 5, 131072 bytes)
[    5.402472] audit: initializing netlink subsys (disabled)
[    5.402552] audit: type=2000 audit(5.367:1): initialized
[    5.403502] workingset: timestamp_bits=44 max_order=18 bucket_order=0
[    5.419727] FS-Cache: Netfs 'nfs' registered for caching
[    5.420629] NFS: Registering the id_resolver key type
[    5.420701] Key type id_resolver registered
[    5.420726] Key type id_legacy registered
[    5.423075] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 250)
[    5.423255] io scheduler noop registered
[    5.423290] io scheduler deadline registered (default)
[    5.423555] io scheduler cfq registered
[    5.423752] pinctrl-bcm2835 3f200000.gpio: Starting probe
[    5.424663] pinctrl-bcm2835 3f200000.gpio: Probe successful!
[    5.425778] BCM2708FB: allocated DMA memory fa040000
[    5.425825] BCM2708FB: allocated DMA channel 0 @ ffffff8008048000
[    5.453264] Console: switching to colour frame buffer device 240x67
[    5.470208] Serial: 8250/16550 driver, 1 ports, IRQ sharing disabled
[    5.471188] console [ttyS0] disabled
[    5.471307] 3f215040.uart: ttyS0 at MMIO 0x3f215040 (irq = 61, base_baud = 31250000) is a 16550
[    6.232399] console [ttyS0] enabled
[    6.237050] bcm2835-rng 3f104000.rng: hwrng registered
[    6.242627] Unable to detect cache hierarchy from DT for CPU 0
[    6.262804] brd: module loaded
[    6.274013] loop: module loaded
[    6.278296] bcm2835_vchiq 3f00b840.vchiq: slot mem 00000000fa080000
[    6.284769] vchiq: vchiq_init_state: slot_zero = 0xffffff800f7fd000, is_master = 0
[    6.294263] Loading iSCSI transport class v2.0-870.
[    6.299961] usbcore: registered new interface driver smsc95xx
[    6.305929] dwc_otg: version 3.00a 10-AUG-2012 (platform bus)
[    6.512116] Core Release: 2.80a
[    6.515373] Setting default values for core params
[    6.520330] Finished setting default values for core params
[    6.726378] Using Buffer DMA mode
[    6.729798] Periodic Transfer Interrupt Enhancement - disabled
[    6.735800] Multiprocessor Interrupt Enhancement - disabled
[    6.741538] OTG VER PARAM: 0, OTG VER FLAG: 0
[    6.746031] Dedicated Tx FIFOs mode
[    6.749626] DMA Enabled, allocating DMA buffer
[    6.754399] Calling pcd_reinit
[    6.757768] WARN::dwc_otg_hcd_init:1052: FIQ DMA bounce buffers: virt = 0x0e7f0000 dma = 0xfa054000 len=9024
[    6.767904] FIQ FSM acceleration enabled for :
[    6.767904] Non-periodic Split Transactions
[    6.767904] Periodic Split Transactions
[    6.767904] High-Speed Isochronous Endpoints
[    6.767904] Interrupt/Control Split Transaction hack enabled
[    6.790872] dwc_otg: Microframe scheduler enabled
[    6.795789] WARN::hcd_init_fiq:459: MPHI regs_base at 0x0e7ea000
[    6.802023] dwc_otg 3f980000.usb: DWC OTG Controller
[    6.807164] dwc_otg 3f980000.usb: new USB bus registered, assigned bus number 1
[    6.814714] dwc_otg 3f980000.usb: irq 15, io mem 0x00000000
[    6.820492] Init: Port Power? op_state=1
[    6.824533] Init: Power Port (0)
[    6.828040] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
[    6.838952] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    6.850263] usb usb1: Product: DWC OTG Controller
[    6.859015] usb usb1: Manufacturer: Linux 4.6.3-17583-g7d976fc52ae3-dirty dwc_otg_hcd
[    6.871037] usb usb1: SerialNumber: 3f980000.usb
[    6.880512] hub 1-0:1.0: USB hub found
[    6.888406] hub 1-0:1.0: 1 port detected
[    6.896927] dwc_otg: FIQ enabled
[    6.904127] dwc_otg: NAK holdoff enabled
[    6.912011] dwc_otg: FIQ split-transaction FSM enabled
[    6.921164] Module dwc_common_port init
[    6.929213] usbcore: registered new interface driver usb-storage
[    6.939478] mousedev: PS/2 mouse device common for all mice
[    6.950069] device-mapper: ioctl: 4.34.0-ioctl (2015-10-28) initialised: dm-devel@redhat.com
[    6.962980] bcm2835-cpufreq: min=600000 max=1200000
[    6.972192] sdhci: Secure Digital Host Controller Interface driver
[    6.982469] sdhci: Copyright(c) Pierre Ossman
[    6.991062] sdhost: failed to allocate log buf
[    7.069592] mmc0: sdhost-bcm2835 loaded - DMA enabled (>1)
[    7.093657] Indeed it is in host mode hprt0 = 00021501
[    7.101507] mmc-bcm2835 3f300000.mmc: mmc_debug:0 mmc_debug2:0
[    7.101513] mmc-bcm2835 3f300000.mmc: DMA channel allocated
[    7.133715] sdhci-pltfm: SDHCI platform and OF driver helper
[    7.143973] ledtrig-cpu: registered to indicate activity on CPUs
[    7.145547] mmc0: host does not support reading read-only switch, assuming write-enable
[    7.154203] mmc0: new high speed SDHC card at address 1234
[    7.154807] mmcblk0: mmc0:1234 SA16G 14.5 GiB
[    7.167829]  mmcblk0: p1 p2 p3
[    7.193606] hidraw: raw HID events driver (C) Jiri Kosina
[    7.203287] usbcore: registered new interface driver usbhid
[    7.212939] usbhid: USB HID core driver
[    7.221235] optee: probing for conduit method from DT.
I/TC:  Dynamic shared memory is disabled
[    7.234602] optee: initialized driver
[    7.242590] Initializing XFRM netlink socket
[    7.250911] NET: Registered protocol family 17
[    7.259521] Key type dns_resolver registered
[    7.268196] Registered cp15_barrier emulation handler
[    7.277243] Registered setend emulation handler
[    7.286475] registered taskstats version 1
[    7.293602] usb 1-1: new high-speed USB device number 2 using dwc_otg
[    7.294562] Indeed it is in host mode hprt0 = 00001101
[    7.315747] 3f201000.uart: ttyAMA0 at MMIO 0x3f201000 (irq = 72, base_baud = 0) is a PL011 rev2
[    7.316232] mmc1: queuing unknown CIS tuple 0x80 (2 bytes)
[    7.338092] mmc1: queuing unknown CIS tuple 0x80 (3 bytes)
[    7.338484] of_cfs_init
[    7.338568] of_cfs_init: OK
[    7.362169] mmc1: queuing unknown CIS tuple 0x80 (3 bytes)
[    7.374891] mmc1: queuing unknown CIS tuple 0x80 (7 bytes)
[    7.476332] mmc1: new high speed SDIO card at address 0001
[    7.479351] Freeing unused kernel memory: 95060K (ffffff8008904000 - ffffff800e5d9000)
[    7.486396] usb 1-1: New USB device found, idVendor=0424, idProduct=9514
Thi[    7.486406] usb 1-1: New USB device strings: Mfr=0, Product=0, SerialNumber=0
s [    7.498373] hub 1-1:1.0: USB hub found
script just mounts[    7.498520] hub 1-1:1.0: 5 ports detected
 and boots the rootfs, nothing else!
Creating dm-verity block device
# cryptsetup 2.0.2 processing "/usr/sbin/veritysetup create vnroot /dev/mmcblk0p2 /dev/mmcblk0p3 81d1a3905b991c2f176d0c1a0ab80e30d13f4ca01f017b9d3e4b665e770ff6a9 --debug"
# Running command open.
# Allocating context for crypt device /dev/mmcblk0p3.
# Trying to open and read device /dev/mmcblk0p3 with direct-io.
# Initialising device-mapper backend library.
# Trying to load VERITY crypt type from device /dev/mmcblk0p3.
# Crypto backend (OpenSSL 1.0.2o  27 Mar 2018) initialized in cryptsetup library version 2.0.2.
# Detected kernel Linux 4.6.3-17583-g7d976fc52ae3-dirty aarch64.
# Reading VERITY header of size 512 on device /dev/mmcblk0p3, offset 0.
# Setting ciphertext data device to /dev/mmcblk0p2.
# Trying to open and read device /dev/mmcblk0p2 with direct-io.
# Activating volume vnroot by volume key.
# dm version   [ opencount flush ]   [16384] (*1)
# dm versions   [ opencount flush ]   [16384] (*1)
# Detected dm-ioctl version 4.34.0.
# Detected dm-verity version 1.3.0.
# Device-mapper backend running with UDEV support disabled.
# dm status vnroot  [ opencount flush ]   [16384] (*1)
# Trying to activate VERITY device vnroot using hash sha256.
# Calculated device size is 10758144 sectors (RW), offset 0.
# DM-UUID is CRYPT-VERITY-dc360f3fec65459ead23a453103c11f0-vnroot
# dm create vnroot CRYPT-VERITY-dc360f3fec65459ead23a453103c11f0-vnroot [ opencount flush ]   [16384] (*1)
# dm reload vnroot  [ opencount flush readonly securedata ]   [16384] (*1)
# dm resume vnroot  [ opencount flush readonly securedata ]   [16384] (*1)
# vnroot: Stacking NODE_ADD (254,0) 0:0 0600
# vnroot: Stacking NODE_READ_AHEAD 256 (flags=1)
# vnroot: Processing NODE_ADD (254,0) 0:0 0600
# Created /dev/mapper/vnroot
# vnroot: Processing NODE_READ_AHEAD 256 (flags=1)
# vnroot (254:0): read ahead is 256
# vnroot: retaining kernel rea[    7.751131] random: nonblocking pool is initialized
d ahead of 256 (requested 256)
# dm status vnroot  [ opencount flush ]   [16384] (*1)
# Verity volume vnroot status is V.
# Releasing crypt device /dev/mmcbl[    7.774032] EXT4-fs (dm-0): mounted filesystem with ordered data mode. Opts: (null)
k0p3 context.
# Releasing device-mapper back[    7.789653] usb 1-1.1: new high-speed USB device number 3 using dwc_otg
end.
Command successful.
Mounting dm-verity block device in newroot
<-------------------------------------------Hello-from-Abhishek------------------------------------------------------>
[    7.901924] usb 1-1.1: New USB device found, idVendor=0424, idProduct=ec00
[    7.912933] usb 1-1.1: New USB device strings: Mfr=0, Product=0, SerialNumber=0
[    7.927191] smsc95xx v1.0.4
[    7.984953] smsc95xx 1-1.1:1.0 eth0: register 'smsc95xx' at usb-3f980000.usb-1.1, smsc95xx USB 2.0 Ethernet, b8:27:eb:bc:6c:63
mount: mounting proc on /proc failed: Device or resour[    8.411799] EXT4-fs (dm-0): re-mounted. Opts: data=ordered
ce busy
Starting logging: OK
read-only file system detected...done
Starting tee-supplicant...
Starting network: OK

Welcome to Buildroot, type root to login
buildroot login: root
#
```

 










