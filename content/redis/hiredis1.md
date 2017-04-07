+++
categories = ["redis"]
date = "2017-03-26T00:03:10+08:00"
title = "redis 使用-hiredis库使用（一） 基本篇 看完本文就可以上手工作了"

+++
hiredis 是redis的客户端sdk，可以让程序操作redis。本文先讲建立连接，基本的get/set命令，读写二进制，获取多个结果来讲。假设读者已经了解redis命令了。
<!--more-->

hiredis的代码也包含在redis代码中，redis\deps\hiredis目录下，接口很简单，几乎不用封装就可以用。

# 1 连接redis数据库
## 1.1 无超时时间，阻塞
	redisContext *redisConnect(const char *ip, int port); 
## 1.2 设置超时时间，阻塞
	redisContext *redisConnectWithTimeout(const char *ip, int port, const struct timeval tv);  

## 1.3 非阻塞，立刻返回，也就无所谓超时
	redisContext *redisConnectNonBlock(const char *ip, int port);

# 2 执行命令
	void *redisCommand(redisContext *c, const char *format, ...);
	
## 2.1 返回值
返回值是redisReply类型的指针：

	/* This is the reply object returned by redisCommand() */
	typedef struct redisReply {
		int type; /* REDIS_REPLY_* */
		PORT_LONGLONG integer; /* The integer when type is REDIS_REPLY_INTEGER */
		int len; /* Length of string */
		char *str; /* Used for both REDIS_REPLY_ERROR and REDIS_REPLY_STRING */
		size_t elements; /* number of elements, for REDIS_REPLY_ARRAY */
		struct redisReply **element; /* elements vector for REDIS_REPLY_ARRAY */
	} redisReply;
	
## 2.2 返回值类型
type字段，包含以下几种类型：
	
	#define REDIS_REPLY_STRING 1	//字符串
	#define REDIS_REPLY_ARRAY 2		//数组，例如mget返回值
	#define REDIS_REPLY_INTEGER 3	//数字类型
	#define REDIS_REPLY_NIL 4		//空
	#define REDIS_REPLY_STATUS 5	//状态，例如set成功返回的‘OK’
	#define REDIS_REPLY_ERROR 6		//执行失败
	
# 3 基本命令，get，set
## 3.1 set
	redisReply *r1 = (redisReply*)redisCommand(c, "set k v");
	
结果：
	type = 5

	len = 2

	str = OK

返回的类型5，是状态。str是OK，代表执行成功。
	
## 3.2 get
	redisReply *r2 = (redisReply*)redisCommand(c, "get k");
	
结果：
	type = 1

	len = 1

	str = v
	
返回的类型是1，字符串类型，str是‘v’ ，刚才保存的。

	
# 4 存取二进制

	char sz[] = { 0,1,2,3,0 };
	redisReply *r3 = (redisReply*)redisCommand(c, "set kb %b",sz,sizeof(sz));
	
存二进制的时候，使用%b，后面需要对应两个参数，指针和长度。

	redisReply *r4 = (redisReply*)redisCommand(c, "get kb");
	
读取二进制的时候，和普通是一样的，str字段是地址，len字段是长度。
	
# 5 存取多个值

## 存多个值
拼接字符串就好啦

	redisReply *r5 = (redisReply*)redisCommand(c, "mset k1 v1 k2 v2");
	
## 取多个值

	redisReply *r6 = (redisReply*)redisCommand(c, "mget k1 k2");
	
这里主要看返回值里的这两个字段，代表返回值个数和起始地址：

    size_t elements; /* number of elements, for REDIS_REPLY_ARRAY */
    struct redisReply **element; /* elements vector for REDIS_REPLY_ARRAY */
	
	
知道了以上知识，基本可以上手干活了，redis的接口还是很不错的，感觉都不用封装了。