---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: The Open3FS Blog
---

{% for post in site.posts %}
  <article class="post">
    <h2>
      <a href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
      </a>
    </h2>
    <time datetime="{{ post.date | date_to_xmlschema }}">
      {{ post.date | date: "%B %-d, %Y" }}
    </time>
    {% if post.excerpt %}
      {{ post.excerpt }}
    {% endif %}
  </article>
{% endfor %}
