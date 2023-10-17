# [Unix Domain Socket](https://man7.org/linux/man-pages/man7/unix.7.html)

> 深入理解Unix Domain Socket

## SCM_RIGHTS

SCM_RIGHTS 传递的文件描述符表现的行为(behave)就像是接收进程使用 dup(2) 创建的一样。也就是说，SCM_RIGHTS 表面上传递的是文件描述符，但实际上并不是简单地传递描述符的数字，而是传递描述符背后的 file 文件。实际上，发送方和接收方的描述符数字是一样的可能性也是很小。