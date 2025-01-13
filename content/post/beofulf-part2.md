---
title: "Building a Beowulf Cluster from old MacBooks: Part 2"
<!-- author: Jon Deaton -->
date: '2017-10-08'
---

_Originally published on [wordpress](https://jondeaton.wordpress.com/2017/10/08/building-a-beowulf-cluster-from-old-macbooks-part-2/)_

<img 
  src="/images/cluster_blog/cluster.webp" 
  alt="Assembled Cluster"
  style="width: 300px; float: right; margin-left: 10px; margin-bottom: 10px;" 
/>

When we left off at part 1, we had ArchLinux installed on each of the computers in the
cluster, and each is also running an SSH server so that is can be accessed remotely. The
next step is to set up a shared file system that each node has access to. This will be
very helpful later when coordinating distributed tasks, if for instance we are
processing a bunch of files and we need each node to have access to any of the files.

There are several approaches to setting up a shared file system including SSHFS (SSH
File System), NFS (Network File System), AFS (Andrew File System). I had some previous
experience with SSHFS, a network file system build on SSH, but because SSHFS is
encrypted it is less performant than the other two options. I also contemplated using
AFS for the shared file system, (like the clusters are my school use), however, I wanted
to use my 2TB Western Digital MyCloud to host the shared file system. Because the
MyCloud already had NFS up and running, and it had little support for installing new
software, I went with NFS for the shared file system on the cluster.

## Configuring NFS

There is a great blog post by Benjamin Cane that goes over the basics of setting up NFS,
so instead of reiterating, I’ll cover the extra steps that and troubles that I ran into.
One of the trickiest parts of setting up the NFS was getting each node to have the
proper permissions and ownership over the files in the NFS. The first problem that I ran
into was permissions issues and not being able to access the files on the NFS even after
mounting it on each node.

The core of this issue came down to two problems, the first of which is user/group ID
mapping. In Linux, each user id and group id are associated with a string. For instance
the user id 0 is associated with the string root. Regular users typically are assigned
user IDs greater than 1000.

The problem comes in that a user id number of say user1 on the client may not have the
same user id on the server hosting the NFS. As pointed out in a server fault thread
there are two ways of resolving mismatched user ids for the same user on different
systems. To summarize, you can either specify a user id mapping in the NFS
configurations file (i.e. /etc/exports) or (more hacky) change the user id to be the
same number on all the systems. I opted for the former.

The second problem, had to do with root permissions. For security reasons, it is
genereally advised (and set by default on the WD MyCloud) for files owned by root on the
NFS server to “squashed” so that even root on the NFS client cannot access them. For me,
this manifested itself as not being able to modify files owned by root on the NFS
partition on any of the nodes. Fixing this was as simple adding a no_root_squash option
to the exported file systems in /etc/exports on the server.

An interesting quirk that I learned about the MyCloud is that it has a special partition
set up for storing data, and a separate partition for the Linux system that it runs.
After originally mounting the system on the (considerably smaller, 2GB) system
partition, I quickly ran out of space. I was at first astounded at this because I
thought that the MyCloud had 2TB of storage. After realizing that I had exported the
wrong part of the MyCloud file system, the issue was quickly fixed by re-directing the
NFS export to be some sub-directory of /DataVolume/ which is where the 1.8TB of free
storage is.

## Changing home directory

After mounting NFS on each node of the cluster, this was a really good time to change
the home directory of each node to be on the NFS. This way, any configurations that
reside in the home directory (e.g. ~/.bashrc, ~/.ssh etc.) were shared across all nodes
in the cluster, thus only needed to be done once. As I learned in a stack exchange
thread there are two ways to change the designated home directory of a user. If you
aren’t logged in as the user that you want to change the home directory for, then this
is easy: simply run

```bash
usermod -d /new/home userName
```

However, if you are logged in as the user, you’ll need to edit /etc/passwd and change
the home directory for the designated user manually, and then log out and in again.
After having changed the home directory of each node to be on the NFS mount,
configuration became a lot easier.

## Password-less SSH

Once all of the node have the same shared home directory, it was a great time to set up
password-less SSH login. This is because all of the nodes now shared the same list of
authorized keys stored in ~/.ssh/authorized_keys. If you’ve set up key-authenticated SSH
before you’ll know that its super easy, but I included it in this post because I
encountered some unique problems that you might not have seen before.

First, not only is it convenient to login to the cluster with keyed authentication, but
for distributed tasks all of the nodes need to be able to communicate with each other
over SSH without password prompts. This is because many of the message passing
interfaces, and cluster monitoring tools utilize SSH for communication.

One interesting problem I encountered involved a potential security issue. In order to
login to every node in the cluster from my remote laptop, I generated an RSA key pair,
and added the public key to ~/.ssh/authorized_keys and added the path to the private key
to ~/.ssh/config under IdentityFile. At this, point, in order to make each node log into
each other node, I was tempted to copy the private key to ~/.ssh/sk.rsa and store that
in the ssh config file on the cluster. I realized that this would have been a security
issue however, because the home directory of each node was actually a mounted NFS. Since
NFS communication is unencrypted, this means that the private key would have been sent
over the network in the clear every time a node read this file in order to authenticate
itself to another nodes. That would have been bad, but it has an easy fix which is
simply placing the key into (the same) path on the local storage of each node and
pointing the (shared) ~/.ssh/config to that path.

When setting up password-less SSH on the WD MyCloud I ran into another interesting issue
that took a long time to diagnose. It turns out that if you store the authorized key
file in ~/.ssh/authorized_keys, the home directory ~ needs to have permissions set to
700 as well (or less permissive than 777, anyway) as pointed out in an extremely helpful
stack exchange post on debugging this problem.

## Synchronizing time across the cluster

Another interesting issue that I ran into was discrepancies between each node’s notion
of current time. Some nodes differed by over 30 seconds from others. This manifested
itself when running commands that manage files on the shared NFS volume, such as the
following warnings that I got while running make

```
make: Warning: File `main.c' has modification time 21s in the future make: Warning:
Clock skew detected. Your build may be incomplete.
```

To fix this, I needed to synchronize the dates with a program called NTP (Network Time
Protocol). This system feature works by running a daemon that periodically synchronizes
the system’s current time with that of an external trusted server. To synchronize clocks
across the cluster, one way is to have a head node (in our case the WD MyCloud) fetch
time from the outside periodically and then have the other nodes synchronize with the
head node.

To do this, on the WDMyCloud I configured `/etc/ntp.conf` to be

```
server time.apple.com
```

so that the system hosting the NFS would fetch time from Apple. I did this because I
wanted to mount the NFS on my personal MacBook and I didn’t want there to be any
difference between my laptop’s time and the NFS time. To make this take effect, I
restarted the ntpdate daemon with:

```
sudo service ntpdate restart
```

Then on the Arch Linux nodes, I configured /etc/ntp.conf to be the same (i.e. server
time.apple.com) and restarted the daemon on them as well with

```
sudo systemctl restart ntpdate
```

Interestingly, even once I did this, the clock continued to be skewed when I would log
days later. The only thing I could think of was quite the hack: I had the ntpdate daemon
restart itself every two minutes by adding a cron job that performed this task. I don’t
think this is the proper fix to the problem (isn’t the ntpdate daemon supposed to fetch
time periodically by itself?!) but it was easy and fixed the problem permanently. To do
this I ran crontab -e and then inserted the line

```
*/2 * * * * * sudo systemctl restart ntpdate
```

into the opened file.

## NFS I/O optimization

One of the first things that I noticed about the NFS partition was that I/O was
unbearably slow. I had a 1Gbit (125 MB/s) ethernet switch and each node (raspberry pi’s
excluded) had fast enough network interface to handle network at this speed. Despite
this, I was noticing very slow write speeds to the NFS. It was only when it took over 10
minutes to write a 100MB to the NFS that I decided it was unacceptable. Tuning the
parameters on the NFS to optimize I/O was one of the more interesting steps in setting
up the system.

I followed the detailed discussion here for how one diagnoses and optimizes NFS. The
first thing to do is quantitatively assess current performance so that future
configurations can be bench-marked for effectiveness. The command dd, which transfers
raw data from source to sink, is a great tool for this. Running

```
dd if=/dev/zero of=/mnt/nfs/testfile bs=16k count=16384
```

will simply copy null bytes from `/dev/zero` into a test file on the NFS, and then report
transfer speed. When I ran this command on the un-configured NFS, I saw 22.5 MB/s of
write speed. Repeating but instead reading from the test file (into `/dev/null`), I saw
45.1 MB/s write speed.

Just to benchmark against the disk speed of the head node, I ran the test again directed
at a local directory and saw 563 MB/s write and 2.3 GB/s read speeds, confirming my
suspicion that indeed it was the NFS was performing poorly. Just to be sure that the
bottle neck wasn’t the WD MyCloud disk speed, I checked the I/O on the WD MyCloud and
found 124 MB/s write, and 98.8 MB/s read: Nope!

The first recommendation for tuning NFS is to edit block size and other mount options in
`/etc/fstab`. Playing around with rsize/wsize on the head node yielded the following

```
rsize/wsize = 4k 13.7 MB/s write, 17.7 MB/s read
rsize/wsize = 8k 20.6 MB/s write, 31.9 MB/s read
rsize/wsize = 16k 27.6 MB/s write, 50.8 MB/s read
rsize/wsize = 32k 36.1 MB/s write, 70.2 MB/s read
rsize/wsize = 64k 26.9 MB/s write, 67.8 MB/s read
rsize/wsize = 524k 35.1 MB/s write, 71.6 MB/s read
```

Note that if you don’t unmount the NFS partition, the file will be buffered in memory
and dd will read from memory instead of the network, purporting >2GB/s transfer. These
results were promising but they only increased speed a little bit, and were still well
below the theoretical limit of 125 MB/s.

Investigating the tips and tricks for using NFS on Arch Linux documentation, I found
that the max_block_size of NFS might indeed be the issue given that
/proc/fs/nfsd/max_block_size held 32767 on the MyCloud. This explained why transfer
speed plateaued after block sizes of 32k. To change this parameter I simply edited the
file, however, since the NFS daemon locks this file I first stopped the NFS daemon on
the MyCloud with service nfs-kernel-server and service nfs-common stop.

Then I edited with echo 524288 > /proc/fs/nfsd/max_block_size and restarted the NFS
daemons. This resulted in a write speed of 48.5 MB/s (better!) and read speed of 90.6
kB/s. Yeah, seriously that’s in kB/s. the system actually froze for a solid 10 minutes.

It was here that I realized that the MyCloud had only has 256Mb of RAM. My first thought
was: “the 524 MB of buffer used in each block is eating up all the RAM on the MyCloud
causing it to thrash?”. I ran the I/O benchmark command again and checked top on the
MyCloud. Surprisingly, no process was using any significant memory. What then? My system
has 4Gb of memory so its hard to believe that it was using up the memory.

I found an interesting article about network socket buffer sizes and decreased rsize
down to 131072 and got 101 MB/s read speed. Okay, its not quite a Gigabit, but its quite
an improvement and I can’t think of anything else so I’ll live with it.

## Conclusion

Finally I had each node in the cluster was set up with a shared file system and
authenticated communication. At this point I was ready to start writing distributed
compute tasks. In my next blog post I will talk about how I used C++ and OpenMPI to
coordinate distributed tasks across the cluster!
