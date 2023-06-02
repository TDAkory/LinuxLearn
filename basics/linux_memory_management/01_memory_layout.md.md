# [物理内存布局探测](https://zhuanlan.zhihu.com/p/435020338)

## 获取内存总大小

内存总大小等信息作为设备的关键信息，应该在硬件启动初期就由CPU获得并存储，操作系统只需要通过CPU的相关协定读取即可，这个协定就是BIOS中断。 在x86芯片中，探测物理内存布局用的BIOS中断向量是0x15，根据ax寄存器值的不同，有三种常见的方式：0xe820，0x801和0x88。

### E820中断

在E820中断的输入中，其中AX是子向量号；CX存储返回结果的长度，其值必须大于等于20，原因参考输出说明；EDX默认为0x534D4150，分别代表'S'，'M'，'A'，'P'这四个字符的ASCII值；DI表示存储返回结果的地址，这里要结合ES寄存器的内存，实际存储的地址为ES:DI(ES << 4 + DI)。

在e820中断的输出中，EAX默认是0x534D4150h，意义参考输入，用来做魔术字校验；CX表示返回结果的长度，默认是20个字节；EBX用来标识是否是最后一项，0表示已经结束；DI是指向返回结果，和输入的DI值相同；如果出错，eflags的CF字段会被置1，相应错误码在AH寄存器。

```
Input:
AX = E820h
EAX = 0000E820h
EDX = 534D4150h ('SMAP')
EBX = continuation value or 00000000h to start at beginning of map
ECX = size of buffer for result, in bytes (should be >= 20 bytes)
ES:DI -> buffer for result (see #00581)
int 0x15

Ouput:
CF clear if successful
EAX = 534D4150h ('SMAP')
ES:DI buffer filled
EBX = next offset from which to copy or 00000000h if all done
ECX = actual length returned in bytes
CF set on error
AH = error code (86h) (see #00496 at INT 15/AH=80h)
```

这里分析下DI指向的返回结果。每一趟调用默认返回的是20字节，分别是8字节的基地址 + 8字节的大小 + 4字节的内存类型，其中内存类型常见的有如下几种：

1. 系统可用内存
2. 系统保留内存，例如ROM
3. ACPI Reclaim内存
4. ACPI NVS内存
5. other

### E801中断 和 88中断 略

## 源码分析

```c
// file: arch\x86\boot\memory.c
/* 物理内存布局探测总入口 */
void detect_memory(void)
{
    /* 使用e820 BIOS中断获取物理内存布局 */
    detect_memory_e820();
    /* 使用e801 BIOS中断获取物理内存布局 */
    detect_memory_e801();
    /* 使用88 BIOS中断获取物理内存布局 */
    detect_memory_88();
}
```

```c
// arch\x86\include\uapi\asm\bootparam.h
/* E820每个实体占20字节，8字节基地址 + 8字节长度 + 4字节被内存类型 */
struct boot_e820_entry {
    __u64 addr;
    __u64 size;
    __u32 type;
} __attribute__((packed));


// file: arch\x86\boot\memory.c
/* 通过E820中断探测 */
static void detect_memory_e820(void)
{
    /* E820探测到的内存块数目 */
    int count = 0;
    /* BIOS中断输入和输出寄存器 */
    struct biosregs ireg, oreg;
    /* E820探测结果存放到boot_params.e820_table中 */
    struct boot_e820_entry *desc = boot_params.e820_table;
    static struct boot_e820_entry buf; /* static so it is zeroed */

    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof(buf);
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;

    /*
     * Note: at least one BIOS is known which assumes that the
     * buffer pointed to by one e820 call is the same one as
     * the previous call, and only changes modified fields.  Therefore,
     * we use a temporary buffer and copy the results entry by entry.
     *
     * This routine deliberately does not try to account for
     * ACPI 3+ extended attributes.  This is because there are
     * BIOSes in the field which report zero for the valid bit for
     * all ranges, and we don't currently make any use of the
     * other attribute bits.  Revisit this if we see the extended
     * attribute bits deployed in a meaningful way in the future.
     */

    do {
        intcall(0x15, &ireg, &oreg);
        ireg.ebx = oreg.ebx; /* for next iteration... */

        /* BIOSes which terminate the chain with CF = 1 as opposed
           to %ebx = 0 don't always report the SMAP signature on
           the final, failing, probe. */
        /* 如果eflags的CF被置位，则出错 */
        if (oreg.eflags & X86_EFLAGS_CF)
            break;

        /* Some BIOSes stop returning SMAP in the middle of
           the search loop.  We don't know exactly how the BIOS
           screwed up the map at that point, we might have a
           partial map, the full map, or complete garbage, so
           just return failure. */
        /* 校验魔术字 */
        if (oreg.eax != SMAP) {
            count = 0;
            break;
        }

        /* buf为返回的结果，存放到boot_params.e820_table中 */
        *desc++ = buf;
        count++;
    } while (ireg.ebx && count < ARRAY_SIZE(boot_params.e820_table));

    /* 记录E820探测到的内存块数目 */
    boot_params.e820_entries = count;
}
```

## 总结

操作系统怎样获取设备总内存大小？
答：通过BIOS 0x15中断，常见有E820、E801和E88子中断号。

设备的所有内存，操作系统都可以使用吗？
答：不是的，只有内存类型为usable的才能被操作系统所使用。

