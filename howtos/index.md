---
layout: default
title: Howtos
---

<div class="hero">
  <h1>Howtos</h1>
  <p>Guides for working effectively with AI — from architecture to automation.</p>
</div>

<ul class="howto-list">
{% for howto in site.howtos %}
  <li>
    <a href="{{ howto.url | relative_url }}">
      <div class="howto-link-title">{{ howto.title }}</div>
      {% if howto.description %}
      <div class="howto-link-desc">{{ howto.description }}</div>
      {% endif %}
    </a>
  </li>
{% endfor %}
</ul>
