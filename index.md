---
layout: dashboard
title: SRE Dashboard
---

# [ SYSTEM_LOG: All Intelligence Nodes ]

<div class="note-list">
  {% for post in site.posts %}
    <a href="{{ post.url | relative_url }}" class="note-row">
      <span class="note-date">
        {{ post.date | date: "%d-%b-%Y" | upcase }}
      </span>
      <span class="note-symbol">>></span>
      <span class="note-title">{{ post.title }}</span>
      {% if post.categories.size > 0 %}
        <span class="post-category-tag">{{ post.categories | first }}</span>
      {% endif %}
    </a>
  {% endfor %}
</div>

---

### Dashboard Telemetry
- **Active Modules:** {{ site.posts | size }} 
- **Status:** Streaming all categories