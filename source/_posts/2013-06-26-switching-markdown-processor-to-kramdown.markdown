---
layout: post
title: "Kramdown в качестве процессора Markdown"
date: 2013-06-26 18:57
comments: true
categories:
    - Kramdown
    - Octorpess
    - Jekyll
    - Meta
---

Перевёл обработку Markdown на [Kramdown][kramdown]. Что это (пока)
дало: сноски и генерацию содержания. Стили для них взял (с небольшими
изменениями)
[отсюда](http://hiltmon.com/blog/2013/05/08/octopress-now-has-footnotes/)
и
[отсюда](http://blog.riemann.cc/2013/04/10/table-of-contents-in-octopress/):

``` scss sass/custom/_styles.scss
#markdown-toc:before {
   content: "Содержание";
   font-weight: bold;
}
ul#markdown-toc {
   list-style: none;
   float: right;
   @include shadow-box;
   background-color: white;
}

.footnotes {
   font-size: 13px;
   line-height: 16px;
   color: #666;
   border-top: 2px groove gray;
   padding-top: 1em;

   ol {
      padding-left: 2em;

      p {
         margin-bottom: 6px;
      }
   }
}
```

Для вставки содержания в тело поста нужно добавить атрибут `toc` к
элементу списка, например:

``` text
* TOC
{:toc}
```

Простановка сносок похоже на простановку ссылок-сносок:

``` text
This is a text[^fn] with footnotes.

...

[^fn]: footnote text.
```

В квадратных скобках с "птичкой" (caret) указывается идентификатор
сноски, который должен быть уникальным в пределах одного документа.
Идентификатор может состоять из цифр и букв. Текст ссылки
задается где-нибудь в документе через двоеточие.

* * *

Kramdown содержит кучю других расширений Markdown, их описание есть на
[сайте](http://kramdown.rubyforge.org/syntax.html).

Результат мне понравился (примечательно, что этот пост не содержит ни
содержания, ни сносок).

[kramdown]: http://kramdown.rubyforge.org/index.html
