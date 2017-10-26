---
title: 敏感词的字典树匹配（lua版）
date: 2016-8-1
tags:
- 敏感词
- 字典树
- Lua
categories: Unity
---

对于国内互联网和出版物来说，屏蔽敏感词和某些众所周知的秘密是一件老生常谈加司空见惯的事情了。。。上周小白也做了一个这个功能，但是我们属于游戏，要屏蔽的东西十分简单，不用像那些大型网站或者平台一样用专门的算法进行匹配，所以就能省则省。。。但是还是想说蛋疼的模式匹配啊
## 几种匹配方式对比
### 普通匹配法
该方法就是直接进行字符匹配，遍历所有的敏感词列表看看用户的输入中是否有敏感词出现，这种对敏感词少且输入短的来说是无所谓，但是真正的应用，我只能说：呵呵。。。
### 正则匹配
我也觉得正则匹配用到此处刚刚好，完全可以担当灵活多变四个字。但是如果是匹配有某些规律的还好说，可敏感词我还真找不出来他都是什么规律，想了想，无奈的放弃吧，当断则断
### 字典树
从运营处拿到了两份敏感词，一份是名字，一份是聊天，其中名字有一万行，聊天也特么有一万多行。使用过普通匹配后，猛喷出一口老血，这酸爽。。。无奈，使用了字典树，具体步骤是：a，预先遍历敏感词，构造字典树；b，根据输入匹配。貌似说了一堆废话。。。（其实我也不想，是现在闲了）。
## 字典树实现
下面直接上代码吧
``` csharp
local chat = require "chat"

local chat_dict
local chat_leaves = {}
--构造字典树
local function init_chat_dict()
	chat_dict = {}
	local record = chat.table
	
	for i = 1, #record do
		local word = record[i]
		local t = chat_dict
		for j = 1, #word do 
			local c = string.byte(word, j)
			if not t[c] then
				t[c] = {}
			end
			t = t[c]
		end			
		chat_leaves[word] = true
	end
end

--匹配
function ShieldChat(msg)
	if not chat_dict then
		init_chat_dict()
	end
	local matchs = {}
	for i = 1, #msg do
		local p = i
		local q = p
		local t = chat_dict
		while true do
			if not t[string.byte(msg,q)] then
				q = q - 1
				break
			end
			t = t[string.byte(msg, q)]
			q = q + 1
		end
		if q >= p then
			local str = string.sub(msg, p, q)
			if chat_leaves[str] then
				table.insert(matchs, {b = p, e = q, l = (q - p + 1)})
			end
		end
	end
	local str = msg
	for _,v in ipairs(matchs) do
		str = string.sub(str, 1, v.b - 1) .. string.rep("*", v.l) .. string.sub(str, v.e + 1)
	end
	return str
end

```
以上包含最基本功能，剩下的想加可以自行添加其他要求，并希望能得到大家的其他指导！