---
layout: page
title: Tag Index
permalink: /tags/
description: "An archive of posts sorted by tag."
---
 
{% capture tags %}
  {% for tag in site.tags %}
    {{ tag[0] | replace: ' ', '&nbsp;' }}
  {% endfor %}
{% endcapture %}
{% assign sortedtags = tags | split: ' ' | sort %}
 
<ul class="tag-box inline">
{% for tag in sortedtags %}
  {% for post in site.tags[tag] %}
		<li><a href="#{{ tag }}">{{ tag }}</a></li>
	{% endfor %}
{% endfor %}
</ul>
 
{% for tag in sortedtags %}
	<h2 id="{{ tag }}">{{ tag }}</h2>
	<ul class="post-list">
	{% for post in site.tags[tag] %}
		<li><a href="{{ site.url }}{{ post.url }}">{{ post.title }}<span class="entry-date"><time datetime="{{ post.date | date_to_xmlschema }}" itemprop="datePublished">{{ post.date | date: "%B %d, %Y" }}</time></span></a></li>
	{% endfor %}
	</ul>
{% endfor %}
