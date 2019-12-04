# File system

> https://www.kernel.org/doc/html/latest/filesystems/vfs.html

Linux kernel通过VFS给用户程序提供接口(open, stat, read, write, chmod...), 同时给不同的文件系统提供抽象.

## directory entry cache

当VFS发生系统调用时, 会通过pathname找到项. Directory entry cache(dcache for short)提供了快速查询的机制, 它们
存储在内存中, 而不会存在磁盘上.

## superblock

superblock记载了filesystem的整体信息(inode/block总量, 使用量, 剩余量), 以及filesystem的格式等信息.

## inode

An inode object represents an object within the filesystem, 例如文件, 目录, FIFOS...它可以保存在磁盘上(block device filesystems),
也可以保存在内存中(pseudo filesystems).

当使用Inode时, 会将它先拷贝到内存中, 当发生修改时再写会到磁盘中.

一个Inode可以被多个dentries指向(hard links, for example).

通过``lookup``方法查找``inode``. 一旦VFS拿到了dentry(也就拿到了inode), 就能够使用open, stat等方法了.

## The File Object

When we open a file, we need to allocate a file structure. The file structure is pointed to the dentry
and has a set of file operation member functions which taken from the inode data. For example, the ``open()``
file method is then called so the specific filesystem implememtation can do its work.

The file structure is placed into the **file descriptor table** for the process.

Reading, writing and closing files (and other assorted VFS operations) is done by using the userspace file
descriptor to grab the appropriate file structure, and then calling the required file structure method to
do whatever is required.

## The Address Space Object

The address space object is used to group and manage pages in the page cache. It can be used to keep track of
the pages in a file (or anything else) and also track the mapping of sections of the file into process address spaces.

## Registering and Mounting a Filesystem

To register and unregister a filesystem, use the following API functions:

```c
extern int register_filesystem(struct file_system_type *);
extern int unregister_filesystem(struct file_system_type *);
```

When a request is made to mount a filesystem onto a directory in your namespace, the VFS will call the
appropriate mount() method for the specific filesystem. New vfsmount referring to the tree returned by
``->mount()`` will be attached to the mountpoint, so that when pathname resolution reaches the mountpoint
it will jump into the root of that vfsmount.

## ``struct file_system_type``

```c
struct file_system_operations {
        const char *name;
        int fs_flags;
        struct dentry *(*mount) (struct file_system_type *, int,
                                 const char *, void *);
        void (*kill_sb) (struct super_block *);
        struct module *owner;
        struct file_system_type * next;
        struct list_head fs_supers;
        struct lock_class_key s_lock_key;
        struct lock_class_key s_umount_key;
};
```
