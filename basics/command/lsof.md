# [lsof](https://man7.org/linux/man-pages/man8/lsof.8.html)

list open files

An  open  file  may be a regular file, a directory, a block special file, a character special file, an executing text reference, a library, a stream or a network file (Internet socket, NFS file or UNIX domain socket.)  A specific file or all the files in a file  system  may  be  selected by path.

* `lsof -i:38262`: selects the listing of files any of whose Internet address matches the address specified in i.