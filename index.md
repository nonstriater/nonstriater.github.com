---
layout: page
title: Nonstriater的博客 
tagline: by @nonstriater
---
{% include JB/setup %}

###My Posts
<ul class="posts">
  {% for post in site.posts %}
    <li>
        <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
        <span>{{ post.date | date_to_string }}</span> &raquo; 
    </li>
  {% endfor %}
</ul>

###Contact me
<ul style="line-height: 1.7; letter-spacing:1px; color:gray;">
    <li style="list-style:none; margin-bottom:3px;">Weibo : <a href="http://weibo.com/2js3">@拓词Joey</a>  </li>
    <li style="list-style:none; margin-bottom:3px;">Twitter : <a href="https://twitter.com/XueshiQiao">@XueshiQiao</a>  </li>
    <li style="list-style:none; margin-bottom:3px;">Email : qiaoxueshi#gmail.com</li>
</ul>


