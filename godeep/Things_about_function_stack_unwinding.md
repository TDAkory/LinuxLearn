# [Function Stack Unwinding](https://blog.opsnull.com/posts/function-stack-unwinding/)

- [stack unwinding](https://stackoverflow.com/questions/2331316/what-is-stack-unwinding)

stack unwinding 指的是获得当前函数调用栈的过程，当前有几种实现方式：

* FP：frame pointer；
* DWARF CFI（Call Frame Information），如 .eh_frame Section 信息；
* ORC： 4.14 及以后版本内核专用，简化版的 DWARF CFI；
* LBR： 新的 Intel CPU 支持；