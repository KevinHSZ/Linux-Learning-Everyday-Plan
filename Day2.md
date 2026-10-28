# Day 2: **`inode`**

**Opening** : Today we will talk about ``inode``, the core building block of the Linux filesystem.

**Main content** Every file and directory on a Linux filesystem has an `inode`. An `inode` stores all metadata about a file, except its filename and the actual file content. Metadata inside an `inode` includes: file size, permission bits, owner UID/GID, timestamps (`atime`, `mtime`, `ctime`), block pointers pointing to data blocks, hard link count.

The filename lives inside the directory entry. A directory is just a special file that maps names to `inode` numbers. That is why hard links exist: multiple filenames can point to the same `inode`. They share all metadata and data. If you delete one filename, the `inode` is freed only when the link count drops to zero.

an inode is indeed a C struct: `struct inode`. The on-disk inode (stored on your disk, inside ext4/xfs) and the in-memory `struct inode` are different.

- On-disk inode: compact, stored on the filesystem, persistent. It keeps basic metadata.
- In-memory `struct inode`: loaded into RAM when the file is accessed. The kernel builds this C structure to operate the file. It adds runtime fields that do not exist on disk.

Fields you can find in `struct inode` (simplified):

- `i_ino`: inode number
- `i_mode`: file type + permission bits
- `i_uid`, `i_gid`: owner and group ID
- `i_size`: file byte size
- `i_atime`, `i_mtime`, `i_ctime`: three timestamps
- `i_nlink`: hard link count
- `i_bdev`: if this is a block device file
- `i_fop`: file operation table (`struct file_operations`), this is the function pointer table for read, write, open, release
- `i_sb`: super block pointer, tells which filesystem this inode belongs to
- `i_private`: private data for filesystem driver

Important separation: The directory entry (dentry, `struct dentry`) is another kernel structure. `dentry` stores filename and points to the `struct inode`. One `struct inode` can be referenced by multiple `struct dentry` → hard links

Commands you can try:

- `ls -i` : view `inode` number
- `stat <file>` : show full `inode` information

Key distinction: Hard links share one `inode`. Symbolic links (soft links) have their own separate `inode`.