# Linux内核内存模型

- [Linux-Kernel Memory Model](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0124r3.html)

探索这个话题的由来，是在阅读linux kernel代码的过程中，看到很多 READ_ONCE 和 WRITE_ONCE 这样的表达，从字面上可以理解是完成了一次‘独占的’内存读写，但并不理解其背后的含义。

