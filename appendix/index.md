---
layout: page
title: 付録
prev_item:
    name: 'おわりに'
    path: '/conclusion'
next_item:
    name: '謝辞'
    path: '/#acknowledgement'
---

{% for item in site.data.book.appendix %}
- {% include section-link.md section=item %}
{% endfor %}

---

{% include pager.html next=page.next_item prev=page.prev_item %}