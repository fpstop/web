+++
categories = ["redis"]
date = "2017-03-12T19:31:50+08:00"
title = "windows编译redis"

+++
假设大家都已经了解redis了，本文只讲redis的windows上的编译。
<!--more-->


# 1.windows编译redis
## 1.1下载
微软维护了一个windows版本，地址在[https://github.com/MSOpenTech/redis](https://github.com/MSOpenTech/redis)

>The Redis project does not officially support Windows. 
>However, the Microsoft Open Tech group develops and maintains this Windows port targeting Win64. 

redis官方不支持windows编译，但是微软维护了一个windows版本。既然这个出现在redis官网上，想必也是认可的。最起码用来研究学习时没问题的。

## 1.2 编译
可以用vs2015 直接打开工程文件
- server："\redis-2.8_win\msvs\RedisServer.sln"
- Hiredis异步例子："\redis-2.8_win\msvs\HiredisExample\HiredisExample.sln"

server 很顺利的编译通过，但是Hiredis异步例子编译时报了一个错误,是个类型重定义错误。

	>\src\win32_interop\win32_types.h(37): error C2371: 'off_t': redefinition; different basic types
	>c:\program files (x86)\windows kits\10\include\10.0.10240.0\ucrt\sys\types.h(42): note: see declaration of 'off_t'
	
很明显，是自己定义的类型和默认的类型重复了。打开 win32_types.h 文件看看：

	/* The Posix version of Redis defines off_t as 64-bit integers, so we do the same.
	 * On Windows, these types are defined as 32-bit in sys/types.h under and #ifndef _OFF_T_DEFINED
	 * So we define _OFF_T_DEFINED at the project level, to make sure that that definition is never included.
	 * If you get an error about re-definition, make sure to include this file before sys/types.h, or any other
	 * file that include it (eg wchar.h).
	 * _off_t is also defined #ifndef _OFF_T_DEFINED, so we need to define it here.
	 * It is used by the CRT internally (but not by Redis), so we leave it as 32-bit.
	 */
	 
原来，微软团队发现redis在Posix体系下，off_t被定义成64位，而在windows下被sys\types.h文件定义成32位。

sys\types.h

	#ifndef _OFF_T_DEFINED
		#define _OFF_T_DEFINED

		typedef long _off_t; // file offset value

		#if !__STDC__
			typedef _off_t off_t;
		#endif
	#endif

然后windows团队就在工程属性里定义了_OFF_T_DEFINED （So we define _OFF_T_DEFINED at the project level），使32位的不生效，用自己定义在文件的，但是为什么还是出现重定义了呢？

因为他们忘记在工程属性里定义啦，加回来就行拉。。。忘记定义了这个宏，所以默认的就生效了，自己也定义一份，当然编不过了。。