---
title: Contact
tags: [contact]
modified: 2015-07-19T20:53:07.573882-04:00
comments: false
permalink: /contact/
---
{% assign author = site.author %}

<div>

  <h3 itemprop="name">{{ author.name }}</h3>
  <p>{{ author.bio }}</p>
  <a href="tel:{{ author.phone | replace:' ','' }}" class="author-social"><i class="fa fa-fw fa-phone"></i> Mobile: {{ author.phone }} (United Kingdom)</a><br />
  <a href="tel:{{ author.phone | replace:' ','' }}" class="author-social"><i class="fa fa-fw fa-phone"></i> Mobile: {{ author.phone-hu }} (Hungary)</a><br />
  {% if author-social %}<span class="author-social"><i class="fa fa-fw fa-map-marker"></i> Address: {{ author.address }}</span>{% endif %}
  {% if author.email %}<a href="mailto:{{ author.email }}" class="author-social" target="_blank"><i class="fa fa-fw fa-envelope-square"></i> Email: {{ author.email }}</a>{% endif %}<br />
</div>
