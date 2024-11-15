# 深入理解`/proc`文件系统

> [/proc](https://docs.kernel.org/filesystems/proc.html)
> [procfs](https://en.wikipedia.org/wiki/Procfs)

* `/proc` directory is NOT a real File System，It is a Virtual File System
* It is mapped to `/proc` and mounted at boot time.
* These files are listed, but don’t actually exist on disk; the operating system creates them on the fly if you try to read them.
* You need to work as root to be able to examine the whole directory; some of the files (such as the process-related ones) are owned by the user who launched it.

## `/proc` directory organization

There are two type of files under `/proc`, a whole bunch of numbered directories representing processes, and some familiar sounding files.

* a special `self` symlink points to the current process.
* In fact, several well-known utilities access the /proc directory to get their information. 

### some interesting files

- `/proc/version` uname command access this file
- `/proc/cpuinfo` processor data
- `/proc/loadavg` the average load on the processor in minutes
- `/proc/stat` the statistics goes back to the last boot
- `/proc/uptime` up & idle seconds
- `/proc/meminfo`
- `/proc/interrupts`
- `/proc/acpi` Advanced Configuration and Power Interface
- `/proc/cmdline` the parameters that were passed to the kernel at boot time
- `/proc/filesystems` which filesystem types are supported by this kernel
- `/proc/mounts` Shows all the mounts used by your machine
- `/proc/net` loads of files related to io, network, even RAM
- `/proc/iomem ` Current system memory map for devices.
- `/proc/ioports`  Registered port regions for input output communication with device.

### what's in a process?

The numerical named directories represent all running processes. When a process ends, its /proc directory disappears automatically.

- `cmdline` The command that started the process, with all its parameters.
- `cwd`: A symlink to the current working directory (CWD) for the process; exe links to the process executable, and root links to its root directory.
- `environ`: All environment variables for the process.
- `fd`: Contains all file descriptors for a process, showing which files or devices it is using.
- `maps`, `statm`, and `mem`: Deal with the memory in use by the process.
- `stat` and `status`: Provide information about the status of the process, but the latter is far clearer than the former.

### about sys

/proc/sys not only provides information about the system, it also allows you to change kernel parameters on the fly, and enable or disable features. (Of course, this could prove harmful to your system — consider yourself warned!)

## [/proc/meminfo](https://man7.org/linux/man-pages/man5/proc.5.html)

The _/proc/meminfo_ file inside the [_/proc_](https://www.baeldung.com/linux/cli-hardware-info#the-proc-pseudo-filesystem) pseudo-filesystem provides a usage report about memory on the system. When we want to find out statistics like used and available memory, swap space, or cache and buffers, we can analyze this file’s contents.

```shell
MemTotal:       15763192 kB #total usable RAM
MemFree:         7430864 kB #free RAM, the memory which is not used for anything at all
MemAvailable:   13780888 kB #available RAM, the amount of memory available for allocation to any process
Buffers:          278824 kB #temporary storage element in memory, which doesn’t generally exceed 20 MB
Cached:          5988968 kB #page cache size (cache for files read from the disk), which also includes _tmpfs_ and _shmem_ but excludes _SwapCached_
SwapCached:            0 kB
Active:          2272176 kB
Inactive:        5064036 kB
Active(anon):     963616 kB
Inactive(anon):    44952 kB
Active(file):    1308560 kB
Inactive(file):  5019084 kB
Unevictable:        9716 kB
Mlocked:            9716 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:               432 kB
Writeback:             0 kB
AnonPages:       1078124 kB
Mapped:           296340 kB
Shmem:            157544 kB
Slab:             559496 kB
SReclaimable:     357824 kB
SUnreclaim:       201672 kB
KernelStack:       10304 kB
PageTables:        13252 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     7881596 kB
Committed_AS:    5382328 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
Percpu:             4608 kB
HardwareCorrupted:     0 kB
AnonHugePages:    155648 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      294768 kB
DirectMap2M:    10190848 kB
DirectMap1G:     8388608 kB
```