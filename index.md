---
layout: default
title: Home
---

<div class="hero">
  <h1>AI JEDI</h1>
  <p>Workflow guides for designing with AI — catch patterns, automate solutions, and work like a power user.</p>
</div>

<p class="section-heading">Howtos</p>

<ul class="howto-list">
{% for howto in site.howtos %}
  <li>
    <a href="{{ howto.url | relative_url }}">
      <div class="howto-card-image">
        {% if howto.image %}
        <img src="{{ howto.image | relative_url }}" alt="" loading="lazy" />
        {% else %}
        <div class="howto-card-image-placeholder">✦</div>
        {% endif %}
      </div>
      <div class="howto-card-text">
        <div class="howto-link-title">{{ howto.title }}</div>
        {% if howto.description %}
        <div class="howto-link-desc">{{ howto.description }}</div>
        {% endif %}
      </div>
    </a>
  </li>
{% endfor %}
</ul>
