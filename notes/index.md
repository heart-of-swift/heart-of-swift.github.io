---
layout: page
title: 補足ノート
---

{% assign chapter=site.data.book.chapters[2] %}
{% for section in chapter.sections %}
- {% include section-link.md section=section %}
{% endfor %}