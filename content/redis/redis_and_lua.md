+++
categories = ["redis"]
date = "2017-05-21T23:59:24+08:00"
title = "redis 使用lua脚本提高执行效率"

+++
主要针对使用redis的同步api。
<!--more-->
# redis的同步api，代价非常昂贵
redis的同步api，每一次调用都是相当于一次网络通信，而且时阻塞的，如果赶上网络波动或者redis正在保存数据，会占用很多时间的。
	
	void *redisvCommand(redisContext *c, const char *format, va_list ap)
	{
		send request...
		recv result...
		return ...;
	}


# 2 适应情景分析
我们使用时，有很多类似下面的操作

	val1 = redis.command("get key1");
	if(val1 > 10)
		redis.command("set val1 100");
	else
		redis.command("set val1 0");

简单地说，就是**多次redis api调用，后面的操作以来于前面的执行结果**。

# 3 使用lua脚本优化

redis server支持lua脚本，使用时相当于客户端把一小段lua脚本发到redis server执行，这个lua脚本可能会有多次操作。

但是网络通信只有一次，redis server执行完lua脚本把结果返回给客户端，减少网络交互次数。

