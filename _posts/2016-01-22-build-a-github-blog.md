---
layout : page
---
# github的个人博客搭建

最近想搞个个人博客来写东西,但是又觉得如果用csdn或者博客园啥的来写的话,每次都需要在网页上编辑,太麻烦了,所以呢,在网上瞎搜了一下, git+github+markdown+jekyll 或许是个不错的选择呢.

网上看了蛮多人写的,主要还是根据[这篇博客](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)搞起来的.

页面风格,感觉[这个页面](http://minixalpha.github.io/)是我的菜,所以就大概照着搞了搞.不过什么分类啦,归档什么的还没整,毕竟作为一个患有_未知恐惧症_的人来说,直接copy过来用,不知道别人如何做的,如何实现的,总感觉有哪里不对.

下面就来说说,这个我用来给自己瞎BB的博客是如何起来的吧.


## 注册github帐号

这个比较简单啦,就是到[github](https://github.com/littlePang/littlePang.github.io)
,填一下用户名,邮箱,密码就可以正常注册了.
![图片](/assets/picture/1.png)

## 创建博客所用的仓库

 正常注册完成之后,登录即可看到以下页面了.点击页面中的**new repository**按钮.

![](/assets/picture/2.png)

进入到新仓库创建页面后,在Repository Name那里填写userName.github.io这种格式的仓库名称,下面是对这个仓库的一些描述,以及仓库的一些属性配置,默认就好,填完了之后,就可以点击**create repository**按钮创建一个新的仓库了.

![](/assets/picture/3.png)

## 使用这个仓库生成个人博客首页

新仓库创建好后,就进入到了下面这个页面,在这个页面中点 **setting**,进入设置页面

![](/assets/picture/4.png)

进入到设置页面后,将页面往下滑,就能看到**Git Pages**模块了,点击那里的**Launch automatic page generator**

![](/assets/picture/5.png)

进入页面后,中间是你博客首页的内容.如果不想写,就直接往下拉,可以看到**continue to layouts**, 机智的你肯定知道,点它就对了.

![](/assets/picture/6.png)

接下来的页面,就跳转到为你的博客选择一个你喜欢的布局了(反正后面要用jekyll自己搞,这里就随便选啦...),选完之后,就点*publish page***就好了.

![](/assets/picture/7.png)

现在你的个人博客已经就绪了哟,域名就是你的那个仓库名(username.github.io),
现在试着去访问下你的博客吧

![](/assets/picture/8.png)

## 最后
到这里,你的个人博客的首页,就已经搭起来了,如果你还对git的使用不太熟悉的话,现在就应该打开你的搜索页面,搜索git的使用了呢.

接下来就是如何使用jekyll来方便的写博客了.

关于jekyll的东西,可以到[jekyll中文文档](http://jekyllcn.com/)去瞅瞅.

而我安装jekyll的时候又是一堆坑,说起来就是一把心酸泪啊([我的jekyll安装之路]()).

写到这里,不知不觉,截了一大堆图,说了一大堆话了,看了看右上角的时间,已经半夜00:30了,是时候,洗洗睡了....
