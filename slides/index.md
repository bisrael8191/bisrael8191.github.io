---
layout: page
title: Slides
tags: [slides]
description: List of presentation slides
share: false
permalink: /slides/
---

<section class="listing">
  <ul>
    {% for page in site.pages %}
      {% if page.layout == 'slide' and page.title %}
        <li>
          <a class="page-link" href="{{ page.url | prepend: site.baseurl }}">
            {{ page.title }}
          </a>
        </li>
      {% endif %}
    {% endfor %}
  </ul>
</section>