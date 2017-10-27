---
title: Hexo
date: 2017-10-20
tags:
- hexo
- git
- github
categories: 基础操作
---

### 原博客搬迁
原本一直使用[CSDN的博客](http://blog.csdn.net/cheng624)记录一些自己的东西，但是前段时间登录发现一直需要我真实身份验证，抱着支持国家政策的态度我屈服了... 但是最近，后来为什么又出来一个必须扫描公众号二维码关注再完成手机验证。只想说你推广也就算了为毛还要强制！表示不服！
为了专治“不服”，我决定使用万能Github自行搭建博客，考虑了 [Jekyll](http://jekyll.com.cn/)，看了比人的一堆分析以后最终选择了 [Hexo](https://hexo.io/zh-cn/docs/index.html)，由于我的web前端只停留在html+css阶段故表示不懂他们底下的大区别，但是使用了以后只觉得 hexo很麻烦... 因为jekyll是github自动生成pages，hexo需要自己genegrate+deploy，我不知道这是不是代表我每次上传都必须有nodejs的环境，也不知道jekyll是不是这样...

最终，我还是选择了hexo，因为我看用到了 [Material](https://github.com/viosey/hexo-theme-material/) 的模板0.0

### 安装hexo
安装步骤很简单，一路文档操作：[连接](https://hexo.io/zh-cn/docs/index.html)
安装 Hexo 相当简单。然而在安装前，您必须检查电脑中是否已安装下列应用程序：

	Node.js
	Git
如果您的电脑中已经安装上述必备程序，那么恭喜您！接下来只需要使用 npm 即可完成 Hexo 的安装。
``` bash
$ npm install -g hexo-cli
```

### 使用Material主题

#### 建立一个空的站点
选择一个心仪的文件夹，在此次cmd
``` bash
$ hexo init MyBlog #MyBlog是我要放置博客文件的文件夹
$ cd MyBlog
$ npm install
```

完成以后对应的目录

	.
	├── _config.yml
	├── package.json
	├── scaffolds
	├── source
	|   ├── _drafts
	|   └── _posts
	└── themes

安装完成，在themes文件夹中已存在一个模板，由于它不是我们需要的模板，so把里面的文件删除干净，然后在里面用git或者直接down一份[Material](https://github.com/viosey/hexo-theme-material/) 的代码。解压完成。

#### 修改基本配置
先修改主文件夹的 *_config.yml* 文件，具体含义[参考配置](https://hexo.io/zh-cn/docs/configuration.html)， 主要的是修改

	language: zh-CN

把语言设置成中文
``` bash
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: material
```
主题设置成material


打开material文件夹
复制一份 *_config.template.yml* 命名成 *_config.yml* 
里面的具体参数修改参考[这里](https://material.viosey.com/docs/#/config_basic)

照做基本都能成

#### 添加本地搜索

使用本地搜索需要安装 [hexo-generator-search](https://github.com/PaicHyperionDev/hexo-generator-search) 插件。
在修改的 *_config.yml* 中
把 search: use 的值为 google 改为 local 即可。
然后在原 *_config.yml* 中添加
``` bash
search:
    path: search.xml
    field: all
```

#### 添加标签云、照片墙、时间线

[参考这里](https://material.viosey.com/docs/#/pages)

唯一需要说的就是需要在修改的 *_config.yml* 中仿照about界面添加对应标签

	# Sidebar Customize
	sidebar:
		pages:
			tags: #make sure the value corresponds to another one in language file
	            link: "/tags"
	            icon: label
	            divider: false
	        gallery:
	            link: "/gallery"
	            icon: photo
	            divider: false
	        about:
	            link: "/about"
	            icon: person
	            divider: false
	        timeline:
	            link: "/timeline"
	            icon: timeline
	            divider: false
	        links:
	            link: "/links"
	            icon: link
	            divider: false
修改多语言文件 *zh_CN.yaml*
sidebar:
    homepage: "主页"
    archive: "归档"
    article_num: "文章总数"
    about: "关于我" 
    gallery: "图库"
    links: "友情链接"
    tags: "标签云" #this object name is mentioned above
    timeline: "时间轴"


#### 其他微调
按照配置与自己心意随意更改，只要你喜欢，改代码也是可以的


### 遇坑
#### 自定义Pages的多语言
我们添加了添加标签云、照片墙、时间线，同时，在zh_CN.yaml中配置了对应的中文，但是就是不显示
可修改代码 *layout/_partial/sidebar-navigation.ejs* 中的这一段
``` html
<% if(theme.sidebar.pages[i].icon){ %>
	<i class="material-icons sidebar-material-icons"><%= theme.sidebar.pages[i].icon %></i>
<% } %>
<%= i %>
</a>
</li>
<% if(theme.sidebar.pages[i].divider === true) { %>


把 <%= i %> 替换成 <%= __('sidebar.' + i) %> 
```

同理，主界面有一个分页导航也是如此。

#### 同步到github时Error
同步到github后，随后收到github的邮件， 说 You are attempting to use a Jekyll theme, "material", which is not supported by GitHub Pages.

这是由于hexo的深坑必须得自己手动发布才有效。
安装 deployer 可解决

具体方法是 [https://hexo.io/zh-cn/docs/deployment.html](https://hexo.io/zh-cn/docs/deployment.html)

#### 其他问题
1，关注主题的[Issue](https://github.com/viosey/hexo-theme-material/issues)
2，博客[hexo 博客的神坑及本质原因](https://liguanghe.github.io/2017/05/22/blogRebuilt/)
3，度娘
4，谷歌
5，自行分析


