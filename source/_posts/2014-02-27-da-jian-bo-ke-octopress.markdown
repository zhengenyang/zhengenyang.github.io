---

layout: post
title: "搭建博客-Octopress"
date: 2014-02-27 12:21:45 +0800
comments: true
categories:
published: false

---

{% img right /images/blog/octopress.jpg 300 170 Place Kitten #3 %}

>使用Octopress搭建博客的文章已经多如牛毛，写这篇博客的主要目的是感受一下whitespace主题的样式，顺便根据个人喜好做些小的调整

### 安装Octopress

- Octopress的安装很简单，直接参照官方文档即可：[Octopress Setup](http://octopress.org/docs/setup/)

### 配置Github

- 在[Github](https://github.com/join)注册一个账号
- 因为要通过Octopress将生成的博客提交到Github上，所以要在Github上配置自己的SSH Key，参见Github官方文档：[Generating SSH Keys](https://help.github.com/articles/generating-ssh-keys)
- 创建一个名为_username_.github.io的Repository，其中_username_为自己的Github账号名
> 不用勾选Initialize this repository with a README，如果勾选了，则在之后第一次rake deploy之前记得进_deploy目录先pull一下，详见下文
- 既然自己搭建博客，那当然最好要有一个独立的域名了，网上域名服务商很多，例如：[Godaddy](http://www.godaddy.com/)
- 这些都准备好了后，就可以开始在Octopress里配置Github的部署信息了，参见官方文档：[Deploying to Github Pages](http://octopress.org/docs/deploying/github/)  
如果在之前创建Repository时勾选了创建README选项，则需要进入_deploy目录，更新master分支
{% codeblock lang:sh %}
$ cd _deploy
$ git pull origin master
{% endcodeblock %}
> .com域名的a record生效时间一般在十几分钟到一个小时，.cn域名大概需要几个小时，各个域名服务商会有差异，需要耐心等待  
> Github上CNAME的生效时间一般是十几分钟

### 配置Octopress

- Octopress并不需要过多配置，只需_config.yml里修改几个值就可以使用了，具体参见官方文档：[Configuring Octopress](http://octopress.org/docs/configuring/)
	- url：改成自己的域名，比如我的：leonotes.com
	- title：博客主标题，比如我的：寧靜▪致遠
	- subtitle：博客副标题，比如我的：所見所聞 所思所悟
	- author：自己的名字
	- date_format：博客撰写日期的显示形式，默认是ordinal，日期显示形式为：FEB 27TH, 2014，我改成了%F，显示为：2014-02-27
	> 关于ruby日期格式的更多描述可以参见ruby的官方文档：[Ruby Time](http://www.ruby-doc.org/core-1.9.2/Time.html#method-i-strftime)
	- 还有一点很重要的是关闭twitter，否则会引入[ http://platform.twitter.com/widgets.js ]，在天朝，你只能看着浏览器不停地在加载，直至失败
	
{% codeblock lang:sh %}
# Twitter
twitter_user:
twitter_tweet_button: false
{% endcodeblock %}

### 设置样式

- 网上开源的Octopress Theme有很多，找一个自己喜欢的，再根据自己的喜好做一些小的调整应该是最经济的方式了
- Octopress的官方Github列出了一些可用的主题：[3rd Party Octopress Themes](https://github.com/imathis/octopress/wiki/3rd-Party-Octopress-Themes)，而[Opthemes](http://opthemes.com/)则已更直观的方式列出了一些可用的主题
- 安装主题的方式非常简单，只需要把包含主题的文件夹放入source/.themes，然后rake install['theme_name']即可，以安装whitespace主题为例：
{% codeblock lang:sh %}
$ cd octopress
$ git clone git://github.com/lucaslew/whitespace.git .themes/whitespace
$ rake install['whitespace']
{% endcodeblock %}
- 调整主题最好的方式是修改该主题文件夹下的sass/custom目录下对应的文件
	- _colors.scss 颜色定义，背景色、字体颜色等
	- _fonts.scss 字体定义，定义标题、正文、代码块等所用的字体
	- _layout.scss 布局定义，页面宽度、padding、margin等
	- _styles.scss 样式定义，这个文件会被最后加载，所有在此的定义都会覆盖其它文件里级联定义的值
	
### 一些网页样式的原则
	
- 总的来说，背景、字体、字号以及行间距等都会对一个网页的阅读舒适度造成直观影响
- 过亮的背景色很容易造成阅读疲劳，因此，将背景色设置为乳白色或者淡黄色会是一个不错的选择
- 一般来说非衬线字适合做标题，衬线字适合正文内容，然而中文字体中没有令人特别满意的衬线字，因此，我还是选择了非衬线字作为正文的字体
- 随着目前主流显示器的增大，分辨率的提高，一般网页的字体也由之前的10px，12px向14px，16px，甚至18px过渡。另外使用em代替px定义字体大小是更规范的做法，一般来说1em=16px
- 行间距也是影响阅读舒适度的主要原因，过小的行间距会使文字看上去堆叠在一起，影响美观，阅读时也容易串行，而过大的行间距则会影响阅读的连贯性，一般来说1.5em ~ 1.8em的行间距会比较合适，我选择的是1.6em

### 字体设置的原则

- 英文字体在前，中文字体在后；这是因为绝大部分中文字体中都包含英文字体，然而基本上都很丑，所以英文字符还是交给英文字体来渲染吧
- 定义中文字体的时候要使用对应的英文名称，而不要只使用中文字体的显示名称，比如："STXihei","华文细黑"
- mac系统字体在前，windows系统字体在后，这是因为很多mac系统用户都安装了windows字体（比如安装office系列软件），让mac用户优先用mac系统的字体渲染，而不是用丑陋的windows系统字体
- 定义fallback字体，在指定字体找不到的时候可以fallback

> 我的定义：  
> $sans: "Helvetica Neue", "Tahoma", "Arial", "STXihei","华文细黑", "Microsoft YaHei","微软雅黑", "Heiti","黑体", "SimSun","宋体", sans-serif !default;  
> 英文字体先查找"Helvetica Neue"（For mac），再查找 "Tahoma"（For win），如果找不到，查找"Arial"（For mac&win）  
> 中文字体先查找"STXihei","华文细黑"（For mac），再查找 "Microsoft YaHei","微软雅黑"（For vista及以上），再查找"Heiti","黑体"（For mac&win），再查找"SimSun","宋体"（For xp）  
> 若以上字体都缺失，使用默认的sans-serif字体

### 添加表格样式

First Header | Second Header | Third Header
:----------- | :-----------: | -----------:
Left         | Center        | Right
Left         | Center        | Right
  
<br/>
  
{% codeblock lang:css %}
table {
	max-width: 100%;
	background-color: transparent;
	border-collapse: collapse;
	border-spacing: 0;
	th, td {
		padding: 8px;
		border: 1px solid #aaa;
	}
	th {
		font-weight: bold;
		background-color: #e0e0e0;
	}
	th[align="left"], td[align="left"] {
		text-align: left;
	}
	th[align="center"], td[align="center"] {
		text-align: center;
	}
	th[align="right"], td[align="right"] {
		text-align: right;
	}
}
{% endcodeblock %}

{% codeblock %}
First Header | Second Header | Third Header
:----------- | :-----------: | -----------:
Left         | Center        | Right
Left         | Center        | Right
{% endcodeblock %}
