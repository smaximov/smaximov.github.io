---
layout: post
title: "Добавление категорий к заголовку поста"
date: 2013-06-25 22:14
comments: true
categories:
  - Octopress
  - Jekyll
  - meta
---

Решил вот немного изменить внешний вид блога и заодно разобраться в
расположении и назначении стилевых файлов и шаблонов HTML в
Octorpess/Jekyll.

Изменения затрагивают подзаголовок поста, к которому я решил добавить
список категорий. Шаблон списка категорий располагается в файле
`source/_includes/post/categories.html`. Этот шаблон, соответственно,
следует включить в шаблон `source/_includes/article.html` с помощью
тега Liquid `include`:

{% raw %}
``` html
      ...
      <p class="meta">
        {% include post/date.html %}{{ time }}
        {% if site.disqus_short_name and page.comments != false and post.comments != false and site.disqus_show_comment_count == true %}
         | <a href="{% if index %}{{ root_url }}{{ post.url }}{% endif %}#disqus_thread">Comments</a>
        {% endif %}
        {% include post/categories.html %}
      </p>
      ...
```
{% endraw %}

И немного CSS в `sass/custom/_styles.scss`:

``` scss
article > header p.meta {
    span.categories {
       text-transform: none;
       a.category {
          @extend .serif;
          font-size: 1em;
          &:hover, &:link, &:visited, &:active {
             text-decoration: none;
             color: $text-color-light;
          }
       }
    }

   .separator {
      color: darken($text-color-light, 15);
   }
   time + #disqus_thread:before, time + .categories:before, #disqus_thread + .categories:before {
      @extend .separator;
   }
}
```

Вуаля!
