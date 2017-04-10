---
layout: default
title: Hansen先生的博客
navigation_title: Blog
navigation_weight: 2
---
<script src="tabcontent.js" type="text/javascript"></script>

## Posts:
<link href="css/tabcontent_3.css" rel="stylesheet" type="text/css" />
<div>
<ul class="tabs" data-persist="true">
	<li><a href="#all_blog">All</a></li>
	<li><a href="#tech">Technical</a> </li>
	<li><a href="#other">Other</a> </li>
</ul>
<div class="tabcontents">
	<div id="all_blog">
	<ul>
	{% for post in site.posts limit:5 %}
		<li>{{ post.date | date_to_string }} <a href="{{ post.url }}">{{ post.title }}</a></li>
	{% endfor %}
	</ul>
	</div>
	<div id="tech">
	<ul>
	{% for post in site.categories.tech limit:5 %}
		<li>{{ post.date | date_to_string }} <a href="{{ post.url }}">{{ post.title }}</a></li>
	{% endfor %}
	</ul>
	</div>

	<div id="other">
	<ul>
	{% for post in site.categories.other limit:5 %}
		<li>{{ post.date | date_to_string }} <a href="{{ post.url }}">{{ post.title }}</a></li>
	{% endfor %}
	</ul>
	</div>
</div>
</div>
