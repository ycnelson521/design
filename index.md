---
layout: default
title: ROS2 設計中文化網頁
html_title: ROS 2.0 中文化
---

# ROS 2.0 設計

這個網站是ROS 2.0設計理念的中文化網頁，旨在讓大家了解ROS 2.0的設計理念，並提供開發者努力的方向。
ROS 2.0的設計是要保存ROS 1.x版本的優良部份，並改進ROS1.x所沒有的功能。

# 文章列表

以下是目前已經撰寫完畢的文章，這些文章對於想學習並參與ROS 2.0開發的使用者，是很好的切入點。

{% assign sorted_pages = site.pages | sort:"name" %}
{% for p in sorted_pages %}
    {% if p.url contains 'articles/' and p.published == true %}
----

#### [{{ p.title }}]({{ site.baseurl }}{{ p.url }})

> {{ p.abstract }}
    {% endif %}
{% endfor %}

----

<div class="unpublished" style="display: none;" markdown="1">
# Unpublished Articles

These articles are not finished or maybe not even started yet:

{% assign sorted_pages = site.pages | sort:"name" %}
{% for p in sorted_pages %}
    {% if p.url contains 'articles/' and p.published != true %}
----

#### [{{ p.title }}]({{ site.baseurl }}{{ p.url }})

> {{ p.abstract }}
    {% endif %}
{% endfor %}

----
</div>
