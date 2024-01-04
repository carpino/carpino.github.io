---
layout: post
title:  "Communicating over UDP with system calls in Go"
date:   2024-01-04 16:00:00 +0100
categories: jekyll update
tags:
    - networking
    - go
    - programming
summary: We will see how to send and receive data over UDP using directly system calls, using the methods of the go's syscall package, that are implemented using the libc libraries of the OS provider...
---

# UDP

`User Datagram Protocol` (UDP) is the most used connectionless protocol over IP. 
Other than used directly, UDP is used to implement other network protocols such as in the field of audio-video streaming and VoIP. 

UDP does not implements retransmission of the lost packets. In this way the packets have less guarantees to be delivered in case of networks errors if compared to TCP.

Packets are not properly sequenced, that means they could arrive in a different order respect to which they are sent.

UDP is implemented at kernel level in the OS and is used by the networking libraries through a serie of system calls, these system calls can be implemented in other languages like C by the OS provider so that the networking libraries of the language of reference has not to implement them themselves at low level. Typically these implementation are inside the `libc` libraries.

The main syscalls for using UDP are:

* Socket
* Bind
* Sendto
* Recvfrom
* Close

#### Socket

Create a new socket used in subsequent system calls. 
In the kernel level is created a `Protocol Control Block` (PCB) that will contain all the informations about the current connection.
This is done through two calls, one for the allocation of the new PCB and one for its initialization.

#### Bind

Bind a socket with a local transport network address, that is address and port. 
The port can also be not defined, in that case an implicit lookup will be tried by the system.
In case the port is already in use in the system by onehter socket, the bind will fail.

#### Sendto

Send an UDP datagram from the socket address and port to the destination socket. 
Successful completion of a call does not guarantee delivery of the message. A return value of -1 indicates only locally-detected errors.
If space is not available at the sending socket to hold the message to be transmitted it could block until space is available. 

#### Recvfrom

Put the process in sleep until the kernel wake it up with a received UDP datagram along with the address of the UDP datagram sender.

#### Close

Free the local socket.

# Syscalls in GO

The syscall package provides a mean to interact with the underlying operating system directly.
However, it is essential to note that the syscall package is platform-specific, and some functions might not be available or behave differently across different platforms.

For what regards at least the UDP calls, they are implemented in the syscall package under the hood with the use of the libc libraries.
`libc` is responsible for the mechanism of system calls to the OS implemented in low level code.

For our purposes we will use the procedures:

* `func syscall.Socket(domain int, typ int, proto int) (fd int, err error)`
* `func syscall.Bind(fd int, sa syscall.Sockaddr) (err error)`
* `func syscall.Sendto(fd int, p []byte, flags int, to syscall.Sockaddr) (err error)`
* `func syscall.Recvfrom(fd int, p []byte, flags int) (n int, from syscall.Sockaddr, err error)`
* `func syscall.Close(fd int) (err error)`

For all the procedures `fd` indicates a file descriptor.

For the Socket procedure, `domain` indicates the address family, `type` the type of socket(TCP or UDP), and `proto` the underlaing protocol level IP.

# Example code

Here some example code using the previous described syscalls to send an hello message from one socket to the other.

#### Client

{% highlight golang %}
fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_DGRAM, syscall.IPPROTO_IP)
if err != nil {
	panic(err)
}

err = syscall.Bind(fd, &syscall.SockaddrInet4{Port: 8081, Addr: [4]byte{127, 0, 0, 1}})
if err != nil {
	panic(err)
}

err = syscall.Sendto(fd, []byte("hello world!"), 0, &syscall.SockaddrInet4{Port: 8080, Addr: [4]byte{127, 0, 0, 1}})
if err != nil {
	panic(err)
}

err = syscall.Close(fd)
if err != nil {
	panic(err)
}
{% endhighlight %}

#### Server

{% highlight golang %}
fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_DGRAM, syscall.IPPROTO_IP)
if err != nil {
	panic(err)
}

err = syscall.Bind(fd, &syscall.SockaddrInet4{Port: 8080, Addr: [4]byte{127, 0, 0, 1}})
if err != nil {
	panic(err)
}

msg := make([]byte, 1024)
size, from, err := syscall.Recvfrom(fd, msg, 0)
if err != nil {
	panic(err)
}

fmt.Printf("size: %d, from: %v, err: %v\nmsg: %s\n", size, from, err, string(msg[0:size]))

err = syscall.Close(fd)
if err != nil {
	panic(err)
}
{% endhighlight %}

# Summary

We have seen how to send and receive data over UDP using directly system calls, using the methods of the go's syscall package (that are implemented using the `libc` libraries of the OS provider).
We have tried to give some more awareness in the use of the UDP stack.
Hope you enjoied the article, if you want to either suggest some improvements or have an exhange please contact via `email` or via my `Twitter` profile. 

#### References
* [jameshfisher simple udp server][jameshfisher-simple-udp-server]

[jameshfisher-simple-udp-server]: https://jameshfisher.com/2016/12/19/simple-udp-server