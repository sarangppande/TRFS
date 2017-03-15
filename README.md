# README #

We have implemented a traceable file system called "trfs" using wrapfs template.
Trfs can trace system calls and log them into a file which can be later used for
replaying.

TRFS's design can be divided into following major parts:

### A) System Call Intercepting and Logging: ###

For major interception of system calls we have used the methods provided by
wrapfs. We have logged the system call activities after trfs has completed 
its action on the lower file system. 

To avoid waiting for the trace to be written to the file, we have implemented a 
single thread workqueue. So after the system call returns from the lower 
filesystem which could be either success or failure, we record the state and
add that work to workqueue. The workqueue then processes the record state and
writes the trace to file asynchronously without making the whole system to halt.

Majority of the system calls that are traced are either file operation or 
dentry operation. We have two different methods for recording state and adding
the work to the workqueue. This method are called by the trfs_* methods.

The system calls that we are currently tracing are:


1. write
2. read
3. open
4. create
5. link
6. unlink
7. symlink
8. mkdir
9. rmdir
10. llseek

Major memory releasing and clean-up is done by thread.

### B) Mounting and Unmounting ###

When a directory is mounted on TRFS, the trfs_mount function in main.c is run.
The function parses the options provided while mounting ie. tfile path , checks
if the filepath is valid else returns an error. The function also initializes
workqueue and then calls the trfs_read_super method which is used for initial
setup of trfs super block. In this method we store the tfile file descriptor
into the private field of superblock along with record buffer,
operation tracing flag (default. all sys calls), and atomic record id counter.

When unmounting a filesystem, trfs_put_super method is run in which we are 
closing the file, freeing the buffer memory and destroying the workqueue.

### C) IOCTL code. ###

TRFS uses IOCTL to set tracing a system call on or off. 
The user level program to do this is "trctl". To run the program use following
command

`.trctl cmd /path/to/mount/point `

where cmd can be "all", "none" or "0xNN" ie. any hex value within range.

You can also use the same program to retrieve the current operation tracking 
flag.

`.trctl /path/to/mount/point`

### D) Replaying trace file ###

To replay trace file , we have created a user level program called "treplay.c".
For running the program use following command:

`./treplay [-ns] /path/to/trace/file `

where -n : Only show records, don't replay them
       s : Replay the trace file in strict mode. If deviations are found, abort.
 default : If no option is given, display the records and replay them.

      
### E) INSTALLATION ###

We have also created a shell file for loading new module named
"instal_module.sh". The script unloads the old module and loads the updated
module.

### F) EXTRA CREDIT ###

We have implemented Incremental IOCTL change. To activate this feature, please
uncomment the line "#define EXTRA_CREDIT" from trctl.c and then compile it 
using gcc. 

The feature allows user to trace or untrace system calls using +/- signs.
The user can provide either of the following options to trace or untrace
system calls:

`+read, -read, +write, -write, +open, -open, +create, -create, +link, -link,
+unlink, -unlink, +symlink, -symlink, +mkdir,-mkdir, +rmdir, -rmdir, +lseek
-lseek, all, none`

When a request is issued , the program fetches the current operation map using
ioctl and then perform operation on that bitmap and then stores it back to
the super block.

### H) EXTRA CREDIT/POSSIBLE BUG: ###

While trying to trace llseek we realized that in file.c, in 
file_operations struct trfs_main_fops, the mapping for llseek was set to 
generic_file_seek instead of trfs_file_seek, so we changed it and we were able
to track llseek.

### I) RUNNING INSTRUCTION:###

1) Make sure trfs source files are present at the  location fs/trfs.
2) Run make command using kernel.config that we have provided, it has module
   option set for TRFS.
3) Run install_module.sh to insert the module.
4) Mount the filesystem on TRFS.
5) Run system calls in mounted filesystem.
6) Unmount filesystem
7) Run treplay to replay actions


### New files: ###


1. hw2/README.HW2
2. hw2/kernel.config
3. hw2/fs ---> Contains the copy of files listed below
4. fs/trfs/dentry.c
5. fs/trfs/file.c
6. fs/trfs/inode.c
7. fs/trfs/lookup.c
8. fs/trfs/Kconfig  --> For help section of make menuconfig
9. fs/trfs/install_module.sh --> For loading module
10. fs/trfs/main.c
11. fs/trfs/Makefile
12. fs/trfs/mmap.c
13. fs/trfs/super.c
14. fs/trfs/treplay.c --> For replaying
15. fs/trfs/trctl.c --> For setting/unsetting points
16. fs/trfs/trfs.h
17. fs/trfs/trfs-workqueue.c --> Workqueue code 
18. fs/trfs/trctl.h


### REFERENCES: ###

1. Linux Kernel Code: http://lxr.free-electrons.com/source/
2. https://www.cs.rutgers.edu/~pxk/416/notes/c-tutorials/getopt.html
3. http://courses.cms.caltech.edu/cs11/material/c/mike/misc/cmdline_args.html
4. Man pages for system calls.