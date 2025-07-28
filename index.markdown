---
layout: default
---
 <script async src="https://www.googletagmanager.com/gtag/js?id=G-KNZ4N1ZWVG"></script>
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', 'G-KNZ4N1ZWVG');
  </script>
<h1>My Blog Posts</h1>

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <p>{{ post.date | date: "%b %d, %Y" }}</p>
      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>
