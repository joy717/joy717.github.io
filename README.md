## Welcome to joy717's world!

<ul>
  {% for post in site.posts %}
    <li>
      <a href="/blog{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
