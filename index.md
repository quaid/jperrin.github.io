---
layout: page
---
{% include JB/setup %}

<div class="posts">
  {% for post in site.posts limit 1 %}
    <article class="post">

      <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

      <div class="entry">
        {{ post.content }}
      </div>

      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Permalink</a>
    </article>
  {% endfor %}
</div>


