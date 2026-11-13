+++
date = '2026-11-11T00:00:00+05:30'
draft = false
title = 'File System'
tags = ['linux', 'operating-system', 'file-system']
+++

Please excuse my typo and grammer becuase I don't want to use LLM for this.

Recently I got intrested in the file system. I always noticed that part of my os like btrfs which I
have in my fedora machine. I also had this xfs probably on a server. I learnt one quirk of these
file system in that. the virtual machine we had no physical disk instead they provided logical
volume an abstraction over disk by linux. that make sense for a vm because its very handy. In
logical volume you can partion, grow disk size and mount them back very easily. you can expand your
storage by adding new hardware without any problem.
[read more about the logical volume here](https://linuxhandbook.com/lvm-guide/). So in the XFS file
system quirk was it cant shrink the size. means if you format a 1gig volume and then you want to
shrink without loosing data then it wont let you. I realised that after giving surprisingly large
partition for home. but again mistake like that cost nothing becuase its production server (just
kidding or am i not).

So File System is one of the major comonent of an operating system which help you to store and do
file operation more efficiently. Well there are ton of file systems and We are really not gonna talk
about the types becuase too many to cover and I'm just reading how to design one on my own for
now(more specifically I want to make Distributed FS which is hmm rather concerning for a first try
at this lol).

Since I needed the information to build one I read a book. yes a book. suprising right ? because
when I want something more techincal or how should I put it more better I love to read books. No
hate to LLMs, their response feel empty when you are doing something like this (though I do use LLMs
to help me out with complex topics).

So basically in file system there are lot of things. Lets start from Disk.

**Secondary Storage**: The harddisk/SSD are the main component in this whole thing becuase they are
the one keeping the data. I haven't really looked into how exactly they store data but I will updaet
this blog once I get that part. becuase writing a shallow explanation wont give anything.

**Disk blocks**: So disk have an abstraction before we even make use of it. that is we split a disk
into bunch of chunks. like 4KiB. why do we do that? well if you split the disk byte by byte there is
one problem that is each byte needs its somewhat state tracker data structure if you split the disk
too small more data needed to store the information of each byte is allocated or not. that's why a
number like 4KiB is found efficient.

**INode**: Inode number is what your file real name you could say, a unique identifier on a Inode
table of the file system. means each file and directory(we will know about this part later) have its
own inode number. Inode Table is basically just a datastructure that maps inode number the Inode
Data structure which have the infomration such as file permission, disk block pointers and many more

**File**: Now a let us talk about most obivous form we store data that is files. you can have
different types of file a c source code, image file, video file. File System treat them same well in
general. as you already know form last paragraph the file have a Inode number which says how the
data of the file is stored in storage devices or you could say file system.

**Directory**: A Directory or a Folder is just an file. really its just an file whose data is just
map of filename and inode number. example: (example.txt 102) (main.c 33) If you observer I told that
Inode data structure store the file information but it never really stores the file name at all.
because file names just belong to directory.

**link**: In linux if you want to delete a file you need to do unlink syscall. why do you think its
called unlink ? the reason is pretty simple each inode have an link from a directory. means a
filename -> inode_number creates a link. when the link count goes to 0 the file is deleted or inode
is deallocated. Linux system have two concepts hard link and symbolic link. hard links means each
file name have same inode. so if you have file inode number 102 and you make a name file.txt
file2.txt both dont occupy new space at all. becuase both just points to same inode and they are
basically same file. as for the symbolic link its a pointer to the file path. means its a file that
stores the information like file2 located in /home/soemwhere/somewhere. anayways you can read aobut
it more.

Well, that is my initial understanding of the file system and its components. This is not the final
version of this article. I want to go deeper into how things work under the hood. For now I'm
posting this because I haven't written a blog since 2025 lol.
