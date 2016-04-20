---
layout: post
title: "CentOS 6.5下编译Emacs24.3报错修复"
description: ""
category: emacs
tags: [emacs]
---
{% include JB/setup %}

在CentOS 6.5下编译Emacs24.3会出现如下报错：

    xsettings.o: In function `something_changed_gsettingsCB':
	/home/liuy/download/emacs-24.3/src/xsettings.c:215: undefined reference to `g_settings_get_value'
	/home/liuy/download/emacs-24.3/src/xsettings.c:230: undefined reference to `g_settings_get_value'
	/home/liuy/download/emacs-24.3/src/xsettings.c:244: undefined reference to `g_settings_get_value'
	xsettings.o: In function `init_gsettings':
	/home/liuy/download/emacs-24.3/src/xsettings.c:816: undefined reference to `g_settings_list_schemas'
	/home/liuy/download/emacs-24.3/src/xsettings.c:822: undefined reference to `g_settings_new'
	/home/liuy/download/emacs-24.3/src/xsettings.c:828: undefined reference to `g_settings_get_value'
	/home/liuy/download/emacs-24.3/src/xsettings.c:839: undefined reference to `g_settings_get_value'
	/home/liuy/download/emacs-24.3/src/xsettings.c:848: undefined reference to `g_settings_get_value'
	collect2: ld 返回 1
	make[1]: *** [temacs] 错误 1
	make[1]: Leaving directory `/home/liuy/download/emacs-24.3/src'
	make: *** [src] 错误 2

引用网上的解答：

> Red Hat has taken the unusual step of ripping out the gsettings related
parts of their glib library. So they have something that advertises
itself as gio 2.26.1, but does not include gsettings.
>
> I think this is an odd thing to do. I encourage anyone with a RH
subscription to report it as a bug against glib2-devel.
>
> In Emacs, you can work around it by using configure --without-gsettings.
>
> But Emacs should add a configure test to check that gsettings is really
present. I tried to do this by simply testing for gio/gsettings.h, but
sadly you can't include that header directly, it throws an error if not
included via gio/gio.h.'

据上所述，我们可以通过 `configure --without-gsettings` 绕过这个报错。
