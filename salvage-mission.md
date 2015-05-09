% Salvage Mission

My partner's sister comes to me a few days ago with a Toshiba external
hard drive that has a broken connector. Critical files on it, recovery
required. Okay, fair enough; I used to do this all the time in my days as a
PC tech. I still have a wonderful [usb to hard drive adapter][adapter] that
I used to use to salvage files in cases just like this. Should be trivial;
it's just physical damage to the drive shell's connector, nothing wrong with
the drive itself. I hook it up to the nearest machine.

Explorer asks if I want to format it. What? No. No, I don't.

I look at it in Disk Manager. I see a single big partition marked "Raw." This
was around the time I figure out I'm going to have a bad day. This is the
story of that bad day.

I disconnect the disk and attach it to my Linux laptop to have a closer
look. Trying to mount it fails utterly:

```console
$ mkdir /mnt/test
$ mount -t ntfs /dev/sdc1 /mnt/test
NTFS signature is missing.
Failed to mount '/dev/sdb2': Invalid argument
The device '/dev/sdb2' doesn't seem to have a valid NTFS.
Maybe the wrong device is used? Or the whole disk instead of a
partition (e.g. /dev/sda, not /dev/sda1)? Or the other way around?
```

I have no idea what an NTFS signature is, but I know that this is bad. I try
a few other methods of mounting it without success. I point ntfsfix at it,
also without success:

```console
$ ntfsfix -n /dev/sdc1
Mounting volume... NTFS signature is missing.
FAILED
Attempting to correct errors... NTFS signature is missing.
FAILED
Failed to startup volume: Invalid argument
NTFS signature is missing.
Trying the alternate boot sector
Unrecoverable error
Volume is corrupt. You should run chkdsk.
No change made
```

Note the -n option, which asks ntfsfix to tell you what it will do without
actually doing it. I am kind of nervous at this point, because I have no backup
of this drive and I don't want to break things more than they're already
broken. I consider pointing testdisk at it (which I haven't used before)
but can't figure out how to get it to show me a dry run. I could *probably*
hand-hack a partition table if I accidentally nuked it, but 1. I'd rather not,
and 2. I'm reasonably sure I *can't* hand-hack the NTFS filesystem itself,
and I've no idea if testdisk touches that.

I can't make a backup of the whole disk, it's too big, but before I start
getting invasive let's at least back up the beginning of the drive, so I
have a copy of the partition table and hopefully whatever front-of-filesystem
information NTFS uses:

```console
$ dd if=/dev/sdc of=/root/sdc.bak bs=1k count=10000
```

That gets me a backup of the first 10MB of the drive. It makes me feel a little
better, but not much. I don't know for sure that ntfs keeps its bookkeeping
at the front of the drive. So I'm comfortable breaking the partition table
but no more than that. I'm still afraid to run automated tools at it, like
chkdisk or testdisk. I do try ntfsfix again, without success.

Still, it's something. Let's see if we can work out what's up with that
signature. I'm guessing the ntfs mounter expects some magic number at the
beginning of the partition's filesystem to indicate that it actually is NTFS.

```console
$ hd /dev/sdc1 | head
00000000  52 43 52 44 28 00 09 00  ea df 0d 43 00 00 00 00  |RCRD(......C....|
00000010  01 00 00 00 01 00 01 00  50 0f 00 00 00 00 00 00  |........P.......|
00000020  de df 0d 43 00 00 00 00  83 4a 00 00 00 00 00 00  |...C.....J......|
...
```

If you're not familiar with `hd`, the first column here is an offset, the
main body is the hexadecimal representation of the data at that offset,
and the right column is the ascii translation of the same.

Well, RCRD looks sort of like it might be a signature of something. The Oracle
at Google tells me that it's something to do with NTFS's log or journal. So
there's something like an NTFS filesystem here, but sdc1 seems to be starting
in the middle of it. Where? Let's see if I can find out where on the disk
it's actually looking for sdc1. I don't really trust the partition table,
so I'm going to look directly at the disk.

```console
$ hd /dev/sdc | grep RCRD | head
00113000  52 43 52 44 28 00 09 00  00 60 00 00 00 00 00 00  |RCRD(....`......|
00114000  52 43 52 44 28 00 09 00  00 70 00 00 00 00 00 00  |RCRD(....p......|
00115000  52 43 52 44 28 00 09 00  d4 09 00 4a 00 00 00 00  |RCRD(......J....|
...
```

Oops. Apparently I'm really in the middle of a series of RCRD entries, which
I probably should have expected. I need to search for the whole line. I only
search the first 10MB here because I don't want to be reading the whole disk:

```console
$ hd /dev/sdc -n 10000000 | grep '52 43 52 44 28 00 09 00  ea df 0d 43 00 00 00 00'
00800000  52 43 52 44 28 00 09 00  ea df 0d 43 00 00 00 00  |RCRD(......C....|
```

Okay, I found /dev/sdc1 at the 8MB mark. Let's compare that to the partition
table to see if something's really screwey. parted refuses to look at the
disk, though. Apparently my partition extends off the end of the disk,
and parted's map claims that here there be monsters:

```console
$ parted /dev/sdc
GNU Parted 2.3
Using /dev/sdc
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Error: Can't have a partition outside the disk!
```

cfdisk is slightly more cooperative, and reports that partition sdc1 starts
at sector 2048 (I can't show it here because it's a curses application). With
4k sectors that translates to 0x800000, i.e. where we found that RCRD entry
above. Progress! We appear to have a working partition table pointing to
the wrong place. Maybe that's why it seems to go off the end of the disk? I
don't know, but I actually find that encouraging; if the partition table is
broken, the filesystem may not be. I'm more confident in my ability to fix
the former than the latter. The actual data on the disk is *probably* intact.

On the other hand, I can't think of any causal reason a busted external-shell
connector (remember that, from the beginning?) would muck up actual bytes
on the disk, and such specific ones.

Whatever, presumably it will make sense later. I know I can nuke the partition
table and make a new one pointing wherever I want with parted. But where is
it *supposed* to point? "Wherever the missing NTFS signature is actually
located", presumably, but I still don't know what that is, and the Oracle
remains silent on the matter.

I think of something I should have thought of hours earlier. I ask Google for
a Windows utility capable of pulling a hex dump of disks and partitions; it
points me to Hex Workshop. I fire that up on one of my own PCs, and open the
system drive, hoping for something that looks like a signature. It gives me...

```console
00000000  EB 52 90 4E 54 46 53 20 20 20 20 00 02 08 00 00  .R.NTFS    .....
00000010  00 00 00 00 00 F8 00 00 3F 00 FF 00 3F 00 00 00  ........?...?...
00000020  00 00 00 00 80 00 80 00 73 3C 0D 03 00 00 00 00  ........s<......
```

...the bloody obvious. Three bytes of I-don't-know followed by "NTFS" in
ASCII. ARRRRRRGGGGGGGHHHHHHH.

Back to the problem drive:

```console
$ hd sdc.bak -n 10000000| grep NTFS | head
00100000  eb 52 90 4e 54 46 53 20  20 20 20 00 02 08 00 00  |.R.NTFS    .....|
```

Paydirt. So our partition actually starts at 0x100000, not 0x800000. The
partition table specifies sectors, not bytes. Sector 2048 on a 4k-sector
drive is 0x800000, so that's where we end up when we look at the start of
/dev/sdc1. 0x100000, where the actual NTFS signature is, corresponds to
sector 2048 on a 512b-sector drive.

But we don't have a 512b-sector drive! We have a 4k! I'm looking at a correct
partition table for a different address scheme than that used by the drive
it is actually on! WTF??!

I can't for the life of me remember what mess of google searches turned up the
root cause, but it eventually led me to [this askubuntu answer][answer]. I'll
quote the relevant bit:

> Some enclosures translate 512-byte logical sectors on the disk into
> 4096-byte logical sectors presented to the computer -- that is, the
> opposite of what the firmware in an Advanced Format disk does. I'm
> not positive, but I suspect that some enclosures do this only on
> over-2TiB disks. Both MBR and GPT partitioning schemes refer to data
> by sector numbers, so changing the sector size invalidates the
> partitioning data. Thus, if you prepare the disk in a USB enclosure
> that translates in this way and then try to use the disk directly
> (or vice-versa), you'll see errors because the partitions (and even
> GPT backup data) won't be where the computer expects it to be.

Toshiba, why you make me want to hit you?

Five minutes and a few commands fixes the problem:
I delete the partition in cfdisk, then re-create it using parted, because
parted will let you dictate where it starts. On a 4k disk, 0x100000 is the
start of sector 256.

```console
$ parted /dev/sdc
GNU Parted 2.3
Using /dev/sdc
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mkpart primary ntfs 256s
End? 100%
(parted) quit

$ mount /dev/sdc1 /mnt/sdc1
$ ls /mnt/sdc1
file1 file2 file3 file4 file5....
```

Woot!

I copied everything important off the disk, just in case, and then gave it
back to its owner.

**Afterword**

This quote is also from the answer above, and it makes me think.

> The solution to this problem is to prepare and use the disk in one
> way -- either use the USB enclosure or use a direct connection, not
> both. If both are necessary for some reason, you'll need to find an
> enclosure that works without applying this type of translation.

Do we really? Just how valid does the partition table need to be, to not
make things explode?

Suppose you created two partitions, overlapping, one valid for 4k sector
sizes and one for 512b, both addressing the same space. I don't think
partition-management tools will let you do this, but suppose you hand-hacked
it. Could you then swap the drive in and out of an enclosure performing this
sort of translation, and expect one volume to work each way, both accessing
the same data?

I don't know, and since the original translating enclosure is busted I
don't have a way to try it, and I can't think of any reason you would *want*
to do it, but it would be awesome if it worked.

**Afterword 2**

When the replacement enclosure (different make and model) arrived, it
turned out to do the same demented translation as the old one. Fortunately,
it wasn't hard to revert the changes I'd made. But seriously, how common
is this crap? Who should I hurt for it?

[adapter]: http://www.newegg.com/Product/Product.aspx?Item=N82E16812196455
[answer]: http://askubuntu.com/a/337993
