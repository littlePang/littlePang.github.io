---
layout: post
title: jekyll的配置
category: 杂项
tags: jekyll
keywords:
description:
---

# jekyll用到过的一些配置

## __config.yml配置文件



配置详解: [中文](http://jekyllcn.com/docs/configuration/) and [英文](http://jekyllrb.com/docs/configuration/)

    # 目录结构
    source:      .
    destination: ./_site
    plugins:     ./_plugins
    layouts:     ./_layouts
    data_source: ./_data
    collections: null

    # 阅读处理
    safe:         false
    include:      [".htaccess"]
    exclude:      []
    keep_files:   [".git", ".svn"]
    encoding:     "utf-8"
    markdown_ext: "markdown,mkdown,mkdn,mkd,md"

    # 内容过滤
    show_drafts: null
    limit_posts: 0
    future:      true
    unpublished: false

    # 插件
    whitelist: []
    gems:      []

    # 转换
    markdown:    kramdown
    highlighter: rouge
    lsi:         false
    excerpt_separator: "\n\n"

    # 服务器选项
    detach:  false
    port:    4000
    host:    127.0.0.1
    baseurl: "" # does not include hostname

    # 输出
    permalink:     date
    paginate_path: /page:num
    timezone:      null

    quiet:    false
    defaults: []

    # Markdown 处理器
    rdiscount:
      extensions: []

    redcarpet:
      extensions: []

    kramdown:
      auto_ids:       true
      footnote_nr:    1
      entity_output:  as_char
      toc_levels:     1..6
      smart_quotes:   lsquo,rsquo,ldquo,rdquo
      enable_coderay: false

      coderay:
        coderay_wrap:              div
        coderay_line_numbers:      inline
        coderay_line_number_start: 1
        coderay_tab_width:         4
        coderay_bold_every:        10
        coderay_css:               style

    # Conversion
    markdown:    kramdown
    highlighter: rouge
    lsi:         false
    excerpt_separator: "\n\n" #默认的文章摘要分割符号, 文章中出现这个分割符之前的内容作为文章摘要
