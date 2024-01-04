---
layout: post
title:  "Communicating over TCP with system calls in Go"
date:   2024-01-04 10:00:00 +0100
categories: jekyll update
---

# TCP

`Transmission Controll Protocol` (TCP) is the most used connection oriented network protocol over IP. 
Other than used directly, TCP is used also to implement other network protocols such as the widely known HTTP. 

TCP implements retransmission of the lost packets, for a number of times that has a default of 15, with the possibility for the client to receive duplicate packets, but these duplicates are managed by the protocol in a way that is transparent to the user.

Packets are properly sequenced, that means they arrive in the same sequence in which they are sent.

TCP is implemented at kernel level in the OS and is used by the networking libraries through a serie of system calls, these system calls can be implemented in other languages like C by the OS provider so that the networking libraries of the language of reference has not to implement them themselves at low level. These calls are implemented typically in the `libc` libraries.

The main syscalls for using TCP are:

* Socket
* Bind
* Listen
* Accept
* Connect 
* Write
* Read
* Close

#### Socket

Create a new socket used in subsequent system calls. 
In the kernel level is created a `Protocol Control Block` (PCB) that will contain all the informations about the current connection.
This is done through two calls, one for the allocation of the new PCB and one for its initialization.

#### Bind

Bind a socket with a local transport network address, that is address and port. 
The port can also be not defined, in that case an implicit lookup will be tried by the system.
In case the port is already in use in the system by onehter socket, the bind will fail.

#### Listen

Indicates to the protocol that the server process is ready to accept new incoming connections on the socket. 
It also indicates the number of connactions that can be queued, after which any further connection requests are ignored.
In the kernel level the call set the socket to listen state.

#### Accept

Blocking call that waits for incoming connections. 
At incoming connection, a new socket descriptor is returned. 
This new socket is connected to the client and the previous socket remains in LISTEN state.
At kernel level when a new connection arrives, it wake up the server process passing it the new connection after have checked for socket errors tah mya have occured in the meantime.

#### Connect

Used by the client process to connect to the server address through `3-way tcp handshake`.

#### Write

Send bytes through the socket making at kernel level the specific call for sending data through the network.
If the message is too large to live in a single packet (64K) it is decomposed in multiple packets in a way that is transparent to the user.
At kernel level data are appended to a sending buffer and then sent onto the network interface.

#### Read

Read bytes through the socket making at kernel level the specific calls for receiving data through the network, sending acks for received packets and copying received data from the socket buffer.

#### Close

Close or aborts any pending connection on the socket.
At kernel level first check if the socket is in lisenting mode, if so traverse the queue and for each pendinc connection abort it.


# Syscalls in GO

The syscall package provides a mean to interact with the underlying operating system directly.
However, it is essential to note that the syscall package is platform-specific, and some functions might not be available or behave differently across different platforms.

For what regards at least the TCP calls, they are implemented in the syscall package under the hood with the use of the `libc` libraries.
`libc` is responsible for the mechanism of system calls to the OS implemented in low level code.

For our purposes we will use the procedures:

* `func syscall.Socket(domain int, typ int, proto int) (fd int, err error)`
* `func syscall.Bind(fd int, sa syscall.Sockaddr) (err error)`
* `func syscall.Listen(s int, backlog int) (err error)`
* `func syscall.Accept(fd int) (nfd int, sa syscall.Sockaddr, err error)`
* `func syscall.Connect(fd int, sa syscall.Sockaddr) (err error)`
* `func syscall.Write(fd int, p []byte) (n int, err error)`
* `func syscall.Read(fd int, p []byte) (n int, err error)`
* `func syscall.Close(fd int) (err error)`

In particular the backlog parameter of the Listen procedure indicates the number of connection that can be put in the queue before beign ingored.

For all the procedures `fd` indicates a file descriptor.

For the Socket procedure, `domain` indicates the address family, `type` the type of socket(TCP or UDP), and `proto` the underlaing protocol level IP.

Based on the platform you are using you can receive an error in case the size of the  `p` parameter in  `Write` procedure is too big.

To note that the TCP protocol is stream based, in this way there could not be an exact one to one corrispondence between the content of the `Write` and of the `Read` calls.

# Example code

Here some example code using the previous described syscalls to send an hello message from one socket to the other.

#### Client

```
fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, syscall.IPPROTO_IP)
if err != nil {
	panic(err)
}

err = syscall.Bind(fd, &syscall.SockaddrInet4{Port: 8081, Addr: [4]byte{127, 0, 0, 1}})
if err != nil {
	panic(err)
}

err = syscall.Connect(fd, &syscall.SockaddrInet4{Port: 8080, Addr: [4]byte{127, 0, 0, 1}})
if err != nil {
	panic(err)
}

size, err := syscall.Write(fd, []byte("hello "))
if err != nil {
	panic(err)
}
println("written bytes: ", size)

size, err = syscall.Write(fd, []byte("world!"))
if err != nil {
	panic(err)
}
println("written bytes: ", size)

err = syscall.Close(fd)
if err != nil {
	panic(err)
}
```

#### Server

```
fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, syscall.IPPROTO_IP)
if err != nil {
	panic(err)
}

err = syscall.Bind(fd, &syscall.SockaddrInet4{Port: 8080, Addr: [4]byte{127, 0, 0, 1}})
if err != nil {
	panic(err)
}

err = syscall.Listen(fd, 1)
if err != nil {
	panic(err)
}

nfd, _, err := syscall.Accept(fd)
if err != nil {
	panic(err)
}

buf := make([]byte, 4)
size := 1
totSize := 0
for size > 0 {
	size, err = syscall.Read(nfd, buf)
	if err != nil {
		panic(err)
	}

	fmt.Println(string(buf[0:size]))
	buf = make([]byte, 4)
	totSize = totSize + size
}

fmt.Println()
fmt.Println("Total size: ", totSize)

err = syscall.Close(fd)
if err != nil {
	panic(err)
}
```

# Summary

We have seen how to send and receive data over TCP using directly system calls, using the methods of the go's syscall package (that are implemented using the `libc` libraries of the OS provider).
We have tried to give some more awareness in the use of the TCP stack.
Hope you enjoied the article, if you want to either suggest some improvements or have an exhange please contact via `email` or via my `Twitter` profile. 

References for this article:
* [IBM TCP syscalls][ibm-blog-tcp-syscalls]

[ibm-blog-tcp-syscalls]: https://developer.ibm.com/articles/au-tcpsystemcalls