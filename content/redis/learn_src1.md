+++
categories = ["网络编程"]
date = "2017-03-24T21:47:04+08:00"
title = "socket的属性设置 （持续更新）"

+++
今天看redis代码时，发现了如下代码，设置阻塞socket的读写超时时间，仔细一看就是简单的设置了一下socket的属性，索性把socket一些属性总结一下。

	/* Set read/write timeout on a blocking socket. */
	int redisSetTimeout(redisContext *c, const struct timeval tv)
<!--more-->
## 1 读超时

>The timeout, in milliseconds, for blocking receive calls. The default for this option is zero, which indicates that a receive operation will not time out. If a blocking receive call times out, the connection is in an indeterminate state and should be closed.
	注意，linux和windows的参数略不同。
	
		bool SetRecvTimeOut(uint32 millisecond)
		{
	#ifdef WIN32
			DWORD time = millisecond;
			if (setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (const char*)&time, sizeof(DWORD)) == -1) {
				return false;
			}
	#else
			struct timeval tv;
			tv.tv_sec = millisecond / 1000;
			tv.tv_usec = (millisecond % 1000) * 1000;
			if (setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, (const char*)&tv, sizeof(tv)) == -1) {
				return false;
			}
	#endif
			return true;
		}
## 2 写超时

>The timeout, in milliseconds, for blocking send calls. The default for this option is zero, which indicates that a send operation will not time out. If a blocking send call times out, the connection is in an indeterminate state and should be closed.

	注意，linux和windows的参数略不同。

		bool SetSendTimeOut(uint32 millisecond)
		{
	#ifdef WIN32
			DWORD time = millisecond;
			if (setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, (const char*)&time, sizeof(DWORD)) == -1) {
				return false;
			}
	#else
			struct timeval tv;
			tv.tv_sec = millisecond / 1000;
			tv.tv_usec = (millisecond % 1000) * 1000;
			if (setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, (const char*)&tv, sizeof(tv)) == -1) {
				return false;
			}
	#endif
			return true;
		}
	
## 3 地址重用

>Allows a socket to bind to an address and port already in use. The SO_EXCLUSIVEADDRUSE option can prevent this.

一个端口释放后会等待两分钟之后才能再被使用，SO_REUSEADDR是让端口释放后立即就可以被再次使用。

p2p打洞时也需要设置这个属性。

公司的网络库貌似都会设置这个属性呢。

	bool SetReUseAddr(int v)
	{
		int ret = setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, (const char*)&v, sizeof(v));
		return 0 == ret;
	}
	
## 4 keepalive
	
设置心跳包，还可以指定心跳包频率，不过建议还是在逻辑层设计心跳协议，来检查连接存活。

	int val = 1;
	setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &val, sizeof(val))
	
## 5 TCP_NODELAY

TCP_NODELAY和TCP_CORK基本上控制了包的“Nagle化”，这里我们主要讲TCP_NODELAY.Nagle化在这里的含义是采用Nagle算法把较小的包组装为更大的帧。JohnNagle是Nagle算法的发明人，后者就是用他的名字来命名的，他在1984年首次用这种方法来尝试解决福特汽车公司的网络拥塞问题（欲了解详情请参看IETF RFC 896）。他解决的问题就是所谓的silly window syndrome，中文称“愚蠢窗口症候群”，具体含义是，因为普遍终端应用程序每产生一次击键操作就会发送一个包，而典型情况下一个包会拥有一个字节的数据载荷以及40个字节长的包头，于是产生4000%的过载，很轻易地就能令网络发生拥塞,。Nagle化后来成了一种标准并且立即在因特网上得以实现。它现在已经成为缺省配置了，但在我们看来，有些场合下把这一选项关掉也是合乎需要的。 

现在让我们假设某个应用程序发出了一个请求，希望发送小块数据，比如sns游戏中的点击确定按钮。我们可以选择立即发送数据或者等待产生更多的数据然后再一次发送两种策略。如果我们马上发送数据，那么交互性的以及客户/服务器型的应用程序将极大地受益。例如，当我们正在发送一个较短的请求并且等候较大的响应时，相关过载与传输的数据总量相比就会比较低，而且，如果请求立即发出那么响应时间也会快一些。以上操作可以通过设置套接字的TCP_NODELAY选项来完成，这样就禁用了Nagle算法，在nginx中设置tcp_nodelay on,注意放在http标签里。
	  

**总之这种把小包组成大包的操作应该由逻辑层来做**

	bool SetNoDelay()
	{
		int yes = 1;

		if (setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof(yes)) == -1) {
			return false;
		}
		return true;
	}


## 6 阻塞非阻塞

这个属性应该是最重要最常用的了。

		//1 :non block
		int SetNonBlock(int value)
		{
	#ifndef WIN32
			int oldflags = ::fcntl(sock, F_GETFL, 0);
			/* If reading the flags failed, return error indication now. */
			if (oldflags == -1)
				return -1;

			/* Set just the flag we want to set. */
			if (value != 0)
				oldflags |= O_NONBLOCK;
			else
				oldflags &= ~O_NONBLOCK;
			/* Store modified flag word in the descriptor. */
			return ::fcntl(m_iSock, F_SETFL, oldflags);
	#else
			if (::ioctlsocket(sock, FIONBIO, (u_long FAR*)&value) == SOCKET_ERROR)
			{
				return -1;
			}

			return 0;
	#endif
		}
	
	
上文中的例子[XSock.hpp](https://github.com/xjp342023125/Code/blob/master/common/XSock.hpp)