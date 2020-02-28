---
layout: page
title: About
description: 智慧改变人生
keywords: 潘蕾
comments: true
menu: 关于
permalink: /about/
---

掌中舞罢箫声绝，三十六宫秋夜长。

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
