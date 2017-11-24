---
title: Home
layout: default
image: /images/fulls/04.jpg
---

<!-- This loops through the paginated posts -->
{% for post in site.posts %}
  <section>
    <h2><a href="{{ post.url | prepend: site.github.url }}">{{ post.title }}</a></h2>
    <p>
      <span><i class="fa fa-calendar"></i> {{ post.date | date: "%d %B %Y" }}{% if post.categories %} on <i class="fa fa-tags"></i> {{ post.categories | join: ', ' }}{% endif %}</span>
    </p>
    <div>
      {{ post.excerpt }}
    </div>
  </section>
{% endfor %}

{% if paginator.total_pages > 1 %}
<!-- Pagination links -->
<section>
  {% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path }}">Previous</a>
  {% endif %}
  <span>Page: {{ paginator.page }} of {{ paginator.total_pages }}</span>
  {% if paginator.next_page %}
    <a href="{{ paginator.next_page_path }}">Next</a>
  {% endif %}
</section>
{% endif %}