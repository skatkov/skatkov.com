---
layout: page
title: Blog 
subtitle: Some of the thoughts, without any particular topic.
sitemap:
  priority: 0.9
---

<section>
	<ul>
	    {% for post in site.posts %}
	        <li class="post-teaser">
	            <a href="{{ post.url | prepend: site.baseurl }}">
	                {{ post.title }}
	            </a>

	            <small>{{ post.date | date: "%d %B %Y" }}</small>
	        </li>
	    {% endfor %}
	</ul>
</section>
