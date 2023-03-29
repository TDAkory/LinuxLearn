# [The Lab](https://play.instruqt.com/embed/isovalent/tracks/ebpf-getting-started/challenges/build-run-opensnoop/notes?auto_start=true)

BPF is a revolutionary technology with origins in the Linux kernel that can run sandboxed programs in an operating system kernel. It is used to safely and efficiently extend the capabilities of the kernel without requiring to change kernel source code or load kernel modules.

##

* `opensnoop` an eBPF based tool that reports whenever a file is opened, is one of a collection of eBPF-based tools from the [BCC project](02_bcc.md)

## Examine the BPF object file
  
* `readelf --section-details --headers .output/opensnoop.bpf.o`

```shell
root@ubuntu-2110:~/bcc/libbpf-tools# readelf --section-details --headers .output/opensnoop.bpf.o
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Linux BPF
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          11304 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         20
  Section header string table index: 19

Section Headers:
  [Nr] Name
       Type              Address          Offset            Link
       Size              EntSize          Info              Align
       Flags
  [ 0] 
       NULL             0000000000000000  0000000000000000  0
       0000000000000000 0000000000000000  0                 0
       [0000000000000000]: 
  [ 1] .text
       PROGBITS         0000000000000000  0000000000000040  0
       0000000000000000 0000000000000000  0                 4
       [0000000000000006]: ALLOC, EXEC
  [ 2] tracepoint/syscalls/sys_enter_open
       PROGBITS         0000000000000000  0000000000000040  0
       0000000000000178 0000000000000000  0                 8
       [0000000000000006]: ALLOC, EXEC
  [ 3] tracepoint/syscalls/sys_enter_openat
       PROGBITS         0000000000000000  00000000000001b8  0
       0000000000000178 0000000000000000  0                 8
       [0000000000000006]: ALLOC, EXEC
  [ 4] tracepoint/syscalls/sys_exit_open
       PROGBITS         0000000000000000  0000000000000330  0
       00000000000002d0 0000000000000000  0                 8
       [0000000000000006]: ALLOC, EXEC
  [ 5] tracepoint/syscalls/sys_exit_openat
       PROGBITS         0000000000000000  0000000000000600  0
       00000000000002d0 0000000000000000  0                 8
       [0000000000000006]: ALLOC, EXEC
  [ 6] .rodata
       PROGBITS         0000000000000000  00000000000008d0  0
       000000000000000d 0000000000000000  0                 4
       [0000000000000002]: ALLOC
  [ 7] .maps
       PROGBITS         0000000000000000  00000000000008e0  0
       0000000000000038 0000000000000000  0                 8
       [0000000000000003]: WRITE, ALLOC
  [ 8] license
       PROGBITS         0000000000000000  0000000000000918  0
       0000000000000004 0000000000000000  0                 1
       [0000000000000003]: WRITE, ALLOC
  [ 9] .BTF
       PROGBITS         0000000000000000  000000000000091c  0
       0000000000000d2b 0000000000000000  0                 1
       [0000000000000000]: 
  [10] .BTF.ext
       PROGBITS         0000000000000000  0000000000001647  0
       00000000000007ec 0000000000000000  0                 1
       [0000000000000000]: 
  [11] .symtab
       SYMTAB           0000000000000000  0000000000001e38  19
       00000000000002d0 0000000000000018  19                8
       [0000000000000000]: 
  [12] .reltracepoint/syscalls/sys_enter_open
       REL              0000000000000000  0000000000002108  11
       0000000000000040 0000000000000010  2                 8
       [0000000000000000]: 
  [13] .reltracepoint/syscalls/sys_enter_openat
       REL              0000000000000000  0000000000002148  11
       0000000000000040 0000000000000010  3                 8
       [0000000000000000]: 
  [14] .reltracepoint/syscalls/sys_exit_open
       REL              0000000000000000  0000000000002188  11
       0000000000000040 0000000000000010  4                 8
       [0000000000000000]: 
  [15] .reltracepoint/syscalls/sys_exit_openat
       REL              0000000000000000  00000000000021c8  11
       0000000000000040 0000000000000010  5                 8
       [0000000000000000]: 
  [16] .rel.BTF
       REL              0000000000000000  0000000000002208  11
       0000000000000070 0000000000000010  9                 8
       [0000000000000000]: 
  [17] .rel.BTF.ext
       REL              0000000000000000  0000000000002278  11
       0000000000000780 0000000000000010  10                8
       [0000000000000000]: 
  [18] .llvm_addrsig
       LOOS+0xfff4c03   0000000000000000  00000000000029f8  0
       000000000000000b 0000000000000000  0                 1
       [0000000080000000]: EXCLUDE
  [19] .strtab
       STRTAB           0000000000000000  0000000000002a03  0
       0000000000000224 0000000000000000  0                 1
       [0000000000000000]: 

There are no program headers in this file.
```

A few interesting things to observe

* Machine is "Linux BPF"
* There is BTF information included in this file
* After the section header named .text in the table, there are four executable sections starting with tracepoint. These correspond to four BPF programs.

you can open the file `opensnoop.bpf.c`. Scroll down to find four different functions with names beginning with int `tracepoint__syscalls....` You should find them on lines 50, 68, 118 and 124. Each of these is preceded by a `SEC()` macro which correspond to the executable sections listed by readelf.

```c
SEC("tracepoint/syscalls/sys_enter_open")
int tracepoint__syscalls__sys_enter_open(struct trace_event_raw_sys_enter* ctx)
{
	u64 id = bpf_get_current_pid_tgid();
	/* use kernel terminology here for tgid/pid: */
	u32 tgid = id >> 32;
	u32 pid = id;

	/* store arg info for later lookup */
	if (trace_allowed(tgid, pid)) {
		struct args_t args = {};
		args.fname = (const char *)ctx->args[0];
		args.flags = (int)ctx->args[1];
		bpf_map_update_elem(&start, &pid, &args, 0);
	}
	return 0;
}
```

## See BPF programs in the Kernel

Let's have a look if we have any eBPF programs running on our machine. `bpftool prog list`

```shell
root@ubuntu-2110:~# bpftool prog list
228: tracepoint  name tracepoint__sys  tag 9f196d70d0c1964b  gpl
        loaded_at 2023-03-29T06:12:11+0000  uid 0
        xlated 248B  jited 140B  memlock 4096B  map_ids 5,2
        btf_id 34
230: tracepoint  name tracepoint__sys  tag 47b06acd3f9a5527  gpl
        loaded_at 2023-03-29T06:12:11+0000  uid 0
        xlated 248B  jited 140B  memlock 4096B  map_ids 5,2
        btf_id 34
231: tracepoint  name tracepoint__sys  tag 387291c2fb839ac6  gpl
        loaded_at 2023-03-29T06:12:11+0000  uid 0
        xlated 696B  jited 475B  memlock 4096B  map_ids 2,5,3
        btf_id 34
232: tracepoint  name tracepoint__sys  tag 387291c2fb839ac6  gpl
        loaded_at 2023-03-29T06:12:11+0000  uid 0
        xlated 696B  jited 475B  memlock 4096B  map_ids 2,5,3
        btf_id 34
```

You see four more BPF programs loaded! Those correspond to the four `opensnoop` BPF programs mentioned earlier. They are all of type tracepoint. Note that the names are truncated so you can’t really see which is for entry or exit.

However, they each refer to two or three map IDs, like `map_ids 17,14` (the numbers might be different). Let's use this information!

```shell
root@ubuntu-2110:~# bpftool map list
2: hash  name start  flags 0x0
        key 4B  value 16B  max_entries 10240  memlock 245760B
        btf_id 34
3: perf_event_array  name events  flags 0x0
        key 4B  value 4B  max_entries 1  memlock 4096B
5: array  name opensnoo.rodata  flags 0x480
        key 4B  value 13B  max_entries 1  memlock 4096B
        btf_id 34  frozen
```

Observe that there is a hash table with name `start` and a perf event array with name `events`. These are defined in the source code in `~/bcc/libbpf-tools/opensnoop.bpf.c lines 13-24`. 

```c
struct {
	__uint(type, BPF_MAP_TYPE_HASH);
	__uint(max_entries, 10240);
	__type(key, u32);
	__type(value, struct args_t);
} start SEC(".maps");
```

There is also an array for opensnoop read-only data (`array name opensnoo.rodata`).

At the start of each line, you see the ID of the corresponding BPF program. Take an ID of a tracepoint program, and dump the bytecode `bpftool prog dump xlated id 48 linum`

## Write our own code

eBPF programs can write tracing messages for debugging purposes. Those can be read from `/sys/kernel/debug/tracing/trace_pipe`. So let’s do exactly that and add our own message into `opensnoop`!