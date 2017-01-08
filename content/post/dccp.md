+++
title = "DCCP: The socket type you probably never heard of"
draft = false
date = "2016-12-13T23:10:50+05:30"
tags = ['tech', 'linux']
+++

*TL;DR: DCCP is a relatively newer transport layer protocol which draws from both TCP and UDP. Jump straight to the [example C code]({{< relref "#example-in-c" >}}).*


Background
--

Historically, the majority of the traffic on the Internet has been over [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) which provides a reliable connection-oriented stream between two hosts. [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol) has been mainly used by applications whose brief transfers would be unacceptably slowed by TCP's connection establishment overhead or those for which timeliness is more important than reliability. However, the increasing use of UDP for applications such as internet telephony and streaming media which transfer a large amount of data can lead to significant [network congestion](https://en.wikipedia.org/wiki/Network_congestion). Since unlike TCP, UDP provides no inherent congestion control mechanism, an application can send UDP datagrams at a much higher rate than the available path capacity and cause congestion along the path. Increased congestion may lead to delays, packet loss and the degradation of the network's quality of service.

Applications and protocols that choose to use UDP as their transport must, therefore, employ mechanisms to prevent congestion and to establish some degree of fairness with concurrent traffic so that the network remains usable. A prominent example of such a congestion control scheme is [LEDBAT](https://en.wikipedia.org/wiki/LEDBAT) employed by [BitTorrent](https://en.wikipedia.org/wiki/BitTorrent). However, implementing a congestion control scheme is difficult, time-consuming and error-prone. Multiple non-standard implementations also make it difficult to reason about how applications would respond to network congestion. [DCCP](https://en.wikipedia.org/wiki/Datagram_Congestion_Control_Protocol) - Datagram Congestion Control Protocol is intended to mitigate this problem as a transport for unreliable datagrams with built-in congestion control.

From an application programmer's perspective, DCCP differs from UDP by providing four additional features:

- Explicit connection establishment between hosts
- Selectable congestion control schemes
- Path MTU discovery to avoid fragmentation
- Service Codes for identifying applications

DCCP makes use of Explicit Congestion Notification but it is transparent the  application. DCCP is designed to leave additional functionality such as reliability or Forward Error Correction (FEC) to be layered on top, as and when required rather than at the protocol level itself.

Explicit connection establishment
--

The connection establishment semantics of DCCP mirror those of TCP with a client that actively connects to a server that is passively listening on a port. DCCP connections are bidirectional. Logically, however, a DCCP connection consists of two separate unidirectional connections, called half-connections. Each half-connection is a one-way, unreliable datagram pipe. The rationale for this explained in the next section.


Selectable congestion control schemes
--

TCP implements congestion control entirely transparently to the application. While it is possible to configure the host to use a specific variant, there is no way for the application to discover which congestion control scheme is in force, let alone negotiate one. DCCP, however, can cater to the different needs of applications by allowing applications to negotiate the congestion control schemes. In fact, each of the half-connections can use a different scheme, allowing for greater control.

Congesting the network by sending data at a rate that is faster than the slowest link between the endpoints will overwhelm it. This may lead to packet loss leading to retransmissions which may, in turn, lead to further congestion. The solution to this problem is to start transmitting data at a slow rate on a new connection and to then ramp up the speed until packet loss is detected. The transmission rate may then be scaled back until no further packet loss occurs. The optimum speed at which to transfer data  will change with network conditions over the life of the connection. Congestion control schemes differ in how packet loss is estimated and the rate at which is the transmission speed is ramped up or scaled back. DCCP congestion control schemes are denoted by Congestion Control Identifiers - CCIDs. Currently, three CCIDs have been formally specified:

- **[CCID 2](https://tools.ietf.org/html/rfc4341) -  TCP-like Congestion Control:** A quick reacting scheme modelled after TCP which will rapidly ramp up speed to take advantage of available bandwidth and also rapidly scale back when congestion is detected. Suitable for applications that can handle large swings in transmission rates.

- **[CCID 3](https://tools.ietf.org/html/rfc5348) - TCP-Friendly Rate Control (TFRC):** A slower reacting scheme intended to be friendly to concurrent TCP flows in the network. Provides a relatively smoother sending rate at the expense of possibly not utilising all available bandwidth. Suitable for media streaming applications that prefer to minimise abrupt changes in the sending rate.

- **[CCID 4](https://tools.ietf.org/html/rfc4828) - TCP-Friendly Rate Control for Small Packets (TFRC-SP):** An experimental scheme for applications that use a small datagram size and those that change their sending rate by varying the datagram size.

In addition, the Linux kernel's [DCCP Test Tree](https://github.com/uoaerg/linux-dccp) contains an experimental implementation of a scheme modelled after [TCP CUBIC](https://en.wikipedia.org/wiki/CUBIC_TCP). There is also a mode that disables congestion control altogether for *UDP-like* behaviour. 

PMTU discovery
--

Data between two internet hosts is transferred transmitted as a series of IP packets that pass through intermediate links. Each of these links has a maximum packet size or maximum transmission unit (MTU) that it can transmit without having to break it up into smaller fragments. The largest packet size that does not require fragmentation anywhere along a path is referred to as the path maximum transmission unit or PMTU. Applications can usually get better error tolerance by producing packets smaller than the PMTU. DCCP defines a maximum packet size (MPS) based on the PMTU and the congestion control scheme used for each connection. DCCP implementations will not send any packet bigger than the MPS and instead return an appropriate error to the application. The application can query the DCCP stack for the current MPS and restrict itself from sending datagrams larger than this value and thereby avoid [fragmentation](https://en.wikipedia.org/wiki/IP_fragmentation).


Service Codes
--

DCCP defines a 32 bit Service Code to disambiguate between multiple applications associated with a single a server port. The client specifies the Service Code it wants to connect to and this is used to identify the intended service or application to process a DCCP connection request. Essentially, Service Codes provide an additional level of indirection for connection multiplexing. A server listening on a port may be associated with multiple Service Codes but a client may have only one Service Code, indicating the application it wishes to connect to.

Usage
--
The mainline Linux kernel has included DCCP support since [2.6.14](https://lwn.net/Articles/149756/) and mainstream distributions like Ubuntu enable it by default. However, to get the newer experimental features, you will have to build the kernel from the DCCP Test Tree. Or you can also grab the latest stable kernel release merged with the experimental DCCP changes from [here](https://github.com/unmole/linux-dccp/releases/latest). Be sure to enable all the CCIDs in the kernel configuration in *Networking Support* --> *Networking Options* --> *The DCCP Protocol* --> *DCCP CCIDs Configuration*. Like the Debian Installation Guide Says, "*Don't be afraid to try compiling the kernel. It's fun and profitable.*" For now, Linux is the only operating system supporting native DCCP, unless you count the patch for an ancient version of FreeBSD.

Example in C
--

The server and client look almost exactly the same as their TCP counterparts with the exception fo the socket type and setting of the service code. The client uses *getsockopt()* to read the current maximum packet size. Reading the available CCIDs on the host is shown in **probe.c**. As libc doesn't still have a **netinet/dccp.h** header, you will have to get the required constants from the kernel sources or directly use the **dccp.h** header below. [Download Code](/dl/dccp_socket_example.tar.gz)

**server.c**
``` C
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <errno.h>

#include "dccp.h"

#define PORT 1337
#define SERVICE_CODE 42

int error_exit(const char *str)
{
	perror(str);
	exit(errno);
}

int main(int argc, char **argv)
{
	int listen_sock = socket(AF_INET, SOCK_DCCP, IPPROTO_DCCP);
	if (listen_sock < 0)
		error_exit("socket");

	struct sockaddr_in servaddr = {
		.sin_family = AF_INET,
		.sin_addr.s_addr = htonl(INADDR_ANY),
		.sin_port = htons(PORT),
	};

	if (setsockopt(listen_sock, SOL_SOCKET, SO_REUSEADDR, &(int) {
		       1}, sizeof(int)))
		error_exit("setsockopt(SO_REUSEADDR)");

	if (bind(listen_sock, (struct sockaddr *)&servaddr, sizeof(servaddr)))
		error_exit("bind");

	// DCCP mandates the use of a 'Service Code' in addition the port
	if (setsockopt(listen_sock, SOL_DCCP, DCCP_SOCKOPT_SERVICE, &(int) {
		       htonl(SERVICE_CODE)}, sizeof(int)))
		error_exit("setsockopt(DCCP_SOCKOPT_SERVICE)");

	if (listen(listen_sock, 1))
		error_exit("listen");

	for (;;) {

		printf("Waiting for connection...\n");

		struct sockaddr_in client_addr;
		socklen_t addr_len = sizeof(client_addr);

		int conn_sock = accept(listen_sock, (struct sockaddr *)&client_addr, &addr_len);
		if (conn_sock < 0) {
			perror("accept");
			continue;
		}

		printf("Connection received from %s:%d\n",
		       inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));

		for (;;) {
			char buffer[1024];
			// Each recv() will read only one individual message.
			// Datagrams, not a stream!
			int ret = recv(conn_sock, buffer, sizeof(buffer), 0);
			if (ret > 0)
				printf("Received: %s\n", buffer);
			else
				break;

		}

		close(conn_sock);
	}
}
```

**client.c**
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <errno.h>

#include "dccp.h"

int error_exit(const char *str)
{
	perror(str);
	exit(errno);
}

int main(int argc, char *argv[])
{
	if (argc < 5) {
		printf("Usage: ./client <server address> <port> <service code> <message 1> [message 2] ... \n");
		exit(-1);
	}
	struct sockaddr_in server_addr = {
		.sin_family = AF_INET,
		.sin_port = htons(atoi(argv[2])),
	};

	if (!inet_pton(AF_INET, argv[1], &server_addr.sin_addr.s_addr)) {
		printf("Invalid address %s\n", argv[1]);
		exit(-1);
	}

	int socket_fd = socket(AF_INET, SOCK_DCCP, IPPROTO_DCCP);
	if (socket_fd < 0)
		error_exit("socket");

	if (setsockopt(socket_fd, SOL_DCCP, DCCP_SOCKOPT_SERVICE, &(int) {htonl(atoi(argv[3]))}, sizeof(int)))
		error_exit("setsockopt(DCCP_SOCKOPT_SERVICE)");

	if (connect(socket_fd, (struct sockaddr *) &server_addr, sizeof(server_addr)))
		error_exit("connect");

	// Get the maximum packet size
	uint32_t mps;
	socklen_t res_len = sizeof(mps);
	if (getsockopt(socket_fd, SOL_DCCP, DCCP_SOCKOPT_GET_CUR_MPS, &mps, &res_len))
		error_exit("getsockopt(DCCP_SOCKOPT_GET_CUR_MPS)");
	printf("Maximum Packet Size: %d\n", mps);

	for (int i = 4; i < argc; i++) {
		if (send(socket_fd, argv[i], strlen(argv[i]) + 1, 0) < 0)
			error_exit("send");
	}

	// Wait for a while to allow all the messages to be transmitted
	usleep(5 * 1000);

	close(socket_fd);
	return 0;
}

```
**probe.c**
```C
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>

#include "dccp.h"

int main()
{
	int sock_fd = socket(AF_INET, SOCK_DCCP, IPPROTO_DCCP);

	// Check the congestion control schemes available
	socklen_t res_len = 6;
	uint8_t ccids[6];
	if (getsockopt(sock_fd, SOL_DCCP, DCCP_SOCKOPT_AVAILABLE_CCIDS, ccids, &res_len)) {
		perror("getsockopt(DCCP_SOCKOPT_AVAILABLE_CCIDS)");
		return -1;
	}

	printf("%d CCIDs available:", res_len);
	for (int i = 0; i < res_len; i++)
		printf(" %d", ccids[i]);

	return res_len;
}
```

**dccp.h**
```C
/* This file only contains constants necessary for user space to call
 * into the kernel and thus, contains no copyrightable information. */

#ifndef DCCP_DCCP_H
#define DCCP_DCCP_H

// From the kernel's include/linux/socket.h
#define SOL_DCCP                        269

// From kernel's include/uapi/linux/dccp.h
#define DCCP_SOCKOPT_SERVICE            2
#define DCCP_SOCKOPT_CHANGE_L           3
#define DCCP_SOCKOPT_CHANGE_R           4
#define DCCP_SOCKOPT_GET_CUR_MPS        5
#define DCCP_SOCKOPT_SERVER_TIMEWAIT    6
#define DCCP_SOCKOPT_SEND_CSCOV         10
#define DCCP_SOCKOPT_RECV_CSCOV         11
#define DCCP_SOCKOPT_AVAILABLE_CCIDS    12
#define DCCP_SOCKOPT_CCID               13
#define DCCP_SOCKOPT_TX_CCID            14
#define DCCP_SOCKOPT_RX_CCID            15
#define DCCP_SOCKOPT_QPOLICY_ID         16
#define DCCP_SOCKOPT_QPOLICY_TXQLEN     17
#define DCCP_SOCKOPT_CCID_RX_INFO       128
#define DCCP_SOCKOPT_CCID_TX_INFO       192

#endif //DCCP_DCCP_H

```

Caveats and Conclusion
--

DCCP is not mainstream. It is not widely deployed or even supported. Documentation is sparse. Although Linux DCCP NAT is functional, many intermediate boxes will probably just drop DCCP traffic. DCCP is the Fixed-gear bicycle of Layer 4, it is the ultimate hipster transport.











