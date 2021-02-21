---
layout: info
title: About
description: 曾梦想仗剑天涯，后来工作忙没去
keywords: 
comments: true
menu: 关于
permalink: /about/
---

## 联系方式

{% for website in site.data.social %}
* {{ website.sitename }}：[{{ website.name }}]({{ website.url }})
{% endfor %}

## 技能

{% for category in site.data.skills %}


<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
