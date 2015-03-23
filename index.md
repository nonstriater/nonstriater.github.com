---
layout: page
title:  
tagline: 
---

{% include JB/setup %}

##My Posts

{% for category in site.categories %}
<a style="padding: 3px 6px; margin: 2px;background: #eee;color: #555;border-radius: 3px;text-decoration: none;border: 1px dashed #cccccc;">{{ category[0] | join: "/" }}</a>
<ul class="posts">
    {% for post in category[1] %}
    <li style="list-style:none; margin-bottom:3px; line-height:1.7; letter-spacing:1px; font-size:18px">
        <a style="margin-right:3px" href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
        <span style="font-size:12px;color:gray;">{{ post.date | date:"%Y-%m-%d" }}</span>
    </li>
    {% endfor %}
</ul>
{% endfor %}


##Contact me
<ul style="line-height: 1.7; letter-spacing:1px; color:gray;">
    <li style="list-style:none; margin-bottom:3px;">Weibo  : <a href="http://weibo.com/ranwj">@移动开发小冉</a>  </li>
    <li style="list-style:none; margin-bottom:3px;">Twitter : <a href="https://twitter.com/nonstriater">@nonstriater</a>  </li>
    <li style="list-style:none; margin-bottom:3px;">Github  : <a href="https://github.com/nonstriater">@nonstriater</a>  </li>
    <li style="list-style:none; margin-bottom:3px;">Email : 510495266@qq.com</li>
</ul>


