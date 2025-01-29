---
layout: page
title: Blog Archive
---

{% assign sorted_tags = site.tags | sort: "size" | reverse %}
{% for tag in sorted_tags %}
  <h3>{{ tag[0] }}</h3>
  <ul>
    {% for post in tag[1] %}
      <li><a href="{{ post.url }}">{{ post.date | date: "%B %Y" }} - {{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}
