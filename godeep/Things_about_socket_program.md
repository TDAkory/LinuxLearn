# Socket Programing

- [Socket Programing](#socket-programing)
	- [What is socket](#what-is-socket)
	- [Basic Struct](#basic-struct)
		- [Socket](#socket)
		- [`setsockopt`](#setsockopt)
			- [notes](#notes)
			- [`socket-level options`](#socket-level-options)
	- [socket bind NIC](#socket-bind-nic)

## What is socket

* A network [socket](https://en.wikipedia.org/wiki/Network_socket) is a software structure within a network node of a computer network that serves as an endpoint for sending and receiving data across the network.
* A [Unix domain socket](https://en.wikipedia.org/wiki/Unix_domain_socket) aka UDS or IPC socket (inter-process communication socket) is a data communications endpoint for exchanging data between processes executing on the same host operating system. 
* [`socket() `](https://en.wikipedia.org/wiki/Berkeley_sockets) a system call defined by the [Berkeley sockets](https://en.wikipedia.org/wiki/Berkeley_sockets) API
* [CPU socket](https://en.wikipedia.org/wiki/CPU_socket), the connector on a computer's motherboard for the CPU

## Basic Struct

### Socket

```cpp
// /include/linux/net.h
/**
 *  struct socket - general BSD socket
 *  @state: socket state (%SS_CONNECTED, etc)
 *  @type: socket type (%SOCK_STREAM, etc)
 *  @flags: socket flags (%SOCK_NOSPACE, etc)
 *  @ops: protocol specific socket operations
 *  @file: File back pointer for gc
 *  @sk: internal networking protocol agnostic socket representation
 *  @wq: wait queue for several uses
 */
struct socket {
	socket_state		state;

	short			type;

	unsigned long		flags;

	struct file		*file;
	struct sock		*sk;
	const struct proto_ops	*ops;

	struct socket_wq	wq;
};
```

### `setsockopt`

```cpp
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);

int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

#### notes 

POSIX.1-2001 does not require the inclusion of <sys/types.h>, and this header file is not required on Linux. However, some historical (BSD) implementations required this header file, and portable applications are probably wise to include it.

#### [`socket-level options`](https://www.gnu.org/software/libc/manual/html_node/Socket_002dLevel-Options.html)


## socket bind NIC

**SO_BINDTODEVICE** 

[Deep thought of SO_BINDTODEVICE](https://stackoverflow.com/questions/1207746/problems-with-so-bindtodevice-linux-socket-option)

Bind this socket to a particular device like "eth0", as specified in the passed interface name. If the name is an empty string or the option length is zero, the socket device binding is removed. The passed option is a variable-length null-terminated interface name string with the maximum size of IFNAMSIZ. If a socket is bound to an interface, only packets received from that particular interface are processed by the socket. Note that this only works for some socket types, particularly AF_INET sockets. It is not supported for packet sockets (use normal bind(2) there).

Before Linux 3.8, this socket option could be set, but could not retrieved with getsockopt(2). Since Linux 3.8, it is readable. The optlen argument should contain the buffer size available to receive the device name and is recommended to be IFNAMSZ bytes. The real device name length is reported back in the optlen argument.