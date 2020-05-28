---
layout: page
title: About
description: 分享技术成长，管理心得。
keywords: Hongjun Jia, 贾洪军
comments: true
menu: 关于
permalink: /about/
---

我是Piter Jia，分享技术成长，管理心得。

仰慕「优雅编码的艺术」。


## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
</ul>


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
