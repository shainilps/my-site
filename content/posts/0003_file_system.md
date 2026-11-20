+++
date = '2026-11-13T00:00:00+05:30'
draft = false
title = 'File System'
tags = ['linux', 'operating-system', 'file-system']
+++

If there is any typo or grammer mistake, please don't mind.
and this is the file system which I'm writing/wrote(depending on the time) [pelican](https://github.com/shainilps/pelican)


Recently I got intrested in the file systems. I always noticed that part of my OS but I never knew what exactly they did.
Like there are lot of types of file systems for example a common one you can find in fedora systems (which i use) is btrfs(better file systems)
I also had this xfs on a virtual machine of companies server(yes we deploy to baremetal like ancient people for the love of it).
I learnt one quirk of these file system. the virtual machine we had no disk partition for us to interact instead they provide us
logical volume. an abstraction over disk by linux. that make sense for a vm because its very handy. In
logical volume you can partion, grow volume size and also do a RAID like behaviour without much effort.
the logical volume makes an abstraction such that if you have mutlple disks it makes a pool of storage and you make
partition out of that pool instead of managing each partion of each disk. [read more about the logical volume here](https://linuxhandbook.com/lvm-guide/).
So the quirk of XFS file system was it can't shrink the size. means if you format a 1G volume with xfs and then you want to
shrink without loosing data then it wont let you. I realised that after giving surprisingly large
partition for home. but again mistake like that cost nothing becuase its production server (becuase storage is cheap, well not really).

So File System is one of the major component of an operating system which help you to store and do
file operation more efficiently. Well there are ton of file systems and We are really not gonna talk
about the types becuase too many to cover and I'm just reading how to design one on my own for
now(more specifically I want to make Distributed FS which is hmm rather concerning for a first try
at this lol).

Since I needed the information to build one I read a book. yes a book. suprising right ? because
when I want something more techincal or how should I put it more better I love to read books. No
hate to LLMs, their response feel empty when you are doing something like this (though I do use LLMs
to help me out with different topics).

So basically in file system there are couple of parts we need to cover to have one good sense of the system.
Lets start one by one.

**Physical Disks**: The harddisk/SSD are the actual device that stores data. Disks usually write
data in blocks. for example a this block size can be 512 Bytes depends on the device. from this point
we now know that computers dont write individiual bytes in storage devices they write in a block.

**Disk blocks**: Well I told in last paragraph that physical disk have a block abstaraction in hardware
level. We also have another block level abstraction over the devices in software layer too. becuase
physical disk blocks are not really visible to us and its internal to the driver that writes to the
device. so we make our own disk abstraction over the storage devices. a typical block size can be
512 Bytes to 4KiB. really depends on your file system.

**Super Block**: Super block is basically the first block of the storage device. why this exists is pretty simple
this stores some meta information such as format of the file system, block size, total blocks,
inode counts, free blocks, free inodes and etc.

**INode**: Inode number is your file's unique identifier on a Inode Table of the file system.
means each file and directory(we will know about this part later) have it's
own inode number. Inode Table is basically just a Data Structure that maps inode number the INode Data
which have the infomration such as kind (file/dir), permission, uid, guid, number of links, size in byts,
access time, modify time, create time, direct block pointers,
indirect pointer and etc.

**File**: Files are one of the greatest abstraction of unix systems probably. In unix systems everything 
is file. even your storage devices are files. isnt that amazing. so Files usually consist of inode number and
file is basically just the Inode data that gets stored. file can have any format of names. wide used practice of
files are filename.filetype. If you notice in the last pargraph I never mentioned that Inode stores file name. I will 
explain that in the directory section


**Directory**: A Directory or a Folder is a regular file. really it's just a file whose data is just
map of filename and inode number. example: (example.txt 102) (main.c 33). From this you now know that
file names just belong to directory.

**link**: In linux if you want to delete a file you need to do unlink syscall. why do you think its
called unlink ? the reason is pretty simple each inode have an link from a directory. means a
filename -> inode_number creates a link. when the link count goes to 0 the file is deleted or inode
is deallocated. Linux system have two concepts hard link and symbolic link. hard links means each
file name have same inode. so if you have file inode number 102 and you make a name file.txt
file2.txt both dont occupy new space at all. becuase both just points to same inode and they are
basically same file. as for the symbolic link its a pointer to the file path. means its a file that
stores the information like file2 located in /home/soemwhere/somewhere. anayways you can read about
it more and I wont go more in detail about the symlink and hardlink here.


# How to make a File System ?

So After learning the above concept I wondered how to make my own file system. I got to know 
that there is two types of file systems. one who run in kernel space and one runs on user space. 
FUSE if you ever heard It lets you run a file system in userspace. Inorder to understand we need to 
understand how usually file systems are invoked.

usually when you do write on a file, a write syscall gets invoked. it then forward the calls to 
VFS (virtual file system). Think of it as a interface for the all file system by kernel. So the 
file system such as xfs, btrfs and etc register thier name here. usually in linux you mount the 
file system to a directory. this is very interesting. becasue you can mount a / to a 20G parition of 
disk lets call it A. and then you can mount /home to another parition B which is 100G.  and both parition can 
be different file system. Now when you make a write to a file the path gets resovled so thats way you know
which directory your file lies and then you invoke the registered file system driver to do the operation such as
write 5 bytes to a file called /home/example.txt. 

Fuse also does the same thing but the kernel leve file system usually registered at kernel leve and 
its pretty vulerable, If your file system crash you may crash entire kernel. So user space file 
system like FUSE lets you do file operation in user space. but how ? Fuse itself isnt a file 
system its a interface which registers itself to the VFS. so any file operation related to the FUSE
mounted file system delegated to the FUSE and then FUSE lets makes the call of calling your program to
do the operation. usually FUSE wont really call, we implemnt the function for FUSE to call.

so a quick guide to how to make a FUSE file sytem is [Write a filesystem with FUSE](https://engineering.facile.it/blog/eng/write-filesystem-fuse/)

Now you know how the file system actually works. Now Let's Design and See How the different component of
File System fit together in Disks.

continuing....
