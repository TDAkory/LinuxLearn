# [Memory Barrier](https://www.kernel.org/doc/Documentation/memory-barriers.txt)

> 非常好的一篇文章，对其关键点进行摘抄，不做翻译是为了保留作者的原始含义
>
> 一个译文参考：http://ifeve.com/linux-memory-barriers/

## excerpt from the article

### GUARANTEES

There are some minimal guarantees that may be expected of a CPU:

1.  On any given CPU, dependent memory accesses will be issued in order, with
     respect to itself.  This means that for:

    `Q = READ_ONCE(P); D = READ_ONCE(*Q);`

     the CPU will issue the following memory operations:

    `Q = LOAD P, D = LOAD *Q`

     and always in that order.  However, on DEC Alpha, READ_ONCE() also
     emits a memory-barrier instruction, so that a DEC Alpha CPU will
     instead issue the following memory operations:

    `Q = LOAD P, MEMORY_BARRIER, D = LOAD *Q, MEMORY_BARRIER`

     Whether on DEC Alpha or not, the READ_ONCE() also prevents compiler
     mischief.

2.  Overlapping loads and stores within a particular CPU will appear to be
     ordered within that CPU.  This means that for:

    `a = READ_ONCE(*X); WRITE_ONCE(*X, b);`

     the CPU will only issue the following sequence of memory operations:

    `a = LOAD *X, STORE *X = b`

     And for:

    `WRITE_ONCE(*X, c); d = READ_ONCE(*X);`

     the CPU will only issue:

    `STORE *X = c, d = LOAD *X`

     (Loads and stores overlap if they are targeted at overlapping pieces of
     memory).

And there are a number of things that _must_ or _must_not_ be assumed:

* It **_must_not_** be assumed that the compiler will do what you want with memory references that are not protected by READ_ONCE() and WRITE_ONCE().  Without them, the compiler is within its rights to do all sorts of "creative" transformations, which are covered in the COMPILER BARRIER section.
* It **_must_not_** be assumed that independent loads and stores will be issued in the order given.
* It **_must_** be assumed that overlapping memory accesses may be merged ordiscarded.

And there are anti-guarantees:

* These guarantees do not apply to bitfields, because compilers often generate code to modify these using non-atomic read-modify-write sequences.  Do not attempt to use bitfields to synchronize parallel algorithms.
* Even in cases where bitfields are protected by locks, all fields in a given bitfield must be protected by one lock.  If two fields in a given bitfield are protected by different locks, the compiler's non-atomic read-modify-write sequences can cause an update to one field to corrupt the value of an adjacent field.
* These guarantees apply only to properly aligned and sized scalar variables.

### what are memory barriers

Memory barriers come in four basic varieties:

1. **Write (or store) memory barriers** : A write memory barrier gives a guarantee that all the STORE operations specified before the barrier will appear to happen before all the STORE operations specified after the barrier with respect to the other components of the system. （**在写屏障之前的STORE操作将先于所有在写屏障之后的STORE操作**）
2. **Address-dependency barriers (historical)** : a weaker form of read barrier.  In the case where two loads are performed such that the second depends on the result of the first (eg: the first load retrieves the address to which the second load will be directed), an address-dependency barrier would be required to make sure that the target of the second load is updated after the address obtained by the first load is accessed. （**两条Load指令，第二条Load指令依赖于第一条Load指令的结果，则数据依赖屏障保障第二条指令的目标地址将被更新**）
3. **Read (or load) memory barriers** : A read barrier is an address-dependency barrier plus a guarantee that all the LOAD operations specified before the barrier will appear to happen before all the LOAD operations specified after the barrier with respect to the other components of the system. （**读屏障包含数据依赖屏障的功能, 并且保证所有出现在屏障之前的LOAD操作都将先于所有出现在屏障之后的LOAD操作被系统中的其他组件所感知**）
4. **General memory barriers** : A general memory barrier gives a guarantee that all the LOAD and STORE operations specified before the barrier will appear to happen before all the LOAD and STORE operations specified after the barrier with respect to the other components of the system. A general memory barrier is a partial ordering over both loads and stores. General memory barriers imply both read and write memory barriers, and so can substitute for either. （**通用内存屏障保证所有出现在屏障之前的LOAD和STORE操作都将先于所有出现在屏障之后的LOAD和STORE操作被系统中的其他组件所感知**）

And a couple of implicit varieties:

5. **ACQUIRE operations** : It guarantees that all memory operations after the ACQUIRE operation will appear to happen after the ACQUIRE operation with respect to the other components of the system. Memory operations that occur before an ACQUIRE operation may appear to happen after it completes. An ACQUIRE operation should almost always be paired with a RELEASE operation. （**LOCK操作.它的作用相当于一个单向渗透屏障.它保证所有出现在LOCK之后的内存操作都将在LOCK操作被系统中的其他组件所感知之后才能发生. 出现在LOCK之前的内存操作可能在LOCK完成之后才发生.LOCK操作总是跟UNLOCK操作配对出现**）
6. **RELEASE operations** : It guarantees that all memory operations before the RELEASE operation will appear to happen before the RELEASE operation with respect to the other components of the system.  Memory operations that occur after a RELEASE operation may appear to happen before it completes. （**UNLOCK操作。它保证所有出现在UNLOCK之前的内存操作都将在UNLOCK操作被系统中的其他组件所感知之前发生**）

     The use of ACQUIRE and RELEASE operations generally precludes the need for other sorts of memory barrier.  In addition, a RELEASE+ACQUIRE pair is -not- guaranteed to act as a full memory barrier.  However, after an ACQUIRE on a given variable, all memory accesses preceding any prior RELEASE on that same variable are guaranteed to be visible.  In other words, within a given variable's critical section, all accesses of all previous critical sections for that variable are guaranteed to have completed.

Memory barriers are only required where there's a possibility of interaction between two CPUs or between a CPU and a device.  If it can be guaranteed that there won't be any such interaction in any particular piece of code, then memory barriers are unnecessary in that piece of code.

There are certain things that the Linux kernel memory barriers do not guarantee:

* There is no guarantee that any of the memory accesses specified before a memory barrier will be _complete_ by the completion of a memory barrier instruction; the barrier can be considered to draw a line in that CPU's access queue that accesses of the appropriate type may not cross.

* There is no guarantee that issuing a memory barrier on one CPU will have any direct effect on another CPU or any other hardware in the system.  The indirect effect will be the order in which the second CPU sees the effects of the first CPU's accesses occur, but see the next point:

* There is no guarantee that a CPU will see the correct order of effects from a second CPU's accesses, even _if_ the second CPU uses a memory barrier, unless the first CPU _also_ uses a matching memory barrier (see the subsection on "SMP Barrier Pairing").

* There is no guarantee that some intervening piece of off-the-CPU hardware[*] will not reorder the memory accesses.  CPU cache coherency mechanisms should propagate the indirect effects of a memory barrier between CPUs, but might not do so in order.

#### ADDRESS-DEPENDENCY BARRIERS (HISTORICAL)

#### CONTROL DEPENDENCIES

#### SMP BARRIER PAIRING

When dealing with CPU-CPU interactions, certain types of memory barrier should always be paired.  A lack of appropriate pairing is almost certainly an error.

General barriers pair with each other, though they also pair with most other types of barriers, albeit without multicopy atomicity.  An acquire barrier pairs with a release barrier, but both may also pair with other barriers, including of course general barriers.  A write barrier pairs with an address-dependency barrier, a control dependency, an acquire barrier, a release barrier, a read barrier, or a general barrier.  Similarly a read barrier, control dependency, or an address-dependency barrier pairs with a write barrier, an acquire barrier, a release barrier, or a general barrier:

```shell
    CPU 1                    CPU 2
    ===============          ===============
    WRITE_ONCE(a, 1);
    <write barrier>
    WRITE_ONCE(b, 2);        x = READ_ONCE(b);
                             <read barrier>
                             y = READ_ONCE(a);

Or:

    CPU 1                    CPU 2
    ===============          ===============================
    a = 1;
    <write barrier>
    WRITE_ONCE(b, &a);       x = READ_ONCE(b);
                             <implicit address-dependency barrier>
                             y = *x;

Or even:

    CPU 1                    CPU 2
    ===============          ===============================
    r1 = READ_ONCE(y);
    <general barrier>
    WRITE_ONCE(x, 1);        if (r2 = READ_ONCE(x)) {
                                 <implicit control dependency>
                                 WRITE_ONCE(y, 1);
                             }

    assert(r1 == 0 || r2 == 0);
```

Basically, the read barrier always has to be there, even though it can be of the "weaker" type.

[!] Note that the stores before the write barrier would normally be expected to match the loads after the read barrier or the address-dependency barrier, and vice versa:

```shell
     CPU 1                               CPU 2
     ===================                 ===================
     WRITE_ONCE(a, 1);    }----   --->{  v = READ_ONCE(c);
     WRITE_ONCE(b, 2);    }    \ /    {  w = READ_ONCE(d);
     <write barrier>            \        <read barrier>
     WRITE_ONCE(c, 3);    }    / \    {  x = READ_ONCE(a);
     WRITE_ONCE(d, 4);    }----   --->{  y = READ_ONCE(b);
```

#### EXAMPLES OF MEMORY BARRIER SEQUENCES

1. write barriers act as partial orderings on store operations.

```shell
    CPU 1
    =======================
    STORE A = 1
    STORE B = 2
    STORE C = 3
    <write barrier>
    STORE D = 4
    STORE E = 5

    +-------+       :      :
    |       |       +------+
    |       |------>| C=3  |     }     /\
    |       |  :    +------+     }-----  \  -----> Events perceptible to
    |       |  :    | A=1  |     }        \/       the rest of the system
    |       |  :    +------+     }
    | CPU 1 |  :    | B=2  |     }
    |       |       +------+     }
    |       |   wwwwwwwwwwwwwwww }   <--- At this point the write barrier
    |       |       +------+     }        requires all stores prior to the
    |       |  :    | E=5  |     }        barrier to be committed before
    |       |  :    +------+     }        further stores may take place
    |       |------>| D=4  |     }
    |       |       +------+
    +-------+       :      :
                       |
                       | Sequence in which stores are committed to the
                       | memory system by CPU 1
                       V
```

2. address-dependency barriers act as partial orderings on address-dependent loads.

```shell
# Before
    CPU 1                      CPU 2
    =======================    =======================
        { B = 7; X = 9; Y = 8; C = &Y }
    STORE A = 1
    STORE B = 2
    <write barrier>
    STORE C = &B               LOAD X
    STORE D = 4                LOAD C (gets &B)
                               LOAD *C (reads B)

    +-------+       :      :                :       :
    |       |       +------+                +-------+  | Sequence of update
    |       |------>| B=2  |-----       --->| Y->8  |  | of perception on
    |       |  :    +------+     \          +-------+  | CPU 2
    | CPU 1 |  :    | A=1  |      \     --->| C->&Y |  V
    |       |       +------+       |        +-------+
    |       |   wwwwwwwwwwwwwwww   |        :       :
    |       |       +------+       |        :       :
    |       |  :    | C=&B |---    |        :       :       +-------+
    |       |  :    +------+   \   |        +-------+       |       |
    |       |------>| D=4  |    ----------->| C->&B |------>|       |
    |       |       +------+       |        +-------+       |       |
    +-------+       :      :       |        :       :       |       |
                                   |        :       :       |       |
                                   |        :       :       | CPU 2 |
                                   |        +-------+       |       |
        Apparently incorrect --->  |        | B->7  |------>|       |
        perception of B (!)        |        +-------+       |       |
                                   |        :       :       |       |
                                   |        +-------+       |       |
        The load of X holds --->    \       | X->9  |------>|       |
        up the maintenance           \      +-------+       |       |
        of coherence of B             ----->| B->2  |       +-------+
                                            +-------+
                                            :       :

# After
    CPU 1                      CPU 2
    =======================    =======================
        { B = 7; X = 9; Y = 8; C = &Y }
    STORE A = 1
    STORE B = 2
    <write barrier>
    STORE C = &B               LOAD X
    STORE D = 4                LOAD C (gets &B)
                               <address-dependency barrier>
                               LOAD *C (reads B)

    +-------+       :      :                :       :
    |       |       +------+                +-------+
    |       |------>| B=2  |-----       --->| Y->8  |
    |       |  :    +------+     \          +-------+
    | CPU 1 |  :    | A=1  |      \     --->| C->&Y |
    |       |       +------+       |        +-------+
    |       |   wwwwwwwwwwwwwwww   |        :       :
    |       |       +------+       |        :       :
    |       |  :    | C=&B |---    |        :       :       +-------+
    |       |  :    +------+   \   |        +-------+       |       |
    |       |------>| D=4  |    ----------->| C->&B |------>|       |
    |       |       +------+       |        +-------+       |       |
    +-------+       :      :       |        :       :       |       |
                                   |        :       :       |       |
                                   |        :       :       | CPU 2 |
                                   |        +-------+       |       |
                                   |        | X->9  |------>|       |
                                   |        +-------+       |       |
      Makes sure all effects --->   \   aaaaaaaaaaaaaaaaa   |       |
      prior to the store of C        \      +-------+       |       |
      are perceptible to              ----->| B->2  |------>|       |
      subsequent loads                      +-------+       |       |
                                            :       :       +-------+
```

3. a read barrier acts as a partial order on loads.

```shell
    CPU 1                      CPU 2
    =======================    =======================
                    { A = 0, B = 9 }
    STORE A=1
    <write barrier>
    STORE B=2
                               LOAD B
                               LOAD A

    +-------+       :      :                :       :
    |       |       +------+                +-------+
    |       |------>| A=1  |------      --->| A->0  |
    |       |       +------+      \         +-------+
    | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
    |       |       +------+        |       +-------+
    |       |------>| B=2  |---     |       :       :
    |       |       +------+   \    |       :       :       +-------+
    +-------+       :      :    \   |       +-------+       |       |
                                 ---------->| B->2  |------>|       |
                                    |       +-------+       | CPU 2 |
                                    |       | A->0  |------>|       |
                                    |       +-------+       |       |
                                    |       :       :       +-------+
                                     \      :       :
                                      \     +-------+
                                       ---->| A->1  |
                                            +-------+
                                            :       :

    CPU 1                      CPU 2
    =======================    =======================
                    { A = 0, B = 9 }
    STORE A=1
    <write barrier>
    STORE B=2
                               LOAD B
                               <read barrier>
                               LOAD A

    +-------+       :      :                :       :
    |       |       +------+                +-------+
    |       |------>| A=1  |------      --->| A->0  |
    |       |       +------+      \         +-------+
    | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
    |       |       +------+        |       +-------+
    |       |------>| B=2  |---     |       :       :
    |       |       +------+   \    |       :       :       +-------+
    +-------+       :      :    \   |       +-------+       |       |
                                 ---------->| B->2  |------>|       |
                                    |       +-------+       | CPU 2 |
                                    |       :       :       |       |
                                    |       :       :       |       |
      At this point the read ---->   \  rrrrrrrrrrrrrrrrr   |       |
      barrier causes all effects      \     +-------+       |       |
      prior to the storage of B        ---->| A->1  |------>|       |
      to be perceptible to CPU 2            +-------+       |       |
                                            :       :       +-------+
```

To illustrate this more completely, consider what could happen if the code
contained a load of A either side of the read barrier:

The guarantee is that the second load will always come up with A == 1 if the
load of B came up with B == 2.  No such guarantee exists for the first load of
A; that may come up with either A == 0 or A == 1.

```shell
    CPU 1                      CPU 2
    =======================    =======================
                    { A = 0, B = 9 }
    STORE A=1
    <write barrier>
    STORE B=2
                               LOAD B
                               LOAD A [first load of A]
                               <read barrier>
                               LOAD A [second load of A]

    +-------+       :      :                :       :
    |       |       +------+                +-------+
    |       |------>| A=1  |------      --->| A->0  |
    |       |       +------+      \         +-------+
    | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
    |       |       +------+        |       +-------+
    |       |------>| B=2  |---     |       :       :
    |       |       +------+   \    |       :       :       +-------+
    +-------+       :      :    \   |       +-------+       |       |
                                 ---------->| B->2  |------>|       |
                                    |       +-------+       | CPU 2 |
                                    |       :       :       |       |
                                    |       :       :       |       |
                                    |       +-------+       |       |
                                    |       | A->0  |------>| 1st   |
                                    |       +-------+       |       |
      At this point the read ---->   \  rrrrrrrrrrrrrrrrr   |       |
      barrier causes all effects      \     +-------+       |       |
      prior to the storage of B        ---->| A->1  |------>| 2nd   |
      to be perceptible to CPU 2            +-------+       |       |
                                            :       :       +-------+

    +-------+       :      :                :       :
    |       |       +------+                +-------+
    |       |------>| A=1  |------      --->| A->0  |
    |       |       +------+      \         +-------+
    | CPU 1 |   wwwwwwwwwwwwwwww   \    --->| B->9  |
    |       |       +------+        |       +-------+
    |       |------>| B=2  |---     |       :       :
    |       |       +------+   \    |       :       :       +-------+
    +-------+       :      :    \   |       +-------+       |       |
                                 ---------->| B->2  |------>|       |
                                    |       +-------+       | CPU 2 |
                                    |       :       :       |       |
                                     \      :       :       |       |
                                      \     +-------+       |       |
                                       ---->| A->1  |------>| 1st   |
                                            +-------+       |       |
                                        rrrrrrrrrrrrrrrrr   |       |
                                            +-------+       |       |
                                            | A->1  |------>| 2nd   |
                                            +-------+       |       |
                                            :       :       +-------+
```

#### READ MEMORY BARRIERS VS LOAD SPECULATION

Many CPUs speculate with loads: that is they see that they will need to load an item from memory, and they find a time where they're not using the bus for any other loads, and so do the load in advance - even though they haven't actually got to that point in the instruction execution flow yet.  This permits the actual load instruction to potentially complete immediately because the CPU already has the value to hand.

#### MULTICOPY ATOMICITY

Multicopy atomicity is a deeply intuitive notion about ordering that is not always provided by real computer systems, namely that a given store becomes visible at the same time to all CPUs, or, alternatively, that all CPUs agree on the order in which all stores become visible.  However, support of full multicopy atomicity would rule out valuable hardware optimizations, so a weaker form called ``other multicopy atomicity`` instead guarantees only that a given store becomes visible at the same time to all -other- CPUs.  The remainder of this document discusses this weaker form, but for brevity will call it simply ``multicopy atomicity``.

**如果你的代码需要所有操作的完整排序，请在整个过程中使用通用屏障。**

### EXPLICIT KERNEL BARRIERS
