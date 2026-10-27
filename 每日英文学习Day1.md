# Linux Presentation Everyday Plan

**Great plan! Starting from today, I’ll give you a short Linux knowledge snippet (小片，片断，片段) in presentation style, English first + simple Chinese summary, not too long. You can use it for both English reading and Linux learning.**

------

## Day 1: `umask` 

**Opening** Today we are going to talk about **`umask`** a common but often misunderstood concept for Linux file permissions.

**Main content** When you create a new file or directory, Linux does not give it full default 777 permission directly. **`umask`** sets the permission bits to *take away* from the base permission.

- Base for files: 666
- Base for directories: 777

**Example**: If your **`umask`** is 022

- New file: 666 - 022 = 644 (`rw-r--r--`)
- New folder: 777 - 022 = 755 (`rwxr-xr-x`)

Important note: `umask` is NOT a simple subtraction in math. It masks permission bits. You cannot use `umask` to make a new executable file automatically. Files start without execute bits by default.

You can check your current mask with `umask`, set it temporarily with `umask 027`. To make it permanent, edit `.bashrc` or `/etc/profile`.

**Quick takeaway**

 `umask` controls the default permissions for newly created files and directories.

- `1 & 1 = 1`
- `1 & 0 = 0`
- `0 & 1 = 0`
- `0 & 0 = 0`

& and 



2026 10.26 11：30