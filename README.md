SD card autocopy
================

Copying photos from the SD card of my camera is a very common task. And always the
same pattern. Take the card out of the camera, put it into the sd card reader, wait
for the automount popup, browse to the picture directory, select all images, cut
all images, browse to my photo directory, create a new folder, name the folder,
move to the folder, paste the pictures into it, wait for the finished job.

Sound boring? Sure, it is!

Now, how to make this one a little smarter? Maybe scripting some of those steps is
a cool idea. So how would this work in a perfect world?

The idea
--------

Taking the SD card out of the camera and putting it into the SD card reader does not
seem to be easily scripted, that's why I focused on the other steps ;-). Okay, I'd
like to put the card into the reader, just wait for all the images to be moved and
then put the card back into the camera.

But how to reach this goal? Maybe you're interested in my solution...

One thing we need to know: When hotplugging devices to my Ubuntu system the kernel
tells the udev daemon about that new device. The kernel informs the udev daemon
about a lot of different events, like adding and removing hardware, volumes and so
on. But how to get closer to these events?

Digging into udev events
------------------------

There is an interesting tool named `udevadm` which can be used to monitor
the incoming events. Let's see which events are fired when I put the SD card into
the card reader:

    > udevadm monitor
    monitor will print the received events for:
    UDEV - the event which udev sends out after rule processing
    KERNEL - the kernel uevent

    KERNEL[1316365297.577602] add /devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368 (mmc)
    KERNEL[1316365297.578091] add /devices/virtual/bdi/179:0 (bdi)
    UDEV [1316365297.579008] add /devices/virtual/bdi/179:0 (bdi)
    UDEV [1316365297.579058] add /devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368 (mmc)
    KERNEL[1316365297.580552] add /devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368/block/mmcblk0 (block)
    KERNEL[1316365297.580644] add /devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368/block/mmcblk0/mmcblk0p1 (block)
    UDEV [1316365297.617556] add /devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368/block/mmcblk0 (block)
    UDEV [1316365298.314850] add /devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368/block/mmcblk0/mmcblk0p1 (block)

These are a lot of of events, but there is only one needed, so it is needed to
choose the correct one. One way would be to try one event after each other, but
there is a better way: Now one need to know that it is possible to react on
specific events and run some script every time such an event is received. The
script is executed and has access to the whole environment which is populated
with some useful information.

To get a little more information create a matching udev rule and a small script
which appends the whole environment to a temporary dump file:

Store the udev rule in the rules directory `/etc/udev/rules.d/99-sd-autocopy.rules`:

    SUBSYSTEM=="block", ACTION=="add", RUN="/usr/local/bin/sd-autocopy"

The udev daemon should recognize that rule immediately. If a restart of the
udev daemon might help.

Now create the script to dump the environment at the path specified in the
udev rule `/usr/local/bin/sd-autocopy`:

    #!/bin/bash
    echo ======================== >> /tmp/test
    env >> /tmp/test

Then make the script executable:

    chmod +x /usr/local/bin/sd-autocopy

Now simply plug the SD card of your choice into your SD card reader and have a
look at the contents of `/tmp/test` afterwards. The file should contain
information of several events where the single events are separated by the
lines of equal sings.

Having these events recorded it is now possible to grab the important
information for the script. When looking at the single events we see that there
is one `ADD` event for the added disk and one `ADD` event for each partition.
We are only interested in the partition events and need the first partition of
the device, so add the following conditions to the udev rule:

    ENV{DEVTYPE}=="partition", ENV{UDISKS_PARTITION_NUMBER}=="1"

which should result in this rule:

    SUBSYSTEM=="block", ACTION=="add", ENV{DEVTYPE}=="partition", ENV{UDISKS_PARTITION_NUMBER}=="1", RUN="/usr/local/bin/sd-autocopy"

After changing the rule, deleting the `/tmp/test` file and an additional test
we can see that the test sd-autocopy script gets only called with the correct
event. This event comes with the following environment variables:

    DEVTYPE=partition
    SUBSYSTEM=block
    ID_SERIAL=0x000006d4
    ID_FS_UUID=F84E-1690
    UDISKS_PARTITION_OFFSET=4194304
    DEVPATH=/devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368/block/mmcblk0/mmcblk0p1
    UDISKS_PARTITION_NUMBER=1
    ID_FS_VERSION=FAT32
    MINOR=1
    ACTION=add
    PWD=/
    UDISKS_PARTITION_ALIGNMENT_OFFSET=0
    ID_FS_TYPE=vfat
    ID_NAME=SDC
    MAJOR=179
    DEVLINKS=/dev/disk/by-id/mmc-SDC_0x000006d4-part1 /dev/disk/by-path/pci-0000:15:00.2-part1 /dev/disk/by-uuid/F84E-1690
    DEVNAME=/dev/mmcblk0p1
    SHLVL=1
    ID_FS_USAGE=filesystem
    ID_PART_TABLE_TYPE=dos
    UDISKS_PRESENTATION_NOPOLICY=0
    UDISKS_PARTITION_SIZE=16130244608
    ID_FS_UUID_ENC=F84E-1690
    UDISKS_PARTITION_SCHEME=mbr
    UDISKS_PARTITION_TYPE=0x0c
    UDISKS_PARTITION=1
    UDISKS_PARTITION_SLAVE=/sys/devices/pci0000:00/0000:00:1e.0/0000:15:00.2/mmc_host/mmc0/mmc0:b368/block/mmcblk0
    ID_PATH=pci-0000:15:00.2
    SEQNUM=2825
    _=/usr/bin/env

The sd-autocopy script
----------------------

With these information the SD card can be identified. It will use the
`ID_SERIAL` and `DEVNAME` environment variables in the script. Now I identified
the tasks which need to be performed by the sd-autocopy script:


1. Verify the sd card to be a handled one
2. Mount the partition
3. Find all images on the partition
4. Create a folder for each date in a target base path
5. Move the files to the date specific folder
6. Unmount the partition

The script is placed in `/usr/local/bin/sd-autocopy` (just remove the testing
code from that file). The finished script can be found in the git next to this
README file.

The script can easily be extended by modifying the first few lines:

    USER=lm
    TARGET=/home/$USER/Pictures/Photos
    HANDLED_SERIALS="0x000006d4"
    MOUNT=/mnt/sd-autocopy
    FILE_MATCH=".*\.(jpg|cr2)$"
    # Move or copy - or something different
    TRANSFER=mv

* `USER` is needed a) for gathering the target directory and b) for owning the copied files.
* `TARGET` holds the path to the target base directory.
* `HANDLED_SERIALS` holds a list of space separated serial numbers of the SD cards to use the script for.
* `MOUNT` contains the mountpoint for the first partition on the SD card.
* `FILE_MATCH` is the matching regex for the files to be copied. It applies to the whole path of the files.</li>
* `TRANSFER` defines the transfer mode. For example "mv" to move the files or "cp" to copy them.

The script sends it's log entries to syslog, so you might want to have a look
at the syslog files for debugging.

Btw. the script is not limited to handling images. By just modifying the
`FILE_MATCH` pattern it is possible to copy other filetypes.
