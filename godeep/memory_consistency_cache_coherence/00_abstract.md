# Abstract

- [The Original Book](https://github.com/kaitoukito/Computer-Science-Textbooks/blob/master/A-Primer-on-Memory-Consistency-and-Cache-Coherence-2nd-Edition.pdf)
- [Translate](https://github.com/kaitoukito/A-Primer-on-Memory-Consistency-and-Cache-Coherence)

**内存连贯性(memory consistency)** 和 **缓存一致性(cache coherence)**

* The memory model specifies the allowed behavior of multithreaded programs executing with shared memory. For a multithreaded program executing with specific input data, the memory model specifies what values dynamic loads may return and, optionally, what possible final states of the memory are. 
* Access to stale data (incoherence) is prevented using a coherence protocol, which is a set of rules implemented by the distributed set of actors within a system. Coherence protocols come in many variants but follow a few themes, all of the variants make one processor’s write visible to the other processors by propagating the write to all caches. But protocols differ in when and how the syncing happens.