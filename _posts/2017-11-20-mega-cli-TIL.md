---
title:  "TIL - Creating RAID volumes in MegaCLI"
date:   2017-11-20 00:00:00 -0600
categories: TIL MegaCLI MegaRAID
---
# Overview
I learned something new today. At work, we have a decent number (~300 or so) bare metal servers that my teams use for higher throughput workloads - things like [Cassandra](/categories/#cassandra) or Kafka. These servers all have anywhere from 8 to 24 hard drives and MegaRAID controllers. In the past, the hard drives/RAID/data directories were created for us by a different team, so we had no control over RAID level, JBOD, anything. Recently, we've wanted to change the RAID configuration on some of these servers. This brought us to the MegaCLI command-line utility. This tool turned out to be very hard to use, as there doesn't seem to be much documentation at all. I am going to try and document the process we went through here.

# Installing MegaCLI
First, we needed to make sure the RAID controller on our boxes was supported. You can check the RAID controller with this command: `lspci | grep -i raid`. You should see something like:
```
02:00.0 RAID bus controller: LSI Logic / Symbios Logic MegaRAID SAS 2208 [Thunderbolt] (rev 05)
```

According to [this article](http://hwraid.le-vert.net/wiki/LSIMegaRAIDSAS#a2.Linuxkerneldrivers), the RAID controller on our boxes is supported for MegaCLI. The README for MegaCLI also contains the following:
```
Supported Controllers
==================

MegaRAID SAS 8208ELP
MegaRAID SAS 8208XLP
MegaRAID SAS 8204ELP
MegaRAID SAS 8204XLP
```

Once we've confirmed support, it's time to install:
1. Download MegaCLI from the Broadcom website. You'll have to agree before [downloading](https://www.broadcom.com/support/download-search?dk=megacli)
2. Once the file is downloaded, unzip the files. There will be an RPM inside: `MegaCli-4.00.16-1.i386.rpm`
3. Install with `yum localinstall MegaCli-4.00.16-1.i386.rpm`
4. By default, MegaCLI is installed to the `/opt/MegaRAID/MegaCli/MegaCli64` file. You can make this easier to use by setting an alias: `alias megacli='/opt/MegaRAID/MegaCli/MegaCli64'`
5. Test that MegaCLI is working by listing all the physical drives in your server: `/opt/MegaRAID/MegaCli/MegaCli64 -PDList -a0` This should return something for each drive, similar to:

```
Enclosure Device ID: 32
Slot Number: 25
Drive's position: DiskGroup: 0, Span: 0, Arm: 1
Enclosure position: 1
Device Id: 25
WWN: 50000C0F02C81479
Sequence Number: 4
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SAS

Raw Size: 279.396 GB [0x22ecb25c Sectors]
Non Coerced Size: 278.896 GB [0x22dcb25c Sectors]
Coerced Size: 278.875 GB [0x22dc0000 Sectors]
Sector Size:  0
Firmware state: Online, Spun Up
Device Firmware Level: D1S4
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x50000c0f02c8147a
SAS Address(1): 0x0
Connected Port Number: 0(path0)
Inquiry Data: WD      WD3001BKHG      D1S4WX91E13KNW87
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None
Device Speed: 6.0Gb/s
Link Speed: 6.0Gb/s
Media Type: Hard Disk Device
Drive Temperature :42C (107.60 F)
PI Eligibility:  No
Drive is formatted for PI information:  No
PI: No PI
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s
Port-1 :
Port status: Active
Port's Linkspeed: Unknown
Drive has flagged a S.M.A.R.T alert : No
```

There are some important terms used here that I want to explain:
* Adapter: This is the actual RAID controller we're using. On all of our servers there is only one, and it's designated by the number zero. `-a0` in the command above is referring to "adapter 0."
* Enclosure Device ID: This is the physical chassis number the drive is attached to, represented by an ID. On our servers, all drives have the same ID, but this won't always be the case.
* Physical Drives: Actual physical (spinning or SSD) drives connected to the server, each will have an ID of `$EnclosureID:$DriveID`
* Virtual Drives: This is a virtual drive containing any number of physical drives in a RAID or JBOD configuration. These also have an ID similar to the adapter ID.

In our use case, we had two types of disk configuration we wanted: RAID 10 and JBOD.

# JBOD
Our first use case was to set up [JBOD](https://en.wikipedia.org/wiki/Non-RAID_drive_architectures#JBOD) for some of our Cassandra servers. The servers with SSDs installed had the following config:
* 8 SSD drives
  * 500 gb each

We followed these steps to set up JBOD with MegaCLI:
1. Validate all 8 drives appear: `/opt/MegaRAID/MegaCli/MegaCli64 -PDList -a0`. In our case, there are 8 entries that look like this:

```
Enclosure Device ID: 8
Slot Number: 7
Drive's position: DiskGroup: 1, Span: 1, Arm: 0
Enclosure position: 1
Device Id: 16
WWN: 55cd2e404b7746ef
Sequence Number: 2
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 447.130 GB [0x37e436b0 Sectors]
Non Coerced Size: 446.630 GB [0x37d436b0 Sectors]
Coerced Size: 446.102 GB [0x37c34800 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Online, Spun Up
Device Firmware Level: 0370
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x584b261c2fb68186
Connected Port Number: 0(path0)
Inquiry Data: BTWL5043014Z480QGN  INTEL SSDSC2BB480G4                     D2010370
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None
Device Speed: 6.0Gb/s
Link Speed: 6.0Gb/s
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :12C (53.60 F)
PI Eligibility:  No
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : Enabled
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s
Drive has flagged a S.M.A.R.T alert : No
```

2. List the current virtual drives: `/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -a0`. For our servers - this should already show a single virtual drive for the OS. It's important to note the physical drives that are part of this - as we don't want to use them for our new array. This could corrupt or even delete the OS on the server, so be careful.
3. Figure out the Enclosure Device ID of the 8 drives we want to JBOD: `/opt/MegaRAID/MegaCli/MegaCli64 -PDList -a0 | grep -e '^Enclosure Device ID:' | head -1 |  cut -f2- -d':' | xargs`
4. Figure out the slot number of the 8 drives we are going to JBOD. The only way I was able to do this was manually look through the output of the first command. In our case - the slot numbers were 1 - 8.
5. Set all the drives to "Good" in MegaCLI (this marks them as unconfigured but spun up): `/opt/MegaRAID/MegaCli/MegaCli64 -PDMakeGood -PhysDrv[$id:1,$id:2,$id:3,$id:4,$id:5,$id:6,$id:7,$id:8] -Force -a0` *note* the numbers 1 - 8 are the slot numbers of the disks, make sure the change these to match your slot numbers.
6. Check and see if JBOD support is enabled: `/opt/MegaRAID/MegaCli/MegaCli64 AdpGetProp EnableJBOD -aALL`. On all of our servers, this returns: `Adapter 0: JBOD: Disabled`, so we need to turn it on.
7. If JBOD is disabled from step 6, turn JBOD support on: `/opt/MegaRAID/MegaCli/MegaCli64 AdpSetProp EnableJBOD 1 -a0`
8. Set each disk from above to be in JBOD mode: `/opt/MegaRAID/MegaCli/MegaCli64 -PDMakeJBOD -PhysDrv[$id:1,$id:2,$id:3,$id:4,$id:5,$id:6,$id:7,$id:8] -a0`
9. Once the disks are set to JBOD, each one should appear to the OS. You can check with `lsblk`:

```
sdd                                8:48   0 279.5G  0 disk
sde                                8:64   0 279.5G  0 disk
sdf                                8:80   0 279.5G  0 disk
sdg                                8:96   0 279.5G  0 disk
sdl                                8:176  0 279.5G  0 disk
sdk                                8:160  0 279.5G  0 disk
sdn                                8:208  0 279.5G  0 disk
sdc                                8:32   0 279.5G  0 disk
```

sd* above is the disk id assigned by the OS. Use that in the commands below.

Now that we have the disks and the OS can see them, it's time to format them:
1. `mkfs.xfs -s size=4096 /dev/$disk_id -f`
2. Create a directory to mount the disk: `mkdir /data1/`
3. Mount the disk: `mount -t xfs -o noatime /dev/$disk_id /data1`
4. add an entry to fstab so it survives a reboot: `echo "/dev/${disk_id} /data1 xfs noatime" | sudo tee -a /etc/fstab`
5. Repeat for each disk.

You should now have 8 disks mounted in different directories on the server. I like to check with `df -h`:

```
Filesystem            Size  Used Avail Use% Mounted on
/dev/sdd              276G   50G  212G  19% /data1
/dev/sdl              276G   46G  216G  18% /data2
/dev/sdg              276G   54G  208G  21% /data3
/dev/sdf              276G   53G  209G  21% /data4
/dev/sdk              276G   52G  210G  20% /data5
/dev/sdc              276G   63G  199G  24% /data6
/dev/sde              276G   47G  215G  18% /data7
/dev/sdn              276G   48G  214G  19% /data8
```

# RAID 10
Another use case we had was to set our disks to [RAID 10.](https://en.wikipedia.org/wiki/Nested_RAID_levels#RAID_10_.28RAID_1.2B0.29) This is a combination of RAID 0 and 1, and allows your array to survive the failure of a disk. The servers we were configuring RAID 10 on had the following config:
* 24 hard drives
  * 1 TB each

To set these up in a RAID 10 array, follow these steps:
1. Validate all 24 drives appear: `/opt/MegaRAID/MegaCli/MegaCli64 -PDList -a0`. In our case, there are 24 entries that look like this:

```
Enclosure Device ID: 32
Slot Number: 21
Drive's position: DiskGroup: 1, Span: 0, Arm: 21
Enclosure position: 1
Device Id: 21
WWN: 5000C50056C0B778
Sequence Number: 4
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SAS

Raw Size: 931.512 GB [0x74706db0 Sectors]
Non Coerced Size: 931.012 GB [0x74606db0 Sectors]
Coerced Size: 931.0 GB [0x74600000 Sectors]
Sector Size:  0
Firmware state: Online, Spun Up
Device Firmware Level: AS09
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x5000c50056c0b779
SAS Address(1): 0x0
Connected Port Number: 0(path0)
Inquiry Data: SEAGATE ST91000640SS    AS099XG4QFD1
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None
Device Speed: 6.0Gb/s
Link Speed: 6.0Gb/s
Media Type: Hard Disk Device
Drive Temperature :25C (77.00 F)
PI Eligibility:  No
Drive is formatted for PI information:  No
PI: No PI
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s
Port-1 :
Port status: Active
Port's Linkspeed: Unknown
Drive has flagged a S.M.A.R.T alert : No
```

2. List the current virtual drives: `/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -a0`
3. Figure out the Enclosure Device ID ($id below) of the 24 drives we want to RAID: `/opt/MegaRAID/MegaCli/MegaCli64 -PDList -a0 | grep -e '^Enclosure Device ID:' | head -1 |  cut -f2- -d':' | xargs`
4. Figure out the slot number of the 24 drives. The only way I was able to do this was manually look through the output of the first command. In our case - the slot numbers were 0 - 23.
5. Set all the drives to "Good" in MegaCLI (this marks them as unconfigured but spun up):

```
/opt/MegaRAID/MegaCli/MegaCli64 -PDMakeGood -PhysDrv[$id:0,$id:1,$id:2,$id:3,$id:4,$id:5,$id:6,$id:7,$id:8,$id:9,$id:10,$id:11,$id:12,$id:13,$id:14,$id:15,$id:16,$id:17,$id:18,$id:19,$id:20,$id:21,$id:22,$id:23] -Force -a0
```
*note* the numbers 0 - 23 are the slot numbers of the disks, make sure the change these to match your slot numbers.

6. Set up the RAID 10 span:

```
/opt/MegaRAID/MegaCli/MegaCli64 -CfgSpanAdd -r10 -Array1[$id:0,$id:1,$id:2,$id:3,$id:4,$id:5,$id:6,$id:7,$id:8,$id:9,$id:10,$id:11] -Array2[$id:12,$id:13,$id:14,$id:15,$id:16,$id:17,$id:18,$id:19,$id:20,$id:21,$id:22,$id:23] -a0
```

7. Once the RAID array is created, it should appear as a single disk with `lsblk`:

```
sdb                                8:16   0  21.8T  0 disk
```

Now we follow the same steps as above to format/mount the disk.
1. `mkfs.xfs -f -d sunit=128,swidth=2048 -L data0 /dev/sdb`
2. Create a data directory: `mkdir /data0`
3. Add an entry to fstab: `echo "/dev/sdb /data0 xfs noatime" | sudo tee -a /etc/fstab`
4. Mount the drive: `mount /data0`

# Additional Settings
Some other things we needed to do were set things like readahead, disk scheduler, etc. Here's how:
* Set write-through: `/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp WT -L1 -a0`
* Set direct, no cache: `/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -Direct -L1 -a0`
* Turn readahead off: `/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp NORA -L1 -a0`
* Change the scheduler: `echo deadline > /sys/block/${disk_id}/queue/scheduler`

# Cleaning Up
If you mess up and need to delete an array, or just want to convert between different RAID levels, you can delete existing arrays following these steps:
1. List the existing virtual drives: `/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -a0`
2. Delete the virtual drive you don't want: `/opt/MegaRAID/MegaCli/MegaCli64 -CfgLdDel -L$VIRTUAL_DRIVE_ID -a0`
3. Set everything back to good: `sudo /opt/MegaRAID/MegaCli/MegaCli64 -PDMakeGood -PhysDrv[$EnclosureID:$SlotID] -Force -a0`

# Conclusion
Hopefully this helps if anyone is looking to run some MegaRAID commands. It took us a bit to figure it all out, but once we got some scripts together we can now format our servers with minimal issues.
