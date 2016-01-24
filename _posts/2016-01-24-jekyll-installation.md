---
layout: post
---

# jekyll安装之旅

jekyll的安装说起来也是一把心酸泪,我本来以为挺简单一个事儿,就直接跟着[官方文档](http://jekyllcn.com/docs/installation/)来呗, 我是用的ubuntu14.04,由于jekyll依赖ruby,所以我就安个ruby呗,`sudo apt-get install ruby`,

安装ruby倒是挺顺利的,安完ruby,那自然就该按文档上说的来安装jekyll咯 `sudo gem install jekyll`,可是谁曾想到,居然报错说,ruby的版本太低了,至少要2.0的, 那就没办法咯,只有把刚装的那个ruby干掉了 `sudo apt-get remove ruby`

怒了,那就安装个最新的版的ruby呗,跑去官网下载最新版本的ruby ([官网](http://www.ruby-lang.org/en/))

    tar -xzvf ruby版本 -C 指定解压目录

cd到 解压后的ruby根目录下,依次执行以下命令

    $ ./configure
    $ make
    $ sudo make install

完成以上三步后ruby就安装完成了,使用 `ruby --version` 看看ruby版本,应该就没什么问题了.

下面又到了jekyll的时候了, `gem install jekyll` 出现了以下错误一枚:

    ERROR:  Loading command: install (LoadError)
	 cannot load such file -- zlib
    ERROR:  While executing gem ... (NoMethodError)
    undefined method 'invoke_with_build_args' for nil:NilClass

这个错误的原因是依赖的zlib那个包没安装,那咋整呢,其实也蛮简单,在ruby根目录下执行一下命令

    $cd ext/zlib
    $sudo ruby ./extconf.rb
    $sudo make
    $sudo make install

安好之后那就再来一波jekyll安装呗,再次执行 `sudo gem install jekyll` 又出现错误,那一刻我的内心是崩溃的:

    ERROR:  While executing gem ...   (Gem::Exception)
    Unable to require openssl, install OpenSSL
    and rebuild ruby (preferred) or use non-HTTPS sources

查了下,出现这个的原因是gem使用的源是https的,https要求的证书拿不到,解决方案就是把它换成http的, 执行下面两个命令:

    gem source -r https://rubygems.org/
    gem source -a http://rubygems.org/

抱着一颗绝望的心最后再执行了一次 `sudo gem install jekyll` ,娃哈哈,终于成功了.

接下来,就可以看看[使用文档](http://jekyllcn.com/docs/quickstart/),尽情的玩转jekyll了.
