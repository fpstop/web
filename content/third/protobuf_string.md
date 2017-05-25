+++
categories = ["protobuf"]
date = "2017-05-22T18:23:27+08:00"
title = "protobuf的DebugString方法，汉字乱码"

+++
本文不是解决乱码问题，是要提取乱码内容。游戏上线后要做舆情分析，想收集前两日玩家游戏内聊天内容，无奈没有专门的日志，只能从网络库日志提取protobuf的内容。
<!--more-->

网络库会把每条收到的protobuf协议都调用方法，日志打印所有字段的值，但是汉子会打出类似"\346\234\211\344\272\272\345\220\227".想把这些编码转化为能看懂的汉子。

# 1 尝试编码转换
游戏内的聊天文字用的是utf8的格式，不过看着上面的编码有超过256 char最大值的，看着像双字节的unicode，转成unicode后显示也是乱码，后来用utf8，ansi看都是乱码，只好从代码分析是如何打印的了。

# 2 分析DebugString函数

最终会调用到下面的函数，原来是八进制的，仔细一看，确实是。所以剩下的就好做了。

	// Escapes 'src' using C-style escape sequences, and appends the escaped string
	// to 'dest'. This version is faster than calling CEscapeInternal as it computes
	// the required space using a lookup table, and also does not do any special
	// handling for Hex or UTF-8 characters.
	// ----------------------------------------------------------------------
	void CEscapeAndAppend(StringPiece src, string* dest) {
	  size_t escaped_len = CEscapedLength(src);
	  if (escaped_len == src.size()) {
		dest->append(src.data(), src.size());
		return;
	  }

	  size_t cur_dest_len = dest->size();
	  dest->resize(cur_dest_len + escaped_len);
	  char* append_ptr = &(*dest)[cur_dest_len];

	  for (int i = 0; i < src.size(); ++i) {
		unsigned char c = static_cast<unsigned char>(src[i]);
		switch (c) {
		  case '\n': *append_ptr++ = '\\'; *append_ptr++ = 'n'; break;
		  case '\r': *append_ptr++ = '\\'; *append_ptr++ = 'r'; break;
		  case '\t': *append_ptr++ = '\\'; *append_ptr++ = 't'; break;
		  case '\"': *append_ptr++ = '\\'; *append_ptr++ = '\"'; break;
		  case '\'': *append_ptr++ = '\\'; *append_ptr++ = '\''; break;
		  case '\\': *append_ptr++ = '\\'; *append_ptr++ = '\\'; break;
		  default:
			if (!isprint(c)) {
			  *append_ptr++ = '\\';
			  *append_ptr++ = '0' + c / 64;
			  *append_ptr++ = '0' + (c % 64) / 8;
			  *append_ptr++ = '0' + c % 8;
			} else {
			  *append_ptr++ = c;
			}
			break;
		}
	  }
	}


	
# 3 提取数据，输出

第一步先用c++11 的正则表达式提取数据，然后再用strtol 将八进制转化输出即可，代码如下：

	#include "stdafx.h"
	#include "../../../Com/CFunc.hpp"
	#include "../../../Com/CStr.hpp"
	#include "../../../Com/CFile.hpp"
	#include <regex>
	using namespace std;



	int main()
	{
		linevec_t lines;
		XReadFileLine("chat.csv", lines);

		std::regex reg("\\\\[0-7]{3}", std::regex_constants::ECMAScript);

		sregex_token_iterator end;
		for (auto s:lines)
		{
			string chat;
			sregex_token_iterator it(s.begin(), s.end(), reg);
			while (it != end)
			{
				//cout << *it<<" ";
				string ss = *it;
				++it;
				
				char sz[4] = {0};
				sz[0] = ss[1];
				sz[1] = ss[2];
				sz[2] = ss[3];
				//cout << sz;

				int ret = strtol(sz, NULL, 8);
				chat += char(ret);
			}
			cout << chat << endl;
		}
		return 0;
	}
