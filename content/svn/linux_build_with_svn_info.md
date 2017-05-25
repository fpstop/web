+++
categories = ["持续集成"]
date = "2017-05-25T05:54:05+08:00"
title = "linux编译版本时日志中打印svn版本号"

+++
做游戏开发，有很多时候查bug，最后都是发现时因为测试没有替补丁。所以想，如果日志里能把svn当前的版本号打印出来多好，先排除版本问题。
<!--more-->

从svn中提取信息貌似很不容易，后来想到一种取巧的方法：

- 编译脚本中，调用 svn info命令，把svn输出的一个文件，命名为 svn_info.h。
- 代码包含 svn_info.h,日志中输出svn版本号。

脚本提取时，用grep过滤一下信息：

	DST=./Common/svn_info.h
	rm -rf DST
	echo '#ifndef _SVN_INFO_' > $DST
	echo '#define _SVN_INFO_' >> $DST

	VERSION=`svn info|grep Revision`
	echo '  #define SVNINFO "'$VERSION' '$1'" ' >> $DST

	echo '#endif' >> $DST
	#  
	echo $VERSION
	
然后就会生成类似如下的文件：

	#ifndef _SVN_INFO_
	#define _SVN_INFO_
	#define SVNINFO  "Revision: 11111"
	#endif
	
很简单吧。