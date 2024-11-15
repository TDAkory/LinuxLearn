# [Memory Barrier](https://www.kernel.org/doc/Documentation/memory-barriers.txt)

> 非常好的一篇文章，对其关键点进行摘抄，不做翻译是为了保留作者的原始含义

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

1. **Write (or store) memory barriers** : A write memory barrier gives a guarantee that all the STORE operations specified before the barrier will appear to happen before all the STORE operations specified after the barrier with respect to the other components of the system.
2. **Address-dependency barriers (historical)** : a weaker form of read barrier.  In the case where two loads are performed such that the second depends on the result of the first (eg: the first load retrieves the address to which the second load will be directed), an address-dependency barrier would be required to make sure that the target of the second load is updated after the address obtained by the first load is accessed.
3. **Read (or load) memory barriers** : A read barrier is an address-dependency barrier plus a guarantee that all the LOAD operations specified before the barrier will appear to happen before all the LOAD operations specified after the barrier with respect to the other components of the system.
4. **General memory barriers** : A general memory barrier gives a guarantee that all the LOAD and STORE operations specified before the barrier will appear to happen before all the LOAD and STORE operations specified after the barrier with respect to d the other components of the system.

And a couple of implicit varieties:

5. **ACQUIRE operations** : It guarantees that all memory operations after the ACQUIRE operation will appear to happen after the ACQUIRE operation with respect to the other components of the system. Memory operations that occur before an ACQUIRE operation may appear to happen after it completes. An ACQUIRE operation should almost always be paired with a RELEASE operation.
6. **RELEASE operations** : It guarantees that all memory operations before the RELEASE operation will appear to happen before the RELEASE operation with respect to the other components of the system.  Memory operations that occur after a RELEASE operation may appear to happen before it completes.

    The use of ACQUIRE and RELEASE operations generally precludes the need for other sorts of memory barrier.  In addition, a RELEASE+ACQUIRE pair is -not- guaranteed to act as a full memory barrier.  However, after an ACQUIRE on a given variable, all memory accesses preceding any prior RELEASE on that same variable are guaranteed to be visible.  In other words, within a given variable's critical section, all accesses of all previous critical sections for that variable are guaranteed to have completed.

Memory barriers are only required where there's a possibility of interaction between two CPUs or between a CPU and a device.  If it can be guaranteed that there won't be any such interaction in any particular piece of code, then memory barriers are unnecessary in that piece of code.

There are certain things that the Linux kernel memory barriers do not guarantee:

* There is no guarantee that any of the memory accesses specified before a
     memory barrier will be _complete_ by the completion of a memory barrier
     instruction; the barrier can be considered to draw a line in that CPU's
     access queue that accesses of the appropriate type may not cross.