---
layout: cmd
title: linux命令
subtitle: 磨刀不误砍柴工
keywords: 命令, cmd
comments: ture
menu: 命令
permalink: /cmd/
---

<ul class="listing">
{% for cmd in site.cmd %}
{% if cmd.cmd and cmd.topmost == true %}
<li class="listing-item"><a href="{{ site.url }}{{ cmd.url }}"><span class="top-most-flag">[置顶]</span>{{ cmd.cmd }}</a></li>
{% elsif cmd.cmd and cmd.topmost != true %}
<li class="listing-item"><a href="{{ site.url }}{{ cmd.url }}">{{ cmd.cmd }}</a></li>
{% endif %}
{% endfor %}
</ul>
