---
layout: page
title: Разбор некоторых задач с Leetcode
permalink: /leetcode/
---

<div id="archives">
  <div class="archive-group">
    
    {% for post in site.categories["leetcode"] %}
    <article class="post">
      <a href="{{ site.baseurl }}{{ post.url }}">
        <h1>{{ post.title }}</h1>

        <div>
          <p class="post_date">{{ post.date | date: "%B %e, %Y" }}</p>
        </div>
      </a>
      <div class="entry">
        {{ post.excerpt }}
      </div>

      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
    </article>
  </div>
</div>
